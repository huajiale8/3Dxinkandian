<script setup lang="ts">
import * as THREE from 'three';
import {Octree} from "three/examples/jsm/math/Octree";
import {Capsule} from "three/examples/jsm/math/Capsule";
import {GLTFLoader} from "three/examples/jsm/loaders/GLTFLoader";
import {onMounted, ref} from "vue";
import Stats from "three/examples/jsm/libs/stats.module";

const clock = new THREE.Clock();
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x88ccee);
const camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.rotation.order = 'YXZ';
// 添加环境光w
const ambientLight = new THREE.AmbientLight(0xffffff, 1); // 参数1：光的颜色，参数2：光的强度
scene.add(ambientLight);

const light = new THREE.DirectionalLight(0xffffff, 1);
light.castShadow = true;
light.position.set(2, 6, 0);
scene.add(light);
// 添加平行光
// const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
// directionalLight.position.set(10, 10, 10);
// scene.add(directionalLight);


onMounted(() => {
    const container = document.getElementById('container');
    const renderer = new THREE.WebGLRenderer({
        antialias: true,
        alpha:true

    });
    renderer.autoClear = true
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.VSMShadowMap;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    container.appendChild(renderer.domElement);
    const stats = new Stats();


    container.appendChild(stats.domElement);
    const geometry = new THREE.CapsuleGeometry(0.35, 0.7, 4, 8);
    // 材料
    const material = new THREE.MeshBasicMaterial({
        color: 0x00ff00,
        wireframe: true,
        transparent: true, // 启用透明度
        alphaTest: 0.5, // 控制透明度的阈值
        visible: false,
    });
    const capsule = new THREE.Mesh(geometry, material);
    scene.add(capsule);
    const GRAVITY = 30;
    const STEPS_PER_FRAME = 5;
    let peopleObj;
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
        actionWalk.stop()
        actionIdle.play()
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


    function crossPlay(curAction, newAction, duration = 0.5) {
        // 如果当前动画为空或者正在播放的动画和新动画相同，则不执行渐变
        if (!curAction || curAction === newAction) {
            return;
        }
        curAction.fadeOut(duration);
        newAction.reset();
        newAction.setEffectiveWeight(1);
        newAction.play();
        newAction.fadeIn(duration);
    }


    const loader = new GLTFLoader().setPath('../../src/assets/models/');
    loader.load('kongjianzhanx.glb?t=' + Math.random(), (gltf) => {
        let obj = gltf.scene;
        // const baseColorTexture = new THREE.TextureLoader().load('../../src/assets/baseMap.jpg?t=' + Math.random());
        // const lightMap = new THREE.TextureLoader().load('../../src/assets/lightMaplight.png?t=' + Math.random());
        // obj.traverse((node) => {
        //     const text =  new THREE.MeshStandardMaterial
        //     if (node.isMesh && node.name === "xiuxiqu_qiang") {
        //         text.map = baseColorTexture
        //         text.lightMap = lightMap
        //         // text.emissive = new THREE.Color(0xffffff)
        //         // text.emissiveIntensity = 0.5
        //         text.lightMapIntensity  = 1
        //         node.material = text;
        //         // text.wireframe = true
        //     }
        // });
        gltf.scene.traverse((child) => {
            child.castShadow = true;
            child.receiveShadow = true;
        });
        obj.remove(obj.getObjectByName("polySurface150")!);
        scene.add(obj);
        console.log(obj)
        worldOctree.fromGraphNode(obj);
        animate();
    });

    let playerMixer;
    let actionIdle;
    let actionWalk;
    new GLTFLoader().load('../../src/assets/models/Xbot.glb', (gltf) => {
        peopleObj = gltf.scene;
        peopleObj.scale.set(1, 1, 1);
        gltf.scene.traverse((child) => {
            child.castShadow = true;
            child.receiveShadow = true;
        });
        camera.position.set(0, 2, -3); //相机位置
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
        playerMixer = new THREE.AnimationMixer(peopleObj);
        const clipIdle = THREE.AnimationUtils.subclip(gltf.animations[0], 'idle', 0, 250);
        actionIdle = playerMixer.clipAction(clipIdle);
        actionIdle.play();
        const clipWalk = THREE.AnimationUtils.subclip(gltf.animations[6], 'walk', 0, 250);
        actionWalk = playerMixer.clipAction(clipWalk);
    })

    function animate() {
        const deltaTime = Math.min(0.05, clock.getDelta()) / STEPS_PER_FRAME;
        for (let i = 0; i < STEPS_PER_FRAME; i++) {
            controls(deltaTime);
            updatePlayer(deltaTime);
        }
        renderer.render(scene, camera);
        stats.update();
        // 更新动画混合器
        if (playerMixer) {
            playerMixer.update(0.015);
        }

        requestAnimationFrame(animate);
        window.addEventListener('resize', onWindowResize);
        // 旋转人物模型
    }

    let prePos;
    let flag = ref(false)
// 鼠标按住时触发移动操作
    const startMoving = () => {
        window.addEventListener('mousemove', onMouseMove);
    };

// 鼠标松开时停止移动操作
    const stopMoving = () => {
        window.removeEventListener('mousemove', onMouseMove);
        prePos = null;
    };


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
        capsule.rotation.y = camera.rotation.y + currentRotation;
        const speedDelta = deltaTime * (playerOnFloor ? 10 : 1);
        let playNewAnimation = false;
        if (keyStates['KeyW']) {

            playerVelocity.add(getForwardVector().multiplyScalar(-speedDelta));
            actionWalk.play()
        }
        if (keyStates['KeyS']) {
            playerVelocity.add(getForwardVector().multiplyScalar(speedDelta));
            actionWalk.play()
        }
        if (keyStates['KeyA']) {
            playerVelocity.add(getSideVector().multiplyScalar(speedDelta));
            actionWalk.play()
        }
        if (keyStates['KeyD']) {
            playerVelocity.add(getSideVector().multiplyScalar(-speedDelta));
            actionWalk.play()
        }
        if (playerOnFloor) {
            if (keyStates['Space']) {
                actionWalk.play()
                playerVelocity.y = 8;
            }
        }
    }

    // 鼠标移动事件处理函数
    // 在顶部定义一个变量来保存当前的角度
    let currentRotation = 0;

    const onMouseMove = (e) => {
        if (prePos) {
            const rotationChange = -(e.clientX - prePos) * 0.01;
            peopleObj.rotateY(rotationChange);
            currentRotation += rotationChange;
        }
        prePos = e.clientX;
    };

    window.addEventListener('mousedown', () => {
        flag.value = true;
        startMoving();
    });

    window.addEventListener('mouseup', () => {
        flag.value = false;
        stopMoving();
    });

// 鼠标离开窗口时也停止移动
    window.addEventListener('mouseout', () => {
        if (flag.value) {
            flag.value = false;
            stopMoving();
        }
    });

})
</script>

<template>
    <div id="container"></div>
</template>
