<template>
  <div id="app">
    <canvas id="cvs" :width="w" :height="h" @touchstart.prevent @touchmove="handleTouchMove" @mousemove="handleMouseMove"></canvas>
    <h2>A demo for movable camera and path tracing</h2>
    <p>How to play on PC or Mac</p>
    <ul>
      <li>move mouse inside to look around</li>
      <li>use WSAD key to walk around</li>
    </ul>

    <p>How to play on mobile devices</p>
    <ul>
      <li>touch and move inside to look around</li>
      <li>use two-finger gesture inside to walk around</li>
    </ul>
  </div>
</template>

<script>
// for a raytracer rather than a rasterizer,
// the only entitiy to draw is a rectangular "canvas" that covers the whole viewport
const planeVertices = [
  -1,-1,
  1,-1,
  -1,1,
  1,1,
  -1,1,
  1,-1
];

function createShaderProgram(gl, vsSource, fsSource) {
  const vertexShaderSource = vsSource;
  const fragmentShaderSource = fsSource;

  const vs = gl.createShader(gl.VERTEX_SHADER);
  gl.shaderSource(vs, vertexShaderSource);
  gl.compileShader(vs);

  if (!gl.getShaderParameter(vs, gl.COMPILE_STATUS)) {
    // eslint-disable-next-line
    console.error('An error occurred compiling the vertex shaders: ' + gl.getShaderInfoLog(vs));
    gl.deleteShader(vs);
    return null;
  }

  const fs = gl.createShader(gl.FRAGMENT_SHADER);
  gl.shaderSource(fs, fragmentShaderSource);
  gl.compileShader(fs);

  if (!gl.getShaderParameter(fs, gl.COMPILE_STATUS)) {
    // eslint-disable-next-line
    console.error('An error occurred compiling the fragment shaders: ' + gl.getShaderInfoLog(fs));
    gl.deleteShader(fs);
    return null;
  }

  const shaderProgram = gl.createProgram();
  gl.attachShader(shaderProgram, vs);
  gl.attachShader(shaderProgram, fs);
  gl.linkProgram(shaderProgram);
  return shaderProgram;
}

