<!--
threejs wave.vue
@author kongjianqiu
@description
@created 2022/10/31 18:06:11
-->
<template>
  <div class="wave" id="wave">

  </div>
</template>

<script>
// threeJs插件
import * as THREE from "three";
// FPS状态插件
import Stats from 'three/addons/libs/stats.module.js';
// 轨道控制器插件
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
const fffSize = 1024
let camera, scene, renderer,controls,stats;
export default {
  name: "wave",
  data() {
    return {};
  },
  methods: {
    init() {
      scene = new THREE.Scene()
      scene.add(new THREE.AxesHelper(100 * 5))
      /**
       * fov — 摄像机视锥体垂直视野角度
       * aspect — 摄像机视锥体长宽比
       * near — 摄像机视锥体近端面
       * far — 摄像机视锥体远端面
       * */
      camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 1, 10000)
      camera.position.z = 200;
      renderer = new THREE.WebGLRenderer({antialias: true});
      renderer.setPixelRatio(window.devicePixelRatio);
      renderer.setSize(window.innerWidth, window.innerHeight);
      let DOM = document.getElementById("wave");
      DOM.appendChild(renderer.domElement);
      controls = new OrbitControls(camera, renderer.domElement);
      stats = new Stats()
      DOM.appendChild(stats.dom)
    },
    animate() {
      controls.update();
      stats.update();
      renderer.render(scene, camera);
      requestAnimationFrame(this.animate.bind(this))
    }
  },
  mounted() {
    this.$nextTick(() => {
      this.init()
      this.animate()
    })
  },
};
</script>

<style scoped lang="less">
.wave {
  position: absolute;
  left: 0;
  right: 0;
  top: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0);
}
</style>
