<template>
  <div id="app">
    <canvas id="cvs" :width="w" :height="h" @touchstart.prevent @touchmove="handleTouchMove" @mousemove="handleMouseMove"></canvas>
    <h2>A demo for movable camera and path tracing</h2>
    <span>Select a camera mode: </span>
    <input type="radio" id="free" value="free" v-model="cameraMode">
    <label for="free">free</label>
    <input type="radio" id="focus" value="focus" v-model="cameraMode">
    <label for="focus">focus</label>
    <p>TIP: On PC or Mac, use F to toggle camera mode</p>
    <p>
      <button @click="showHowToPlay">How to play?</button>
    </p>
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
      cameraMode: 'focus',
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
      screenDist: 6,
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
      screenOrigin: {
        x: 0,
        y: 0,
        z: 0
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
        uniform vec3 u_camera_pos;
        varying vec4 vPosition;
        uniform float u_time;
        uniform float u_sw;
        uniform mat4 u_screen_to_world;
        uniform sampler2D tex;

        // cube min vertx
        const vec3 cmin0 = vec3(-0.2,-1.0,-4.2);
        // cube max vertx
        const vec3 cmax0 = vec3(0.2,-0.6,-3.8);
        // cube diffuse color
        const vec3 cd0 = vec3(0.9);

        // sphere center
        const vec3 sc0 = vec3(-0.8,-0.8,-4.);
        // sphere radius
        const float sr0 = 0.2;
        // sphere diffuse color
        const vec3 sd0 = vec3(0.9);

        // cylinder center
        const vec3 cc0 = vec3(0.8, -0.7, -4.0);
        // cylinder height
        const float ch0 = 0.3;
        // cylinder radius
        const float cr0 = 0.3;

        // y_triangle y value
        const float ty0 = -0.95;

        // y_triangle vertices
        const vec2 tv00 = vec2(-0.5,-2.866);
        const vec2 tv01 = vec2(0.5,-2.866);
        const vec2 tv02 = vec2(0.,-2.);

        // sky (a infinite plane at the back of the scene)
        const float pz = -6.5;
        const vec3 pzd = vec3(1.,0.5,0.5);

        // ground (a infinite plane at the bottom of the scene)
        const float py = -1.;
        const vec3 pyd = vec3(0.78,0.95,0.84);


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

        // the function for a z-axis perpendicular plane and ray intersection
        float z_plane_hit(vec3 ro, vec3 rd, float z) {
          return (z - ro.z) / rd.z;
        }

        // the function for a y-axis perpendicular plane and ray intersection
        float y_plane_hit(vec3 ro, vec3 rd, float y) {
          return (y - ro.y) / rd.y;
        }

        // the function for a y-axis perpendicular disk (circle shaped plane) and ray intersection
        float y_disk_hit(vec3 ro, vec3 rd, vec3 center, float r) {
          float t = y_plane_hit(ro, rd, center.y);
          if (t > 0.) {
            vec3 hit = ro + t * rd;
            if (length(center - hit) < r) {
              return t;
            }
          }
          return 10000.0;
        }

        // the function for ray-cylinder intersection
        float cylinder_hit(vec3 ro, vec3 rd, vec3 center, float h, float r, out int face) {
          vec2 oc = (ro - center).xz;
          vec2 rd2 = rd.xz;
          float a = dot(rd2, rd2);
          float b = 2.0 * dot(oc, rd2);
          float c = dot(oc, oc) - r * r;
          float discriminant = b * b - 4.0 * a * c;
          float top = center.y + h;
          float bottom = center.y - h;

          float temp = (- b - sqrt(discriminant)) / (2.0 * a);
          vec3 hit = ro + temp * rd;
          if (temp > 0. && hit.y < top && hit.y > bottom) {
            face = 0;
            return temp;
          } else if (hit.y > top) {
            face = 1;
            return y_disk_hit(ro, rd, center + vec3(0.,h,0.), r);
          } else {
            face = 2;
            return y_disk_hit(ro, rd, center + vec3(0.,-h, 0.), r);
          }

          return 10000.0;
        }

        float det2x2(vec2 va, vec2 vb) {
          return (va.x * vb.y) - (vb.x * va.y);
        }

        bool in_triangle(vec2 p, vec2 v1, vec2 v2, vec2 v3) {
          float d1 = det2x2(v2 - v1,p - v1);
          float d2 = det2x2(v3 - v2,p - v2);
          float d3 = det2x2(v1 - v3,p - v3);

          return (
            (d1 < 0. && d2 < 0. && d3 < 0.) ||
            (d1 > 0. && d2 > 0. && d3 > 0.)
          );
        }

        // the function for a y-axis perpendicula triangle and ray intersection
        float y_triangle_hit(vec3 ro, vec3 rd, float y, vec2 v1, vec2 v2, vec2 v3) {
          float t = y_plane_hit(ro, rd, y);
          vec3 hit = ro + t * rd;
          if (in_triangle(hit.xz, v1, v2, v3)) {
            return t;
          }
          return 10000.0;
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

          int cylinder_face = -1;

          // the path tracing depth
          for (int i = 0; i < 5; i++) {
            // finding the nearest hit point by checking intersection with every spheres
            float dist = 10000.0;
            vec2 dc0 = box_hit(ro, rd, cmin0, cmax0);
            float d0 = sphere_hit(ro, rd,sc0, sr0);
            float d1 = cylinder_hit(ro, rd, cc0, ch0, cr0, cylinder_face);
            float d2 = y_triangle_hit(ro, rd, ty0, tv00, tv01, tv02);
            float d3 = z_plane_hit(ro, rd, pz);
            float d4 = y_plane_hit(ro, rd, py);

            vec3 normal;
            vec3 diffuse;

            if (dc0.x > 0. && dc0.x < dc0.y && dist > dc0.x) dist = dc0.x;
            if (dist > d0) dist = d0;
            if (d1 > 0. && dist > d1) dist = d1;
            if (d2 > 0. && dist > d2) dist = d2;
            if (d3 > 0. && dist > d3) dist = d3;
            if (d4 > 0. && dist > d4) dist = d4;

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
              diffuse = sd0;
              if (cylinder_face == 0) {
                vec3 n = (hit - vec3(cc0.x, 0., cc0.z)) / cr0;
                normal = vec3(n.x, 0., n.z);
              } else if (cylinder_face == 1) {
                normal = vec3(0,1,0);
              } else {
                normal = vec3(0,-1,0);
              }
            } else if (dist == d2) {
              diffuse = sd0;
              normal = vec3(0,1,0);
            } else if (dist == d3) {
              diffuse = pzd;
              normal = vec3(0.,0.,-1.);
            } else if (dist == d4) {
              diffuse = pyd;
              normal = vec3(0.,1.,0.);
            }

            if (dist == d3 || dist == d4) {
              rd = cosineWeightedDirection(u_time, normal);
            } else {
              rd = customWeightedDirection(u_time, reflect(rd, normal));
            }
            ro = hit + 0.0001 * normal;

            color_mask *= diffuse;
            accu_color *= calc_color(hit, normal, color_mask);
          }
          return vec4(accu_color, 1.);
        }

        const float screen_dist = 6.;

        void main() {
          // s,t is the x,y axis of the screen-space coordinate,
          // first assuming that there is a screen that lies in the world-space with its center at (0,0,0)
          // and the plane of screen is parallel to the xy plane
          vec4 st = vec4(vPosition.xy, 0., 1.);
          st.y = -st.y;

          // then, convert the screen to world-space where a camera position and direction is determined
          // this is done by an inverse matrix operation of gluLookAt
          // first change the basis, then translate to the world-space

          vec4 rd = u_screen_to_world * st;
          vec4 result = trace(u_camera_pos, rd.xyz);
          
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

    const planeBuffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, planeBuffer);
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

    gl.useProgram(tracerProgram);

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

      gl.bindBuffer(gl.ARRAY_BUFFER, planeBuffer);

      time+=deltaTime;
      self.sampleCount++;
      // time uniform
      let tul = gl.getUniformLocation(tracerProgram, 'u_time');
      gl.uniform1f(tul, time);

      // sample weight
      let swul = gl.getUniformLocation(tracerProgram, 'u_sw');
      gl.uniform1f(swul, self.sampleCount / (self.sampleCount + 1));

      // camera pos uniform
      let cpul = gl.getUniformLocation(tracerProgram, 'u_camera_pos');
      gl.uniform3f(cpul, self.cameraPos.x, self.cameraPos.y, self.cameraPos.z);

      // screen to world matrix uniform
      const screenToWorldMatrix = self.getScreenToWorldMatrix();
      let swmul = gl.getUniformLocation(tracerProgram, 'u_screen_to_world');
      gl.uniformMatrix4fv(swmul, false, screenToWorldMatrix);

      // for progressive rendering, two textures are alteratively used to store the current path trace result
      gl.bindTexture(gl.TEXTURE_2D, textures[0]);
      gl.bindFramebuffer(gl.FRAMEBUFFER, fb);
      gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, textures[1], 0);
      gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);
      gl.bindFramebuffer(gl.FRAMEBUFFER, null);

      textures.reverse();

      // render to canvas
      gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);

      window.requestAnimationFrame((timestamp) => draw(timestamp));
    }
    window.requestAnimationFrame((timestamp) => draw(timestamp));
  },
  methods: {
    showHowToPlay() {
      if (this.cameraMode === 'free') {
        alert(`
      How to play on PC or Mac
        1. move mouse inside to look around
        2. use WSAD key to walk around

      How to play on mobile devices
        1. touch and move inside to look around
        2. use two-finger gesture inside to walk around
        `);
      } else {
        alert(`
      How to play on PC or Mac
        1. move mouse inside to rotate camera around the origin

      How to play on mobile devices
        1. touch inside to rotate camera around the origin
        `);
      }
    },
    getScreenToWorldMatrix() {
      let v = [
        this.cameraDir.x,
        this.cameraDir.y,
        this.cameraDir.z
      ];

      const up = [0.0,1.0,0.0];

      let r = this.crossProduct(
        v[0],v[1],v[2],
        up[0],up[1],up[2]
      );

      r = this.normalize(...r);

      let u = this.crossProduct(
        v[0],v[1],v[2],
        r[0],r[1],r[2]
      );

      const result = new Float32Array([
        r[0],r[1],r[2],0.0,
        u[0],u[1],u[2],0.0,
        v[0],v[1],v[2],0.0,
        this.screenDist*v[0],this.screenDist*v[1],this.screenDist*v[2],1.0
      ]);
      return result;
    },
    normalize(x,y,z) {
      const len = Math.sqrt(x*x+y*y+z*z);
      return [ x/len, y/len, z/len ];
    },
    // methods for implementing the movable camera
    crossProduct(x1,y1,z1,x2,y2,z2) {
      return [
        y1*z2-z1*y2,
        z1*x2-x1*z2,
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
      const key = e.key;
      const speed = 0.1;
      if (key === 'w') {
        this.sampleCount = 0;
        if (this.cameraMode === 'free') {
          this.cameraPos.x += speed * this.cameraDir.x;
          this.cameraPos.y += speed * this.cameraDir.y;
          this.cameraPos.z += speed * this.cameraDir.z;

          this.screenOrigin.x = this.cameraPos.x + this.screenDist * this.cameraDir.x;
          this.screenOrigin.y = this.cameraPos.y + this.screenDist * this.cameraDir.y;
          this.screenOrigin.z = this.cameraPos.z + this.screenDist * this.cameraDir.z;
        }
      } else if (key === 's') {
        this.sampleCount = 0;
        if (this.cameraMode === 'free') {
          this.cameraPos.x -= speed * this.cameraDir.x;
          this.cameraPos.y -= speed * this.cameraDir.y;
          this.cameraPos.z -= speed * this.cameraDir.z;

          this.screenOrigin.x = this.cameraPos.x + this.screenDist * this.cameraDir.x;
          this.screenOrigin.y = this.cameraPos.y + this.screenDist * this.cameraDir.y;
          this.screenOrigin.z = this.cameraPos.z + this.screenDist * this.cameraDir.z;
        }
      } else if (key === 'a') {
        if (this.cameraMode === 'free') {
          this.sampleCount = 0;
          const r = this.crossProduct(this.cameraDir.x, this.cameraDir.y, this.cameraDir.z, 0,1,0);
          this.cameraPos.x -= speed * r[0];
          this.cameraPos.y -= speed * r[1];
          this.cameraPos.z -= speed * r[2];

          this.screenOrigin.x = this.cameraPos.x + this.screenDist * this.cameraDir.x;
          this.screenOrigin.y = this.cameraPos.y + this.screenDist * this.cameraDir.y;
          this.screenOrigin.z = this.cameraPos.z + this.screenDist * this.cameraDir.z;
        }
      } else if (key === 'd') {
        if (this.cameraMode === 'free') {
          this.sampleCount = 0;
          const r = this.crossProduct(this.cameraDir.x, this.cameraDir.y, this.cameraDir.z, 0,1,0);
          this.cameraPos.x += speed * r[0];
          this.cameraPos.y += speed * r[1];
          this.cameraPos.z += speed * r[2];

          this.screenOrigin.x = this.cameraPos.x + this.screenDist * this.cameraDir.x;
          this.screenOrigin.y = this.cameraPos.y + this.screenDist * this.cameraDir.y;
          this.screenOrigin.z = this.cameraPos.z + this.screenDist * this.cameraDir.z;
        }
      } else if (key === 'f') {
        if (this.cameraMode === 'free') {
          this.cameraMode = 'focus';
        } else {
          this.cameraMode = 'free';
        }
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

      if (this.cameraMode === 'focus') {
        this.cameraPos.x = this.screenOrigin.x - this.screenDist * this.cameraDir.x;
        this.cameraPos.y = this.screenOrigin.y - this.screenDist * this.cameraDir.y;
        this.cameraPos.z = this.screenOrigin.z - this.screenDist * this.cameraDir.z;
      } else {
        this.screenOrigin.x = this.cameraPos.x + this.screenDist * this.cameraDir.x;
        this.screenOrigin.y = this.cameraPos.y + this.screenDist * this.cameraDir.y;
        this.screenOrigin.z = this.cameraPos.z + this.screenDist * this.cameraDir.z;
      }
    },
    handleTouchMove(e) {
      e.preventDefault();

      this.sampleCount = 0;
      let x = (e.touches[0].clientX - this.canvas.left ) / this.w - 0.5;
      let y = (e.touches[0].clientY - this.canvas.top ) / this.h - 0.5;

      y = -y;

      if (this.cameraMode === 'free') {
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
      } else {
        this.cameraDir.pitch += 0.02*y;
        this.cameraDir.yaw += 0.02*x;
        
        this.cameraDir.z = -Math.cos(this.cameraDir.pitch) * Math.cos(this.cameraDir.yaw);
        this.cameraDir.y = Math.sin(this.cameraDir.pitch);
        this.cameraDir.x = Math.cos(this.cameraDir.pitch) * Math.sin(this.cameraDir.yaw);

        this.cameraPos.x = this.screenOrigin.x - this.screenDist * this.cameraDir.x;
        this.cameraPos.y = this.screenOrigin.y - this.screenDist * this.cameraDir.y;
        this.cameraPos.z = this.screenOrigin.z - this.screenDist * this.cameraDir.z;
      }
    }
  }
};
</script>

<style>
body {
  margin: 0;
  user-select: none;
}
#app {
  width: 100vw;
  overflow: hidden;
}
</style>
