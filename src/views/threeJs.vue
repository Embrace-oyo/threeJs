<template>
  <div id="threeJs" class="threeJs">
    <div v-if="!isLoad" id="overlay">
      <button id="startButton">Play</button>
    </div>
    <div v-if="isLoad" class="loading">loading...</div>
  </div>
</template>

<script>
// threeJs插件
import * as THREE from "three";

// 轨道控制器插件
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";

// 效果合成器
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer.js";

// 渲染通道
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass.js";

// UnrealBloomPass通道
import { UnrealBloomPass } from "three/examples/jsm/postprocessing/UnrealBloomPass.js";

// ShaderPass通道
import { ShaderPass } from "three/examples/jsm/postprocessing/ShaderPass.js";

// CopyShader
import { CopyShader } from "three/examples/jsm/shaders/CopyShader.js";

// 用于在 JavaScript 中更改变量的轻量级图形用户界面
import { GUI } from "dat.gui";

// 用于在 JavaScript 中更改变量的轻量级图形用户界面
import Stats from "three/examples/jsm/libs/stats.module.js";

import { range } from "@/utils/range.js";

import { node } from "@/utils/node.js";

import { Triangle } from "@/utils/Triangle.js";

import { randomRange } from "@/utils/randomRange.js";

let renderer, scene, camera, textureCube, controls, stats, composer;

let gui = {
  R: 20,
  G: 90,
  B: 225,
  TrianglesBgColor: 0x03a9f4,
  TrianglesLineColor: 0x03a9f4,
  lineColor: 0x00ffff,
  rotate: false
};

let audio, analyser; // 音频
let linesGroup,
  outLine,
  inLine,
  barLine = [],
  barNodes; // 线
let barGroup; // 柱子
let Triangles = [],
  TriangleGroup;