export default {
  data() {
    return {
      w: 512,
      h: 512,
      mouse: {
        x: 0,
        y: 0
      },
      canvas: {
        left: 0,
        top: 0
      },
      cameraPos: {
        x: 0,
        y: 0,
        z: 6
      },
      cameraDir: {
        yaw: 0,
        pitch: 0,
        x: 0,
        y: 0,
        z: -1
      },
      sampleCount: 0
    }
  },
  mounted() {
    const rect = document.querySelector('canvas').getBoundingClientRect();
    this.canvas.left = rect.left;
    this.canvas.top = rect.top;

    document.addEventListener('keypress', (e) => {
      this.handleKeyPress(e);
    });

    const gl = document.querySelector('#cvs').getContext('webgl');

    const planeVao = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, planeVao);
    const planeData = new Float32Array(planeVertices);
    gl.bufferData(gl.ARRAY_BUFFER, planeData, gl.STATIC_DRAW);

    // In order to implement progressive rendering
    // 2 textures as image storing, and one framebuffer for drawing rendered result to texture are needed
    const textures = [];
    for (var i = 0; i < 2; i++) {
      textures.push(gl.createTexture());
      gl.bindTexture(gl.TEXTURE_2D, textures[i]);
      gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST);
      gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST);
      gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGB, this.w, this.h, 0, gl.RGB, gl.UNSIGNED_BYTE, null);
    }

    const fb = gl.createFramebuffer();

    // the vertex and fragment shader for rendering a stored render result to viewport
    const renderProgram = createShaderProgram(gl,
      `
        attribute vec4 aPosition;
        varying vec2 texCoord;

        void main() {
          texCoord = aPosition.xy * 0.5 + 0.5;
          gl_Position = aPosition;
        }
      `,
      `
        precision highp float;
        varying vec2 texCoord;
        uniform sampler2D tex;

        void main() {
          gl_FragColor = texture2D(tex, texCoord);
        }
      `
    );

    // the vertex and fragment shader for the actual path tracing
    const tracerProgram = createShaderProgram(gl,
      `
        attribute vec4 aPosition;
        varying vec4 vPosition;

        void main() {
          vPosition = aPosition;
          gl_Position = aPosition;
        }
      `,
      `
        precision highp float;
        uniform vec2 u_resolution;
        uniform vec2 u_mouse;
        uniform vec3 u_camera_pos;
        uniform vec3 u_camera_dir;
        varying vec4 vPosition;
        uniform float u_time;
        uniform float u_sw;
        uniform sampler2D tex;

        const vec3 sc0 = vec3(-0.8,0.,-4.);
        const float sr0 = 0.4;
        const vec3 sc1 = vec3(0.,0.,-4.);
        const float sr1 = 0.4;
        const vec3 sc2 = vec3(0.8,0.,-4.);
        const float sr2 = 0.4;
        // sky (a big sphere at the back of the scene)
        const vec3 sc3 = vec3(0.,0.,-104.4);
        const float sr3 = 100.0;
        // ground (a big sphere at the bottom of the scene)
        const vec3 sc4 = vec3(0.,-100.4,-4.);
        const float sr4 = 100.0;
        // light (a sphere with an intentially overflowed color (brighter than vec3(1.,1.,1.)))
        const vec3 sc5 = vec3(0.8,0.5,-3.5);
        const float sr5 = 0.15;

        // code from https://github.com/evanw/webgl-path-tracing/blob/master/webgl-path-tracing.js
        float random(vec3 scale, float seed) {
          return fract(sin(dot(gl_FragCoord.xyz + seed, scale)) * 43758.5453 + seed);
        }

        vec3 cosineWeightedDirection(float seed, vec3 normal) {
          float u = random(vec3(12.9898, 78.233, 151.7182), seed);
          float v = random(vec3(63.7264, 10.873, 623.6736), seed);
          float r = sqrt(u);
          float angle = 6.283185307179586 * v;
          vec3 sdir, tdir;
          if (abs(normal.x)<.5) {
            sdir = cross(normal, vec3(1,0,0));
          } else {
            sdir = cross(normal, vec3(0,1,0));
          }
          tdir = cross(normal, sdir);
          return r*cos(angle)*sdir + r*sin(angle)*tdir + sqrt(1.-u)*normal;
        }

        // the function for ray-sphere intersection, given the origin and direction of a ray and the center and radius of a sphere
        // returns the minimal distance when the ray hits the surface of the sphere
        // when no intersection, returns 10000.0 
        float sphere_hit(vec3 ro, vec3 rd, vec3 center, float r) {
          vec3 oc = ro - center;
          float a = dot(rd, rd);
          float b = 2.0 * dot(oc, rd);
          float c = dot(oc, oc) - r * r;
          float discriminant = b * b - 4.0 * a * c;
          if (discriminant > 0.0) {
            float temp = (- b - sqrt(discriminant)) / (2.0 * a);
            if (temp > 0.) {
              return temp;
            }
          }
          return 10000.0;
        }

        // the function for resolving color on a specific hit point
        // currently returns only the diffuse color
        vec3 calc_color(vec3 hit, vec3 normal, vec3 diff_c) {
          vec3 light = vec3(0.8,0.5,-3.5);
          vec3 l = normalize(light - hit);
          float diff = dot(l, normal);

          vec3 v = normalize(u_camera_pos - hit);
          float spec = dot(v, reflect(-l, normal));
          return pow(max(diff, 0.), 4.) + pow(max(spec,0.), 32.) + diff_c;
        }

        vec4 trace(vec3 ro, vec3 rd) {
          vec3 color_mask = vec3(1.);
          vec3 accu_color = vec3(1.);

          // the path tracing depth
          for (int i = 0; i < 5; i++) {
            // finding the nearest hit point by checking intersection with every spheres
            float dist = 10000.0;
            float d0 = sphere_hit(ro, rd,sc0, sr0);
            float d1 = sphere_hit(ro, rd,sc1, sr1);
            float d2 = sphere_hit(ro, rd,sc2, sr2);
            float d3 = sphere_hit(ro, rd,sc3, sr3);
            float d4 = sphere_hit(ro, rd,sc4, sr4);

            if (dist > d0) dist = d0;
            if (dist > d1) dist = d1;
            if (dist > d2) dist = d2;
            if (dist > d3) dist = d3;
            if (dist > d4) dist = d4;

            vec3 hit = ro + dist * rd;
            vec3 normal;
            vec3 diffuse;

            if (dist == 10000.0) {
              break;
            // if any sphere is hit, accumulate the color on the sphere,
            // and find a randomly scattered ray from the hit point as new ray
            // for the next loop of path tracing
            } else if (dist == d0) {
              diffuse = vec3(1.,0.3,0.3);
              normal = normalize(hit - sc0);
            } else if (dist == d1) {
              diffuse = vec3(1.,1.,1.);
              normal = normalize(hit - sc1);
            } else if (dist == d2) {
              diffuse = vec3(0.3,0.3,1.);
              normal = normalize(hit - sc2);
            } else if (dist == d3) {
              diffuse = vec3(0.2,0.4,0.4);
              normal = normalize(hit - sc3);
            } else if (dist == d4) {
              diffuse = vec3(0.3,0.2,0.1);
              normal = normalize(hit - sc4);
            }
            color_mask *= diffuse;
            rd = cosineWeightedDirection(u_time+float(i), normal);

            accu_color *= calc_color(hit, normal, color_mask);
            ro = hit;
          }
          return vec4(accu_color, 1.);
        }

        const float screen_dist = 6.;

        void main() {
          // s,t is the x,y axis of the screen-space coordinate,
          // first assuming that there is a screen that lies in the world-space with its center at (0,0,0)
          // and the plane of screen is parallel to the xy plane
          vec3 st = vec3(vPosition.xy, 0.);
          st.x = st.x * u_resolution.x / u_resolution.y;
          st.y = -st.y;

          // then, convert the screen to world-space where a camera position and direction is determined
          // this is done by an inverse matrix operation of gluLookAt
          // first change the basis, then translate to the world-space

          vec3 screen_origin = u_camera_pos + screen_dist * u_camera_dir;
          vec3 v = normalize(u_camera_dir);
          vec3 up = vec3(0.,1.,0.);
          vec3 r = normalize(cross(v,up));
          vec3 u = normalize(cross(v,r));
          
          mat4 basis = mat4(
            r.x,r.y,r.z,0.,
            u.x,u.y,u.z,0.,
            v.x,v.y,v.z,0.,
            0.,0.,0.,1.
          );

          mat4 translate = mat4(
            1.,0.,0.,0.,
            0.,1.,0.,0.,
            0.,0.,1.,0.,
            screen_origin.x,screen_origin.y,screen_origin.z,1.
          );

          mat4 view = translate * basis;
          vec4 str = vec4(st,1.);
          str = view * str;

          vec3 rd = normalize(str.xyz - u_camera_pos);
          vec3 ro = u_camera_pos;
          vec4 result = trace(ro, rd);
          
          // gamma correction
          result.x = sqrt(result.x);
          result.y = sqrt(result.y);
          result.z = sqrt(result.z);
          // progressive rendering, where the results is accumulated with previous frame render result
          // u_sw is a value that approximates 1
          // thus the new frame will contribute less to the final image as time elapses
          vec3 prev = texture2D(tex, gl_FragCoord.xy/512.0).rgb;
          gl_FragColor = vec4(mix(result.rgb, prev, u_sw), 1.);
        }
      `
    );

    // draw call
    let time = 0;
    let then = 0;
    let w = this.w, h = this.h;

    let self = this;
    function draw(now) {
      if (!now) now = 0;
      now *= 0.05;
      const deltaTime = now - then;
      then = now;

      gl.clearColor(0,0,0,0);

      gl.bindBuffer(gl.ARRAY_BUFFER, planeVao);

      // tracer, render to texture
      gl.linkProgram(tracerProgram);
      gl.useProgram(tracerProgram);

      // position attrib
      gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 8, 0);
      gl.enableVertexAttribArray(0);
      gl.bindAttribLocation(tracerProgram, 0, 'aPosition');

      time+=deltaTime;
      self.sampleCount++;
      // time uniform
      let tul = gl.getUniformLocation(tracerProgram, 'u_time');
      gl.uniform1f(tul, time);

      // sample weight
      let swul = gl.getUniformLocation(tracerProgram, 'u_sw');
      gl.uniform1f(swul, self.sampleCount / (self.sampleCount + 1));

      // mouse uniform
      let mul = gl.getUniformLocation(tracerProgram, 'u_mouse');
      gl.uniform2f(mul, self.mouse.x, self.mouse.y);

      // resolution uniform
      let rul = gl.getUniformLocation(tracerProgram, 'u_resolution');
      gl.uniform2f(rul, w, h);

      // camera pos uniform
      let cpul = gl.getUniformLocation(tracerProgram, 'u_camera_pos');
      gl.uniform3f(cpul, self.cameraPos.x, self.cameraPos.y, self.cameraPos.z);

      // camera dir uniform
      let cdul = gl.getUniformLocation(tracerProgram, 'u_camera_dir');
      gl.uniform3f(cdul, self.cameraDir.x, self.cameraDir.y, self.cameraDir.z);

      // for progressive rendering, two textures are alteratively used to store the current path trace result
      gl.bindTexture(gl.TEXTURE_2D, textures[0]);
      gl.bindFramebuffer(gl.FRAMEBUFFER, fb);
      gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, textures[1], 0);
      gl.drawArrays(gl.TRIANGLES, 0, 6);
      gl.bindFramebuffer(gl.FRAMEBUFFER, null);

      textures.reverse();

      // render to canvas
      gl.useProgram(renderProgram);

      gl.bindTexture(gl.TEXTURE_2D, textures[0]);

      // position attrib
      gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 8, 0);
      gl.enableVertexAttribArray(0);
      gl.bindAttribLocation(tracerProgram, 0, 'aPosition');
      gl.drawArrays(gl.TRIANGLES, 0, 6);

      window.requestAnimationFrame((timestamp) => draw(timestamp));
    }
    window.requestAnimationFrame((timestamp) => draw(timestamp));
  },
  methods: {
    // methods for implementing the movable camera
    crossProduct(x1,y1,z1,x2,y2,z2) {
      return [
        y1*z2-z1*y2,
        z1*x2-x1*x2,
        x1*y2-y1*x2
      ];
    },
    degToRad(deg) {
      return deg * 3.14159 / 180;
    },
    handleLeftRight(speed) {
      const r = this.crossProduct(this.cameraDir.x, this.cameraDir.y, this.cameraDir.z, 0,1,0);
      this.cameraPos.x -= speed * r[0];
      this.cameraPos.y -= speed * r[1];
      this.cameraPos.z -= speed * r[2];
    },
    handleForwardBackward(speed) {
      this.cameraPos.x += speed * this.cameraDir.x;
      this.cameraPos.y += speed * this.cameraDir.y;
      this.cameraPos.z += speed * this.cameraDir.z;
    },
    handleKeyPress(e) {
      this.sampleCount = 0;
      const key = e.key;
      const speed = 0.1;
      if (key === 'w') {
        this.cameraPos.x += speed * this.cameraDir.x;
        this.cameraPos.y += speed * this.cameraDir.y;
        this.cameraPos.z += speed * this.cameraDir.z;
      } else if (key === 's') {
        this.cameraPos.x -= speed * this.cameraDir.x;
        this.cameraPos.y -= speed * this.cameraDir.y;
        this.cameraPos.z -= speed * this.cameraDir.z;
      } else if (key === 'a') {
        const r = this.crossProduct(this.cameraDir.x, this.cameraDir.y, this.cameraDir.z, 0,1,0);
        this.cameraPos.x -= speed * r[0];
        this.cameraPos.y -= speed * r[1];
        this.cameraPos.z -= speed * r[2];
      } else if (key === 'd') {
        const r = this.crossProduct(this.cameraDir.x, this.cameraDir.y, this.cameraDir.z, 0,1,0);
        this.cameraPos.x += speed * r[0];
        this.cameraPos.y += speed * r[1];
        this.cameraPos.z += speed * r[2];
      }
    },
    handleMouseMove(e) {
      e.preventDefault();

      this.sampleCount = 0;
      let x = (e.clientX - this.canvas.left ) / this.w - 0.5;
      let y = (e.clientY - this.canvas.top ) / this.h - 0.5;

      y = -y;


      x *= 360;
      x = this.degToRad(x);

      y *= 180;
      y = this.degToRad(y);

      this.cameraDir.pitch = y;
      this.cameraDir.yaw = x;

      this.cameraDir.z = -Math.cos(this.cameraDir.pitch) * Math.cos(this.cameraDir.yaw);
      this.cameraDir.y = Math.sin(this.cameraDir.pitch);
      this.cameraDir.x = Math.cos(this.cameraDir.pitch) * Math.sin(this.cameraDir.yaw);
    },
    handleTouchMove(e) {
      e.preventDefault();

      this.sampleCount = 0;
      let x = (e.touches[0].clientX - this.canvas.left ) / this.w - 0.5;
      let y = (e.touches[0].clientY - this.canvas.top ) / this.h - 0.5;

      y = -y;

      if (e.touches.length === 2) {
        this.handleLeftRight(-0.05*x);
        this.handleForwardBackward(0.05*y);
      } else {
        x *= 360;
        x = this.degToRad(x);

        y *= 180;
        y = this.degToRad(y);

        this.cameraDir.pitch += 0.002*y;
        this.cameraDir.yaw += 0.002*x;

        this.cameraDir.z = -Math.cos(this.cameraDir.pitch) * Math.cos(this.cameraDir.yaw);
        this.cameraDir.y = Math.sin(this.cameraDir.pitch);
        this.cameraDir.x = Math.cos(this.cameraDir.pitch) * Math.sin(this.cameraDir.yaw);
      }
    }
  }
};
</script>

<style>
body {
  margin: 0;
  overflow: hidden;
  user-select: none;
}
#app {
  width: 100vw;
  overflow: hidden;
}
</style>
