<script setup lang="ts">
import * as THREE from 'three';
import {OrbitControls} from "three/examples/jsm/controls/OrbitControls";
import {Octree} from "three/examples/jsm/math/Octree";
import {Capsule} from "three/examples/jsm/math/Capsule";
import {GLTFLoader} from "three/examples/jsm/loaders/GLTFLoader";
import {onMounted} from "vue";
import Stats from "three/examples/jsm/libs/stats.module";

const clock = new THREE.Clock();
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x88ccee);
const camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.rotation.order = 'YXZ';
// 添加环境光w
const ambientLight = new THREE.AmbientLight(0xffffff, 0.3); // 参数1：光的颜色，参数2：光的强度
scene.add(ambientLight);

const fillLight1 = new THREE.HemisphereLight(0xffffff, 1);
fillLight1.position.set(10, 10, 10);
scene.add(fillLight1);


onMounted(() => {
    const container = document.getElementById('container');
    const renderer = new THREE.WebGLRenderer({
        antialias: true
    });
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.VSMShadowMap;
    renderer.outputEncoding = THREE.sRGBEncoding;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    container.appendChild(renderer.domElement);

    const oControls = new OrbitControls(camera, container);
    oControls.update();

    const stats = new Stats();
    container.appendChild(stats.domElement);
    const geometry = new THREE.CapsuleGeometry(0.35, 0.7, 4, 8);
    // 材料
    const material = new THREE.MeshBasicMaterial({
        color: 0x00ff00,
        wireframe: true,
        transparent: true, // 启用透明度
        alphaTest: 0.5 // 控制透明度的阈值
    });
    const capsule = new THREE.Mesh(geometry, material);
    // scene.add(capsule);
    const GRAVITY = 30;
    const STEPS_PER_FRAME = 5;

    let peopleObj, activeAction, peopleAnimations, tween, mixer, peopleBox;
    let mixerArr = [];
    const worldOctree = new Octree();
    const playerCollider = new Capsule(new THREE.Vector3(0, 0.35, 0), new THREE.Vector3(0, 1, 0), 0.26);
    const playerVelocity = new THREE.Vector3();
    const playerDirection = new THREE.Vector3();
    let playerOnFloor = false;
    const keyStates = {};

    document.addEventListener('keydown', (event) => {
        keyStates[event.code] = true;
    });
    document.addEventListener('keyup', (event) => {
        keyStates[event.code] = false;
    });
    window.addEventListener('resize', onWindowResize);

    function onWindowResize() {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
    }

    // 玩家碰撞
    function playerCollisions() {
        const result = worldOctree.capsuleIntersect(playerCollider);
        playerOnFloor = false;
        if (result) {
            playerOnFloor = result.normal.y > 0;
            if (!playerOnFloor) {
                playerVelocity.addScaledVector(result.normal, -result.normal.dot(playerVelocity));
            }
            playerCollider.translate(result.normal.multiplyScalar(result.depth));
        }
    }

    // 更新播放器
    function updatePlayer(deltaTime) {
        let damping = Math.exp(-4 * deltaTime) - 1;
        if (!playerOnFloor) {
            playerVelocity.y -= GRAVITY * deltaTime;
            damping *= 0.1;
        }
        // 玩家速度 阻尼
        playerVelocity.addScaledVector(playerVelocity, damping);
        const deltaPosition = playerVelocity.clone().multiplyScalar(deltaTime);
        playerCollider.translate(deltaPosition);
        playerCollisions();
        let resPos = playerCollider.end;
        capsule.position.copy(resPos);
        if (peopleObj) {
            peopleObj.position.copy(resPos.clone().setY(resPos.y - 0.7));
            let pos = peopleObj.position.clone();
            pos = pos.setY(pos.y)
            oControls.target = pos;
            oControls.update();
        } else {
            oControls.object.position.set(0, 1, -2)
            oControls.update(true); // 初始化
        }
    }

    // 获取前向矢量
    function getForwardVector() {
        capsule.getWorldDirection(playerDirection);
        playerDirection.y = 0;
        playerDirection.normalize();
        return playerDirection;
    }

    // 获取侧向量
    function getSideVector() {
        capsule.getWorldDirection(playerDirection);
        playerDirection.y = 0;
        playerDirection.normalize();
        playerDirection.cross(capsule.up);
        return playerDirection;
    }

    // 控制
    function controls(deltaTime) {
        capsule.rotation.y = camera.rotation.y;
        const speedDelta = deltaTime * (playerOnFloor ? 25 : 8);
        if (keyStates['KeyW']) {
            playerVelocity.add(getForwardVector().multiplyScalar(-speedDelta));
        }
        if (keyStates['KeyS']) {
            playerVelocity.add(getForwardVector().multiplyScalar(speedDelta));
        }
        if (keyStates['KeyA']) {
            playerVelocity.add(getSideVector().multiplyScalar(speedDelta));
        }
        if (keyStates['KeyD']) {
            playerVelocity.add(getSideVector().multiplyScalar(-speedDelta));
        }
        if (playerOnFloor) {
            if (keyStates['Space']) {
                playerVelocity.y = 8;
            }
        }
    }

    const loader = new GLTFLoader().setPath('../../src/assets/models/');
    loader.load('kongjianzhans.glb', (gltf) => {
        let obj = gltf.scene;
        obj.remove(obj.getObjectByName("polySurface150")!);
        scene.add(obj);
        worldOctree.fromGraphNode(obj);
        gltf.scene.traverse(child => {
            if (child.isMesh) {
                child.castShadow = true;
                child.receiveShadow = true;
                if (child.material.map) {
                    child.material.map.anisotropy = 4;
                }
            }
        });
        animate();
    });
    new GLTFLoader().load('../../src/assets/models/Xbot.glb', (gltf) => {
        peopleObj = gltf.scene;
        peopleObj.scale.set(1, 1, 1);
        peopleAnimations = gltf.animations;

        camera.position.set(0, 2, -6); //相机位置
        camera.lookAt(new THREE.Vector3(0, 2, 3));
        peopleObj.add(camera)

        const bbox = new THREE.Box3().setFromObject(peopleObj);
        // 获取包围盒的中心点
        const center = new THREE.Vector3();
        bbox.getCenter(center);

        // 将物体移动到中心点
        peopleObj.position.sub(center);
        // 组合对象添加到场景中
        scene.add(peopleObj);
        mixer = new THREE.AnimationMixer(peopleObj);
        mixerArr.push(mixer)
        activeAction = mixer.clipAction(peopleAnimations[1]);
        activeAction.play();
    })

    function animate() {
        const deltaTime = Math.min(0.05, clock.getDelta()) / STEPS_PER_FRAME;
        for (let i = 0; i < STEPS_PER_FRAME; i++) {
            controls(deltaTime);
            updatePlayer(deltaTime);
        }
        renderer.render(scene, camera);
        stats.update();
        requestAnimationFrame(animate);
        window.addEventListener('resize', onWindowResize);
    }
})
</script>

<template>
    <div id="container"></div>
</template>
