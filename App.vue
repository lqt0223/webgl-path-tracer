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
  1,1
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
        uniform vec2 u_mouse;
        uniform vec3 u_camera_pos;
        uniform vec3 u_camera_dir;
        varying vec4 vPosition;
        uniform float u_time;
        uniform float u_sw;
        uniform sampler2D tex;

        // cube min vertx
        const vec3 cmin0 = vec3(-0.2,-1.0,-1.);
        // cube max vertx
        const vec3 cmax0 = vec3(0.2,-0.6,-0.6);
        // cube diffuse color
        const vec3 cd0 = vec3(0.52);

        // sphere center
        const vec3 sc0 = vec3(-0.8,-0.5,-4.);
        // sphere radius
        const float sr0 = 0.5;
        // sphere diffuse color
        const vec3 sd0 = vec3(1.,0.3,0.3);

        const vec3 sc1 = vec3(0.8,-0.5,-4.);
        const float sr1 = 0.5;
        const vec3 sd1 = vec3(0.84,0.55,0.15);


        const vec3 sc2 = vec3(0.0,-0.7,-3.);
        const float sr2 = 0.3;
        const vec3 sd2 = vec3(1.,1.,1.);


        // sky (a big sphere at the back of the scene)
        const vec3 sc3 = vec3(0.,-0.5,-115.4);
        const float sr3 = 100.0;
        const vec3 sd3 = vec3(0.2,0.4,0.4);


        // ground (a big sphere at the bottom of the scene)
        const vec3 sc4 = vec3(0.,-101.,-4.);
        const float sr4 = 100.0;
        const vec3 sd4 = vec3(0.3,0.2,0.1);


        float random(vec2 scale, float seed) {
          return fract(sin(dot(gl_FragCoord.xy + seed, scale)) * 43758.5453);
        }

        vec3 cosineWeightedDirection(float seed, vec3 normal) {
          float u = random(vec2(12.9898, 78.233), seed);
          float v = random(vec2(63.7264, 10.873), seed);

          float theta = 6.28 * u;
          float r = sqrt(1. - v);
          float x = r * cos(theta);
          float y = sqrt(v);
          float z = r * sin(theta);

          vec3 right = normalize(cross(normal, vec3(0.1)));
          vec3 front = cross(right, normal);
          return right * x + normal * y + front * z;
        }

        vec3 customWeightedDirection(float seed, vec3 normal) {
          float u = random(vec2(12.9898, 78.233), seed);
          float v = random(vec2(63.7264, 10.873), seed);

          float theta = 6.28 * u;
          float k = pow(v, 0.0001);
          float r = sqrt(1. - k*k);
          float x = r * cos(theta);
          float y = k;
          float z = r * sin(theta);

          vec3 right = normalize(cross(normal, vec3(0.1)));
          vec3 front = cross(right, normal);
          return right * x + normal * y + front * z;
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

        // the function for axis-aligned ray-box intersection
        vec2 box_hit(vec3 ro, vec3 rd, vec3 box_min, vec3 box_max) {
          vec3 tmin = (box_min - ro) / rd;
          vec3 tmax = (box_max - ro) / rd;

          vec3 t1 = min(tmin, tmax);
          vec3 t2 = max(tmin, tmax);

          float tnear = max(max(t1.x, t1.y), t1.z);
          float tfar = min(min(t2.x, t2.y), t2.z);

          return vec2(tnear, tfar);
        }

        // the function for resolving color on a specific hit point
        // currently returns only the diffuse color
        vec3 calc_color(vec3 hit, vec3 normal, vec3 diff_c) {
          vec3 light = vec3(2.8,2.5,1.5);
          vec3 l = normalize(light - hit);
          float diff = dot(l, normal);

          vec3 v = normalize(u_camera_pos - hit);
          float spec = dot(v, reflect(-l, normal));
          return pow(max(diff, 0.), 4.) + pow(max(spec,0.), 32.) + diff_c;
        }

        vec3 refr(vec3 ray, vec3 normal, float ri) {
          float dt = dot(ray, normal);
          float ni_over_nt;
          vec3 n;
          if (dt > 0.) {
            n = -normal;
            ni_over_nt = ri;
          } else {
            n = normal;
            ni_over_nt = 1./ri;
          }

          float discriminant = 1. - ni_over_nt*ni_over_nt*(1. - dt*dt);
          if (discriminant > 0.) {
            return ri*(ray - n*dt)-n*sqrt(discriminant);
          } else {
            return reflect(ray, normal);
          }
        }

        vec3 box_normal(vec3 hit, vec3 box_min, vec3 box_max) {
          if (hit.x < box_min.x + 0.001) return vec3(-1,0,0);
          else if (hit.x > box_max.x - 0.001) return vec3(1,0,0);

          else if (hit.y < box_min.y + 0.001) return vec3(0,-1,0);
          else if (hit.y > box_max.y - 0.001) return vec3(0,1,0);

          else if (hit.z < box_min.z + 0.001) return vec3(0,0,-1);
          return vec3(0,0,1);
        }

        vec4 trace(vec3 ro, vec3 rd) {
          vec3 color_mask = vec3(1.);
          vec3 accu_color = vec3(1.);

          // the path tracing depth
          for (int i = 0; i < 5; i++) {
            // finding the nearest hit point by checking intersection with every spheres
            float dist = 10000.0;
            vec2 dc0 = box_hit(ro, rd, cmin0, cmax0);
            float d0 = sphere_hit(ro, rd,sc0, sr0);
            float d1 = sphere_hit(ro, rd,sc1, sr1);
            float d2 = sphere_hit(ro, rd,sc2, sr2);
            float d3 = sphere_hit(ro, rd,sc3, sr3);
            float d4 = sphere_hit(ro, rd,sc4, sr4);

            vec3 normal;
            vec3 diffuse;

            if (dc0.x > 0. && dc0.x < dc0.y && dist > dc0.x) dist = dc0.x;
            if (dist > d0) dist = d0;
            if (dist > d1) dist = d1;
            if (dist > d2) dist = d2;
            if (dist > d3) dist = d3;
            if (dist > d4) dist = d4;

            vec3 hit = ro + dist * rd;

            if (dist == 10000.0) {
              break;
            // if any sphere is hit, accumulate the color on the sphere,
            // and find a randomly scattered ray from the hit point as new ray
            // for the next loop of path tracing
            } else if (dist == dc0.x) {
              diffuse = cd0;
              normal = box_normal(hit, cmin0, cmax0);
            } else if (dist == d0) {
              diffuse = sd0;
              normal = (hit - sc0) / sr0;
            } else if (dist == d1) {
              diffuse = sd1;
              normal = (hit - sc1) / sr1;
            } else if (dist == d2) {
              diffuse = sd2;
              normal = (hit - sc2) / sr2;
            } else if (dist == d3) {
              diffuse = sd3;
              normal = (hit - sc3) / sr3;
            } else if (dist == d4) {
              diffuse = sd4;
              normal = (hit - sc4) / sr4;
            }

            if (dist == dc0.x) {
              rd = customWeightedDirection(u_time, refr(rd, normal, 1.2));
            } else if (dist == d0) {
              rd = customWeightedDirection(u_time, reflect(rd, normal));
            } else {
              rd = cosineWeightedDirection(u_time, normal);
            }

            color_mask *= diffuse;
            accu_color *= calc_color(hit, normal, color_mask);
            ro = hit - 0.0001 * normal;
          }
          return vec4(accu_color, 1.);
        }

        const float screen_dist = 6.;

        void main() {
          // s,t is the x,y axis of the screen-space coordinate,
          // first assuming that there is a screen that lies in the world-space with its center at (0,0,0)
          // and the plane of screen is parallel to the xy plane
          vec3 st = vec3(vPosition.xy, 0.);
          st.y = -st.y;

          // then, convert the screen to world-space where a camera position and direction is determined
          // this is done by an inverse matrix operation of gluLookAt
          // first change the basis, then translate to the world-space

          vec3 screen_origin = u_camera_pos + screen_dist * u_camera_dir;
          vec3 v = u_camera_dir;
          vec3 up = vec3(0.,1.,0.);
          vec3 r = normalize(cross(v,vec3(0.,1.,0.)));
          vec3 u = cross(v,r);
          
          vec3 str = st.x * r + st.y * u + st.z * v + screen_origin;

          vec3 rd = normalize(str - u_camera_pos);
          vec3 ro = u_camera_pos;
          vec4 result = trace(ro, rd);
          
          // gamma correction
          result = sqrt(result);
          // progressive rendering, where the results is accumulated with previous frame render result
          // u_sw is a value that approximates 1
          // thus the new frame will contribute less to the final image as time elapses
          vec3 prev = texture2D(tex, gl_FragCoord.xy/512.0).rgb;
          gl_FragColor = vec4(mix(result.rgb, prev, u_sw), 1.);
        }
      `
    );

    const planeVao = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, planeVao);
    const planeData = new Float32Array(planeVertices);
    gl.bufferData(gl.ARRAY_BUFFER, planeData, gl.STATIC_DRAW);
    gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 8, 0);
    gl.enableVertexAttribArray(0);
    gl.bindAttribLocation(tracerProgram, 0, 'aPosition');

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

    // draw call
    let time = 0;
    let then = 0;

    let self = this;
    function draw(now) {
      if (!now) now = 0;
      now *= 0.05;
      const deltaTime = now - then;
      then = now;

      gl.clearColor(0,0,0,0);

      gl.bindBuffer(gl.ARRAY_BUFFER, planeVao);

      // tracer, render to texture
      gl.useProgram(tracerProgram);

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
      gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
      gl.bindFramebuffer(gl.FRAMEBUFFER, null);

      textures.reverse();

      // render to canvas

      // position attrib
      gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);

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
