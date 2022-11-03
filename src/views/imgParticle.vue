<!--
threejs imgParticle.vue
@author kongjianqiu
@description
@created 2022/11/2 15:29:23
-->
<template>
  <div id="imgParticle" class="imgParticle">
  </div>
</template>

<script>
// threeJs插件
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";

let img, canvas, ctx, imgData, particles, size = 1, cw, ch, points, loaded;
let scene, group, camera, renderer, controls;
export default {
  name: "imgParticle",
  data() {
    return {
      imgUrl: new URL("@/assets/img/WechatIMG499.jpeg", import.meta.url).href,
      imgUrl2: new URL("@/assets/img/123.png", import.meta.url).href,
      elementBox: null
    };
  },
  methods: {
    initImg(imgUrl) {
      canvas = document.createElement("canvas");
      ctx = canvas.getContext("2d");
      img = new Image();
      img.src = imgUrl;
      img.onload = () => {
        canvas.width = window.innerWidth;
        let canvasRate = img.width / img.height;
        canvas.height = canvas.width / canvasRate;
        console.log(canvasRate);
        ctx.drawImage(img, 0, 0, canvas.width, (canvas.width / canvasRate));
        imgData = ctx.getImageData(0, 0, canvas.width, canvas.height);
        this.getParticleData();
        this.initThree();
        this.animate();
      };
    },
    getParticleData() {
      if (points) {
        scene.remove(points);
      }
      cw = Math.floor(canvas.width / size);
      ch = Math.floor(canvas.height / size);

      // 计算图片总像素点
      particles = canvas.width * canvas.height;
    },
    initThree() {
      scene = new THREE.Scene();
      camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 1, 3000);
      camera.position.z = 500;
      renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
      renderer.setClearAlpha(0);
      renderer.setSize(window.innerWidth, window.innerHeight);
      controls = new OrbitControls(camera, renderer.domElement);
      let dom = document.getElementById("imgParticle");
      dom.appendChild(renderer.domElement);
      let geometry = new THREE.BufferGeometry();
      let positions = new Float32Array(Math.floor(particles * 3));
      // let positions_af = new Float32Array(Math.floor(particles * 3));
      let colors = new Float32Array(Math.floor(particles * 3));
      let s = 3;
      // 坐标起始点
      let pxStart = -canvas.width / 2;
      let pyStart = canvas.height / 2;
      for (let i = 0; i < positions.length; i += 1) {
        positions[s * i] = pxStart + (i % cw) * size;
        positions[s * i + 1] = pyStart + Math.floor((-1 - i) / cw) * size;
        positions[s * i + 2] = 0;
        let selectPos = size * (i % cw) + Math.floor(i / cw) * cw * size * size;
        colors[s * i] = imgData.data[4 * selectPos] / 255.0;
        colors[s * i + 1] = imgData.data[4 * selectPos + 1] / 255.0;
        colors[s * i + 2] = imgData.data[4 * selectPos + 2] / 255.0;
      }
      geometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));
      geometry.setAttribute("color", new THREE.BufferAttribute(colors, 3));
      geometry.dynamic = true;
      geometry.attributes.position.needsUpdate = true;
      let material = new THREE.PointsMaterial({ size: size, vertexColors: true });
      points = new THREE.Points(geometry, material);
      points.name = "points";
      scene.add(points);
    },
    animate() {
      controls.update();
      renderer.render(scene, camera);
      requestAnimationFrame(this.animate);
    }
  },
  mounted() {
    this.$nextTick(() => {
      this.elementBox = document.getElementById("imgParticle");
      this.initImg(this.imgUrl);
    });
  },
  created() {
  }
};
</script>

<style lang="less" scoped>
.imgParticle {
  position: absolute;
  left: 0;
  right: 0;
  top: 0;
  bottom: 0;
  background-color: #aaaaaa;
}
</style>
