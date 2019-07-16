# WebGL Path Tracer

This is a Vue tiny project and a demo for demostrating path tracing with movable camera.
See https://lqt0223.github.io for the deployed demo.

## How to run

1. install `vue-cli` and `vue-cli instant prototyping addon` globally. Follow the instruction on https://cli.vuejs.org/guide/installation.html and https://cli.vuejs.org/guide/prototyping.html
2. clone the repository
3. `cd ./webgl-path-tracer`
4. disable vue-cli lint globally by `vue config --set useEsLint false` (The debugging of shader program require console output, which is prohibited by lint rules :( )
5. `vue serve App.vue`