export default {
  name: "threeJs",
  data() {
    return {
      positionZ: 80,
      N: 256,
      clock: new THREE.Clock(),
      scale: 1,
      isLoad: true
    };
  },
  methods: {
    // 资源初始化
    initResource() {
      this.initRenderer();
      this.initScene();
      this.initCamera();
    },
    // 初始化
    init() {
      window.addEventListener("resize", this.windowResize, false);
      // 添加鼠标控制
      this.initControls();
      // 添加FPS状态显示
      this.initStats();
      // 播放音乐
      this.$nextTick(() => {
        let startButton = document.getElementById("startButton");
        startButton.addEventListener("click", () => {
          this.initAudio();
        });
      });
      // 添加音频线
      this.audioLines(20, this.N);
      // 添加音频柱子
      this.audioBars(25, this.N / 2);
      // 添加三角形
      this.initTriangle();
      // 添加环境光和平行光
      this.initLight();
      // 添加辉光
      this.initBloomPass();
      // 添加GUI控制显示
      this.initGui();
      // 开始绘制
      this.animate();
    },
    // 初始化渲染器
    initRenderer() {
      // 创建渲染器 antialias（是否执行抗锯齿） alpha（控制默认清除 alpha 值。设置为 true 时，值为 0。否则为 1。默认值为 false）
      renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
      // 设置alpha。合法参数是一个0.0到 1.0之间的浮点数
      renderer.setClearAlpha(0);
      // 将输出canvas的大小调整为(width, height)并考虑设备像素比，且将视口从(0, 0)开始调整到适合大小 将updateStyle设置为false以阻止对canvas的样式做任何改变。
      renderer.setSize(window.innerWidth, window.innerHeight);
      // 网页插入渲染器dom
      let DOM = document.getElementById("threeJs");
      DOM.appendChild(renderer.domElement);
    },
    // 初始化场景
    initScene() {
      // 构建场景
      scene = new THREE.Scene();
    },
    // 初始化相机与场景加载贴图
    initCamera() {
      // 贴图数据
      let urls = [
        new URL("@/assets/skybox/right.jpg", import.meta.url).href,
        new URL("@/assets/skybox/left.jpg", import.meta.url).href,
        new URL("@/assets/skybox/top.jpg", import.meta.url).href,
        new URL("@/assets/skybox/bottom.jpg", import.meta.url).href,
        new URL("@/assets/skybox/front.jpg", import.meta.url).href,
        new URL("@/assets/skybox/back.jpg", import.meta.url).href
      ];
      // 加载场景贴图
      textureCube = new THREE.CubeTextureLoader().load(urls, () => {
        this.isLoad = false;
        textureCube.mapping = THREE.CubeRefractionMapping;
        scene.background = textureCube;
        // 创建摄影机
        camera = new THREE.PerspectiveCamera(
          75,
          window.innerWidth / window.innerHeight,
          1,
          10000
        );
        // 设置摄影机
        camera.position.z = this.positionZ;
        this.init();
      });
    },
    // 鼠标控制
    initControls() {
      controls = new OrbitControls(camera, renderer.domElement);
      // 如果使用animate方法时，将此函数删除
      //controls.addEventListener( 'change', render );
      // 使动画循环使用时阻尼或自转 意思是否有惯性
      controls.enableDamping = true;
      //动态阻尼系数 就是鼠标拖拽旋转灵敏度
      //controls.dampingFactor = 0.25;
      //是否可以缩放
      controls.enableZoom = true;
      //设置相机距离原点的最远距离
      controls.minDistance = 1;
      //设置相机距离原点的最远距离
      controls.maxDistance = 200;
      //是否开启右键拖拽
      controls.enablePan = false;
      //是否自动旋转
      controls.autoRotate = gui.rotate;
      // controls.autoRotate = 0.1;
    },
    // FPS显示
    initStats() {
      stats = new Stats();
      let DOM = document.getElementById("threeJs");
      DOM.appendChild(stats.dom);
    },
    // 加载音频
    initAudio() {
      // 监听者
      let listener = new THREE.AudioListener();
      // 非位置音频对象
      audio = new THREE.Audio(listener);
      // 音频资源路径
      let audioUrl = new URL("@/assets/music/abc.mp4", import.meta.url).href;
      // 音频加载器
      let audioLoader = new THREE.AudioLoader();
      audioLoader.load(audioUrl, (AudioBuffer) => {
        let overlay = document.getElementById("overlay");
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
        audio.setVolume(1);
        //播放
        audio.play();
        // 音频分析器和音频绑定，可以实时采集音频时域数据进行快速傅里叶变换
        analyser = new THREE.AudioAnalyser(audio, this.N * 2);
      });
    },
    // 渲染几何
    renderGeometries(vertices) {
      const res = [];
      vertices = vertices.concat(vertices[0]);
      vertices.forEach(value => {
        res.push(value.x, value.y, 0);
      });
      return new THREE.BufferAttribute(new Float32Array(res), 3);
    },
    // 音频线
    audioLines(radius, countData) {
      // 创建数组
      let array = range(0, countData);
      // 数组填入默认数据
      barNodes = array.map(index => {
        return new node(
          radius,
          ((index / countData) * 360 + 45) % 360,
          new THREE.Vector2(0, 0)
        );
      });
      // 创建基础线条材质
      const lineMaterial = new THREE.LineBasicMaterial({
        color: gui.lineColor
      });
      barLine = range(0, countData).map(index => {
        return new THREE.Line(
          new THREE.BufferGeometry().setAttribute(
            "position",
            this.renderGeometries([
              barNodes[index].positionA(),
              barNodes[index].positionB()
            ])
          ),
          lineMaterial
        );
      });
      outLine = new THREE.Line(
        new THREE.BufferGeometry().setAttribute(
          "position",
          this.renderGeometries(barNodes.map(node => node.positionA()))
        ),
        lineMaterial
      );
      inLine = new THREE.Line(
        new THREE.BufferGeometry().setAttribute(
          "position",
          this.renderGeometries(barNodes.map(node => node.positionB()))
        ),
        lineMaterial
      );
      linesGroup = new THREE.Group();
      linesGroup.add(outLine);
      linesGroup.add(inLine);
      barLine.forEach(line => linesGroup.add(line));
      scene.add(linesGroup);
    },
    // 音频柱子
    audioBars(radius, countData) {
      barGroup = new THREE.Group();
      let R = radius;
      let N = countData;
      for (let i = 0; i < N; i++) {
        let minGroup = new THREE.Group();
        let box = new THREE.BoxGeometry(1, 1, 1);
        let material = new THREE.MeshPhongMaterial({
          color: 0x00ffff
        }); // 材质对象
        let m = i;
        let mesh = new THREE.Mesh(box, material);
        mesh.position.y = 0.5;
        minGroup.add(mesh);
        minGroup.position.set(
          Math.sin(((m * Math.PI) / N) * 2) * R,
          Math.cos(((m * Math.PI) / N) * 2) * R,
          0
        );
        minGroup.rotation.z = ((-m * Math.PI) / N) * 2;
        barGroup.add(minGroup);
      }
      scene.add(barGroup);
    },
    // 三角形
    initTriangle() {
      // 创建三角形组合
      TriangleGroup = new THREE.Group();
      // 每500毫秒创建一个render(三角形
      setInterval(this.addTriangle.bind(this), 500);
      // 场景加入三角形
      scene.add(TriangleGroup);
    },
    // 添加三角形
    addTriangle() {
      // 创建基础网格材质
      const material = new THREE.MeshBasicMaterial({
        color: gui.TrianglesBgColor
      });
      // 创建基础线条材质
      const lineMaterial = new THREE.LineBasicMaterial({
        color: gui.TrianglesLineColor
      });
      const triangle = this.makeTriangle(material, lineMaterial, t => {
        Triangles = Triangles.filter(triangle => {
          return triangle !== t;
        });
        TriangleGroup.remove(t.group);
      });
      TriangleGroup.add(triangle.group);
      Triangles.push(triangle);
    },
    // 创建三角形
    makeTriangle(material, lineMaterial, t) {
      const triangle = new Triangle(
        2,
        new THREE.Vector3(0, 0, 0),
        Math.random() * 360,
        randomRange(5, 1),
        randomRange(0.1, 0.05),
        material,
        lineMaterial,
        {
          startShow: 15,
          endShow: 30,
          startHide: 60,
          endHide: 70
        },
        t
      );
      return triangle;
    },
    // 自适应屏幕
    windowResize() {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
      composer.setSize(window.innerWidth, window.innerHeight);
    },
    // 环境光和平行光
    initLight() {
      scene.add(new THREE.AmbientLight(0x444444));
      let light = new THREE.PointLight(0xffffff);
      light.position.set(80, 100, 50);
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
    // GUI控制显示
    initGui() {
      //声明一个保存需求修改的相关数据的对象
      let datGui = new GUI();
      //将设置属性添加到gui当中，gui.add(对象，属性，最小值，最大值）
      datGui.add(gui, "R", 0, 255);
      datGui.add(gui, "G", 0, 255);
      datGui.add(gui, "B", 0, 255);
      datGui.add(gui, "rotate").onChange(function(val) {
        controls.autoRotate = val;
      });
      datGui.addColor(gui, "TrianglesBgColor").onChange(function() {
        TriangleGroup.traverse(function(child) {
          if (child.isMesh)
            child.material = new THREE.MeshPhongMaterial({
              color: gui.TrianglesBgColor
            });
        });
      });
      datGui.addColor(gui, "TrianglesLineColor").onChange(function() {
        TriangleGroup.traverse(function(child) {
          if (child.isLine)
            child.material = new THREE.LineBasicMaterial({
              color: gui.TrianglesLineColor
            });
        });
      });
      datGui.addColor(gui, "lineColor").onChange(function() {
        linesGroup.traverse(function(child) {
          if (child.isLine)
            child.material = new THREE.LineBasicMaterial({
              color: gui.lineColor
            });
        });
      });
    },
    // 更新圆
    updateCircle() {
      if (barNodes) {
        linesGroup.scale.set(this.scale, this.scale, this.scale);
        const geometryA = outLine.geometry;
        const AttributeA = geometryA.getAttribute("position");
        const geometryB = inLine.geometry;
        const AttributeB = geometryB.getAttribute("position");
        const positions = barNodes.map(value => {
          return [value.positionA(), value.positionB()];
        });
        positions.forEach((position, index) => {
          AttributeA.set([position[0].x, position[0].y], index * 3);
          AttributeB.set([position[1].x, position[1].y], index * 3);
          const geometry = barLine[index].geometry;
          const Attribute = geometry.getAttribute("position");
          Attribute.set(
            [position[0].x, position[0].y, 0, position[1].x, position[1].y, 0],
            0
          );
          Attribute.needsUpdate = true;
        });
        AttributeA.set(
          [AttributeA.array[0], AttributeA.array[1]],
          positions.length * 3
        );
        AttributeB.set(
          [AttributeB.array[0], AttributeB.array[1]],
          positions.length * 3
        );
        AttributeA.needsUpdate = true;
        AttributeB.needsUpdate = true;
      }
    },
    // 绘制帧
    animate() {
      controls.update();
      stats.update();
      // 获取音频解析数据（fffSize-傅里叶变换）
      if (analyser) {
        // 获得频率数据N个
        let arr = analyser.getFrequencyData();
        if (barGroup) {
          // 绕z轴旋转
          barGroup.rotation.z += 0.002;
          // 重置缩放设置
          barGroup.scale.set(this.scale, this.scale, this.scale);
          // barGroup遍历
          barGroup.children.forEach((elem, index) => {
            // 同步GUI颜色
            if (gui.R) {
              elem.children[0].material.color.r = arr[index] / (gui.R * 3);
            }
            if (gui.G) {
              elem.children[0].material.color.g = arr[index] / (gui.G * 3);
            }
            if (gui.B) {
              elem.children[0].material.color.b = arr[index] / (gui.B * 3);
            }
            // 同步GUI颜色 结束
            // 设置缩放
            if (arr[index] === 0) {
              // 当前arr[index]音频数据为0时设置为默认缩放
              elem.scale.set(1, 1, 1);
            } else {
              let m = arr[index] / 20;
              let targetRange = Math.max(
                arr[index] / 20 - arr[index - 1] / 20,
                0
              );
              if (m < targetRange) {
                m = targetRange;
              }
              elem.scale.set(1, m, 1);
            }
          });
        }
        const Delta = this.clock.getDelta();
        barNodes.forEach((node, index, array) => {
          node.strength(arr[index % array.length] * 0.1);
          node.transition(Delta);
        });
        this.scale = 1 + arr[Math.ceil(arr.length * 0.05)] / 500;
        this.updateCircle(arr);
        Triangles.forEach(triangle => triangle.transition(Delta));
      }
      // renderer.render(scene, camera);
      composer.render();
      requestAnimationFrame(this.animate);
    }
  },
  mounted() {
    this.initResource();
  }
};
</script>

<style lang="less" scoped>
.threeJs {
  position: absolute;
  left: 0;
  right: 0;
  top: 0;
  bottom: 0;
  background-color: #000;

  #overlay {
    position: absolute;
    font-size: 16px;
    z-index: 2;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-direction: column;
    background: rgba(0, 0, 0, 0.7);

    button {
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
