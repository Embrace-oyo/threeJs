<!--
threejs wave.vue
@author kongjianqiu
@description
@created 2022/10/31 18:06:11
-->
<template>
  <div class="wave" id="wave">
    <button v-if="!isLoad" id="startButton">Play</button>
  </div>
</template>

<script>
// threeJs插件
import * as THREE from "three";
// FPS状态插件
import Stats from 'three/addons/libs/stats.module.js';
// 轨道控制器插件
import {OrbitControls} from "three/examples/jsm/controls/OrbitControls.js";
import {GUI} from "dat.gui";
import {RenderPass} from "three/addons/postprocessing/RenderPass";
import {UnrealBloomPass} from "three/addons/postprocessing/UnrealBloomPass";
import {EffectComposer} from "three/addons/postprocessing/EffectComposer";
import {ShaderPass} from "three/addons/postprocessing/ShaderPass";
import {CopyShader} from "three/addons/shaders/CopyShader";

const fffSize = 2048
let camera, scene, renderer, controls, stats, sphereGroup, audio, analyser, composer, radius;
radius = 6
let gui = {
  R: 20,
  G: 90,
  B: 225,
  TrianglesBgColor: 0x03a9f4,
  TrianglesLineColor: 0x03a9f4,
  SphereColor: 0x03a9f4,
  lineColor: 0x00ffff,
  rotate: false
};
export default {
  name: "wave",
  data() {
    return {
      isLoad: false
    };
  },
  methods: {
    initPlane() {
      let geometry = new THREE.PlaneGeometry(1000, 1000)
      let material = new THREE.MeshBasicMaterial({color: '#aaaaaa', side: THREE.DoubleSide});
      let plane = new THREE.Mesh(geometry, material)
      plane.position.y = -100
      plane.rotateX(-0.5 * Math.PI)
      scene.add(plane)
    },
    initCircle() {
      sphereGroup = new THREE.Group()
      let nums = 40
      let distance = 15
      for (let i = 0; i < nums; i++) {
        for (let j = 0; j < nums; j++) {
          let miniGroup = new THREE.Group()
          let sphere = new THREE.BoxGeometry(radius, radius, radius)
          let material = new THREE.MeshPhongMaterial({color: '#aaaaaa'});
          let mesh = new THREE.Mesh(sphere, material)
          mesh.position.set(i * distance - ((nums * distance) / 2), 0, j * distance - ((nums * distance) / 2))
          mesh.scale.set(1, 1, 1)
          miniGroup.add(mesh)
          sphereGroup.add(miniGroup)
        }
      }
      scene.add(sphereGroup)
    },
    // 加载音频
    initAudio() {
      // 监听者
      let listener = new THREE.AudioListener();
      // 非位置音频对象
      audio = new THREE.Audio(listener);
      // 音频资源路径
      let audioUrl = new URL("@/assets/music/Just_Run.mp3", import.meta.url).href;
      // 音频加载器
      let audioLoader = new THREE.AudioLoader();
      audioLoader.load(audioUrl, (AudioBuffer) => {
        let overlay = document.getElementById("startButton");
        overlay.remove();
        if (audio.isPlaying) {
          audio.stop();
          audio.setBuffer();
        }
        // 音频缓冲区对象关联到音频对象audio
        audio.setBuffer(AudioBuffer);
        //是否循环
        audio.setLoop(true);
        //音量
        audio.setVolume(0.6);
        //播放
        audio.play();
        // 音频分析器和音频绑定，可以实时采集音频时域数据进行快速傅里叶变换
        analyser = new THREE.AudioAnalyser(audio, fffSize * 2);
      });
    },
    // GUI控制显示
    initGui() {
      //声明一个保存需求修改的相关数据的对象
      let datGui = new GUI();
      //将设置属性添加到gui当中，gui.add(对象，属性，最小值，最大值）
      datGui.add(gui, "R", 0, 255);
      datGui.add(gui, "G", 0, 255);
      datGui.add(gui, "B", 0, 255);
      datGui.add(gui, "rotate").onChange(function (val) {
        controls.autoRotate = val;
      });
      /* datGui.addColor(gui, "SphereColor").onChange(function () {
         sphereGroup.traverse(function (child) {
           if (child.isMesh)
             child.material = new THREE.MeshPhongMaterial({
               color: gui.SphereColor
             });
         });
       });*/
    },
    // 环境光和平行光
    initLight() {
      scene.add(new THREE.AmbientLight(0x444444));
      let light = new THREE.PointLight(0xffffff);
      light.position.set(0, 100, 0);
      //告诉平行光需要开启阴影投射
      light.castShadow = true;
      scene.add(light);
    },
    // 辉光
    initBloomPass() {
      // 辉光
      let params = {
        exposure: 0.5,
        bloomStrength: 1,
        bloomThreshold: 0,
        bloomRadius: 0.8
      };
      let renderScene = new RenderPass(scene, camera);
      let bloomPass = new UnrealBloomPass(
          new THREE.Vector2(window.innerWidth, window.innerHeight),
          1.5,
          0.2,
          0
      );
      bloomPass.threshold = params.bloomThreshold;
      bloomPass.strength = params.bloomStrength;
      bloomPass.radius = params.bloomRadius;
      composer = new EffectComposer(renderer);
      // console.log(composer)
      const copyShader = new ShaderPass(CopyShader);
      copyShader.renderToScreen = true;
      composer.addPass(renderScene);
      composer.addPass(bloomPass);
      composer.addPass(copyShader);
      // 辉光 end
    },
    init() {
      scene = new THREE.Scene()
      // scene.add(new THREE.AxesHelper(100 * 5))
      camera = new THREE.PerspectiveCamera(45, 16 / 9, 1, 10000)
      camera.position.set(500, 500, 500)
      renderer = new THREE.WebGLRenderer({antialias: true});
      renderer.setPixelRatio(window.devicePixelRatio);
      renderer.setSize(window.innerWidth, window.innerHeight);
      let DOM = document.getElementById("wave");
      DOM.appendChild(renderer.domElement);
      controls = new OrbitControls(camera, renderer.domElement);
      controls.autoRotate = true
      stats = new Stats()
      DOM.appendChild(stats.dom)
      // this.initPlane()
      this.initCircle()
      // 播放音乐
      this.$nextTick(() => {
        let startButton = document.getElementById("startButton");
        startButton.addEventListener("click", () => {
          this.initAudio();
        });
      });
      this.initGui()
      this.initLight()
      this.initBloomPass()
    },
    animate() {
      controls.update();
      stats.update();
      if (analyser) {
        // 获得频率数据N个
        let arr = analyser.getFrequencyData();
        sphereGroup.children.forEach((item, index) => {
          item.children[0].position.y = arr[index] / 20
          if (arr[index] === 0) {
            item.scale.set(1, 1, 1)
          } else {
            let m = arr[index] / 10;
            let targetRange = Math.max(
                arr[index] / 10 - arr[index - 1] / 10,
                0
            );
            if (m < targetRange) {
              m = targetRange;
            }
            item.scale.set(1, m, 1);
          }
          // 同步GUI颜色
          if (gui.R) {
            item.children[0].material.color.r = arr[index] / (gui.R * 3);
          }
          if (gui.G) {
            item.children[0].material.color.g = arr[index] / (gui.G * 3);
          }
          if (gui.B) {
            item.children[0].material.color.b = arr[index] / (gui.B * 3);
          }
        })
      }
      // renderer.render(scene, camera);
      composer.render();
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

  button {
    position: absolute;
    font-size: 16px;
    z-index: 2;
    top: 10px;
    left: 50%;
    transform: translate(-50%, 0);
    display: flex;
    align-items: center;
    justify-content: center;
    flex-direction: column;
    background: rgba(0, 0, 0, 0.7);
    background: transparent;
    border: 0;
    border: 1px solid rgb(255, 255, 255);
    border-radius: 4px;
    color: #ffffff;
    padding: 12px 18px;
    text-transform: uppercase;
    cursor: pointer;
  }

  .loading {
    position: absolute;
    font-size: 16px;
    z-index: 2;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    background: transparent;
    border: 0;
    border: 1px solid rgb(255, 255, 255);
    border-radius: 4px;
    color: #ffffff;
    padding: 12px 18px;
    text-transform: uppercase;
    cursor: pointer;
  }
}
</style>
