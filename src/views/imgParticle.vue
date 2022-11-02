<!--
threejs imgParticle.vue
@author kongjianqiu
@description
@created 2022/11/2 15:29:23
-->
<template>
  <div id="imgParticle" class="imgParticle">
    <canvas id="canvas2"></canvas>
  </div>
</template>

<script>
// threeJs插件
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";

let img, canvas, ctx, width, height, imgData, numPixels, originalColors, numVisible = 0, threshold = 100, offsets;
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
    getParticleData() {
      // 统计大于阈值的像素点
      for (let i = 0; i < numPixels; i++) {
        if (originalColors[i * 4 + 0] > threshold) numVisible++;
      }
      offsets = new Float32Array(numVisible * 3);
      // 获取像素点的位置
      for (let i = 0, j = 0; i < numPixels; i++) {
        if (originalColors[i * 4 + 0] > threshold) {
          // 获取 x 方向的坐标
          offsets[j * 3 + 0] = i % width;
          // 获取 y 方向的坐标
          offsets[j * 3 + 1] = Math.floor(i / width);
          j++;
        }
      }
    },
    initThree() {
      scene = new THREE.Scene();
      camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 1, 1000);
      camera.position.z = 300;
      let fovHeight = 2 * Math.tan(camera.fov * Math.PI / 180 / 2) * camera.position.z;
      let canvas2 = document.getElementById("canvas2")
      canvas2.width = window.innerWidth
      canvas2.height = window.innerHeight
      renderer = new THREE.WebGLRenderer({
        canvas: canvas2,
        antialias: true,
        alpha: true
      });
      renderer.setClearColor(0x000000, 1);
      controls = new OrbitControls(camera, renderer.domElement);
    },
    initMesh() {
      group = new THREE.Group();
      let textureLoader = new THREE.TextureLoader();
      let map = textureLoader.load(this.imgUrl2);
      let material = new THREE.SpriteMaterial({
        map,
        color: 0xffffff,
        fog: true
      });
      let positions = offsets;
      for (let index = 0; index < positions.length; index += 2) {
        let particleMaterial = material;
        let particle = new THREE.Sprite(particleMaterial);
        // 粒子目标位置
        let targetX = positions[index];
        let targetY = positions[index + 1];
        let targetZ = positions[index + 2];

        if (targetX && targetY) {
          particle.position.x = targetX;
          particle.position.y = targetY;
          particle.position.z = targetZ;
          /* // 粒子的初始位置
           particle.position.x = 0;
           particle.position.y = 0;
           particle.position.z = 0;
           // 粒子从初始位置过渡到目标位置
           TweenMax.to(particle.position, 1, {
             x: targetX,
             y: targetY,
             z: targetZ,
             delay: Math.random() * 0.1
           });*/
          group.add(particle);
        }
      }
      scene.add(group);
    },
    initImg(imgUrl) {
      img = document.createElement("img");
      img.src = imgUrl;
      img.onload = () => {
        canvas = document.createElement("canvas");
        ctx = canvas.getContext("2d");
        width = img.width;
        height = img.height;
        numPixels = width * height;
        canvas.width = width;
        canvas.height = height;
        ctx.scale(1, -1);
        ctx.drawImage(img, 0, 0, width, height * -1);
        imgData = ctx.getImageData(0, 0, canvas.width, canvas.height);
        originalColors = Float32Array.from(imgData.data);
        this.getParticleData();
        this.initThree();
        this.initMesh();
        this.animate()
      };
    },
    animate() {
      controls.update()
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
