<script setup lang="ts">
import * as THREE from 'three';
import {Octree} from "three/examples/jsm/math/Octree";
import {Capsule} from "three/examples/jsm/math/Capsule";
import {GLTFLoader} from "three/examples/jsm/loaders/GLTFLoader";
import {onMounted, ref} from "vue";
import Stats from "three/examples/jsm/libs/stats.module";
import * as dat from 'lil-gui'
import _ from 'lodash'
import {RectAreaLightHelper} from "three/examples/jsm/helpers/RectAreaLightHelper";

// const gui = new dat.GUI()
let keyV = false

const clock = new THREE.Clock();
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x88ccee);
const camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.rotation.order = 'YXZ';
const renderer = new THREE.WebGLRenderer({
    antialias: true,
    alpha: true
});
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
let prePos;
let flag = ref(false)
onMounted(() => {
    const container = document.getElementById('container');
    container.appendChild(renderer.domElement);
    const stats = new Stats()
    container.appendChild(stats.domElement);
    const geometry = new THREE.CapsuleGeometry(0.35, 0.7, 4, 8);
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

    const loader = new GLTFLoader().setPath('../../src/assets/models/');
    loader.load('kongjianzhan9.7.1517.glb', (gltf) => {
        let obj = gltf.scene;
        gltf.scene.traverse((child) => {
            child.castShadow = true; //投射阴影
            child.receiveShadow = true; //接收影子
        });
        obj.traverse((child: any) => {
            if (child.name === 'kaola') {
                console.log(child)
            }
            if (child.name === 'yanjing') {
                const newMaterial = child.material.clone()
                newMaterial.transparent = true
                newMaterial.opacity = 0.5
                child.material = newMaterial
            }
        });
        obj.remove(obj.getObjectByName("polySurface150")!);
        scene.add(obj);
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
        camera.position.set(0, 2, -3);
        camera.lookAt(new THREE.Vector3(0, 2, 3));
        peopleObj.add(camera)
        const bbox = new THREE.Box3().setFromObject(peopleObj);
        const center = new THREE.Vector3();
        bbox.getCenter(center);
        peopleObj.position.sub(center);
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
        if (playerMixer) {
            playerMixer.update(0.015);
        }
        requestAnimationFrame(animate);
        window.addEventListener('resize', onWindowResize);
    }

    const startMoving = () => {
        window.addEventListener('mousemove', onMouseMove);
    };
    const stopMoving = () => {
        window.removeEventListener('mousemove', onMouseMove);
        prePos = null;
    };

    function updatePlayer(deltaTime) {
        let damping = Math.exp(-10 * deltaTime) - 1;
        if (!playerOnFloor) {
            playerVelocity.y -= GRAVITY * deltaTime;
            damping *= 0.1;
        }
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

    function getForwardVector() {
        capsule.getWorldDirection(playerDirection);
        playerDirection.y = 0;
        playerDirection.normalize();
        return playerDirection;
    }

    function getSideVector() {
        capsule.getWorldDirection(playerDirection);
        playerDirection.y = 0;
        playerDirection.normalize();
        playerDirection.cross(capsule.up);
        return playerDirection;
    }


    function controls(deltaTime) {
        capsule.rotation.y = camera.rotation.y + currentRotation;
        const speedDelta = deltaTime * (playerOnFloor ? 18 : 1);
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
        // if (keyStates['KeyV']) {
        //     // _.throttle(() => {
        //     //     if (keyV) {
        //     //         camera.position.set(peopleObj.position.x, peopleObj.position.y, 0)
        //     //     }
        //     //     keyV = !keyV
        //     //     console.log(keyV)
        //     // }, 1000)()
        // }
        if (playerOnFloor) {
            if (keyStates['Space']) {
                actionWalk.play()
                playerVelocity.y = 8;
            }
        }
    }


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
    window.addEventListener('mouseout', () => {
        if (flag.value) {
            flag.value = false;
            stopMoving();
        }
    });
    document.addEventListener('keydown', (event) => {
        keyStates[event.code] = true;
    });
    document.addEventListener('keyup', (event) => {
        keyStates[event.code] = false;
        actionWalk.stop()
        actionIdle.play()
    });
    window.addEventListener('resize', onWindowResize);
    light()

    window.addEventListener('dblclick', () => {
        const fullscreenElement = document.fullscreenElement || document.webkitFullscreenElement
        if (!fullscreenElement) {
            if (container.requestFullscreen) {
                container.requestFullscreen()
            } else if (container.webkitRequestFullscreen) {
                container.webkitRequestFullscreen()
            }
        } else {
            if (!document.exitFullscreen) {
                document.exitFullscreen()
            } else if (document.webkitExitFullscreen) {
                document.webkitExitFullscreen()
            }
        }
    })
})
const light = () => {

    const lightOptions = [{
        position: {x: 9, y: 4.352, z: 2},
        castShadow: true
    }, {
        position: {x: 5.5, y: 4.352, z: 8.8},
        castShadow: true
    }, {
        position: {x: 5.5, y: 4.352, z: -3.5},
        castShadow: true
    }, {
        position: {x: -6.9, y: 5.352, z: -9.8},
        castShadow: true
    },

    ]

    const ambientLight = new THREE.AmbientLight(0xffffff, 1); // 参数1：光的颜色，参数2：光的强度
    scene.add(ambientLight);

    // 星星左上角方块灯
    const rectLight = new THREE.RectAreaLight(0xffffff, 1, 3, 3.1);
    rectLight.position.set(13.7, 7.1, 10.5);
    rectLight.rotation.set(Math.PI * 0.5, 0, 0)
    scene.add(rectLight)
    // 沙发左上角方块灯
    const rectLight2 = new THREE.RectAreaLight(0xffffff, 1, 3, 3.05);
    rectLight2.position.set(-12.8, 6.25, -10.37);
    rectLight2.rotation.set(Math.PI * 1.5, 0, 0)
    scene.add(rectLight2)
    const rectLightHelper = new RectAreaLightHelper(rectLight2);
    scene.add(rectLightHelper);

    // wecome上方方块灯
    const rectLight3 = new THREE.RectAreaLight(0xffffff, 1, 3, 3.05);
    rectLight3.position.set(-12.9, 6.25, -0.395);
    rectLight3.rotation.set(Math.PI * 1.5, 0, 0)
    scene.add(rectLight3)
    const rectLightHelper3 = new RectAreaLightHelper(rectLight3);
    scene.add(rectLightHelper3);

    // 一号条形灯
    const rectLight4 = new THREE.RectAreaLight(0xffffff, 1, 6, 0.4);
    rectLight4.position.set(7.3, 6.5, 14.2);
    rectLight4.rotation.set(Math.PI * 1.5, 0, 0)
    scene.add(rectLight4)
    const rectLightHelper4 = new RectAreaLightHelper(rectLight4);
    scene.add(rectLightHelper4);

// 二号条形灯
    const rectLight5 = new THREE.RectAreaLight(0xffffff, 1, 0.8, 13.3);
    rectLight5.position.set(14.8, 7.1, 0.25);
    rectLight5.rotation.set(Math.PI * 1.5, 0, 0)
    scene.add(rectLight5)
    const rectLightHelper5 = new RectAreaLightHelper(rectLight5);
    scene.add(rectLightHelper5);

    // 三号条形灯
    const rectLight6 = new THREE.RectAreaLight(0xffffff, 1, 5, 0.4);
    rectLight6.position.set(12.2, 7.1, -9.5);
    rectLight6.rotation.set(Math.PI * 1.5, 0, 0)
    scene.add(rectLight6)
    const rectLightHelper6 = new RectAreaLightHelper(rectLight6);
    scene.add(rectLightHelper6);
    // 四号条形灯
    const rectLight7 = new THREE.RectAreaLight(0xffffff, 1, 0.4, 4.6);
    rectLight7.position.set(8.1, 7.15, -12.8);
    rectLight7.rotation.set(Math.PI * 1.5, 0, 0)
    scene.add(rectLight7)
    const rectLightHelper7 = new RectAreaLightHelper(rectLight7);
    scene.add(rectLightHelper7);
    // 五号条形灯
    const rectLight8 = new THREE.RectAreaLight(0xffffff, 1, 12.5, 0.6);
    rectLight8.position.set(0, 7.05, -15);
    rectLight8.rotation.set(Math.PI * 1.5, 0, 0)
    scene.add(rectLight8)
    const rectLightHelper8 = new RectAreaLightHelper(rectLight8);
    scene.add(rectLightHelper8);
    // 六号条形灯
    const rectLight9 = new THREE.RectAreaLight(0xffffff, 3, 0.6, 18.5);
    rectLight9.position.set(-8.1, 7.1, 4.55);
    rectLight9.rotation.set(Math.PI * 1.5, 0, 0)
    scene.add(rectLight9)
    const rectLightHelper9 = new RectAreaLightHelper(rectLight9);
    scene.add(rectLightHelper9);


    lightOptions.forEach((item, index) => {
        const light = new THREE.PointLight(0xffffff, 8, 7, 1);
        light.position.set(item.position.x, item.position.y, item.position.z);
        light.castShadow = item.castShadow
        scene.add(light);
    })
    // 平行光
    const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
    directionalLight.position.set(0, 6, -7);
    scene.add(directionalLight);

    // 白色聚光灯从侧面照射，经过纹理调节，形成阴影

    const spotLight = new THREE.SpotLight(0xffffff, 8);
    spotLight.position.set(0.9, 2.87, -9.97);
    spotLight.rotation.y = Math.PI * 0.25
    spotLight.castShadow = false;
    spotLight.intensity = 42
    spotLight.distance = 7
    spotLight.angle = 0.51
    spotLight.penumbra = 0.9
    spotLight.target.position.set(3.61, 1.64, -11.84)
    scene.add(spotLight.target)
    scene.add(spotLight);

    // const lightHelper = new THREE.SpotLightHelper(spotLight)
    // scene.add(lightHelper)
    //
    //
    // gui.add(spotLight.position, 'x').min(-10).max(10).step(0.01).name('position x')
    // gui.add(spotLight.position, 'y').min(-10).max(10).step(0.01).name('position y')
    // gui.add(spotLight.position, 'z').min(-15).max(10).step(0.01).name('position z')
    //
    // gui.add(spotLight.target.position, 'x').min(-10).max(10).step(0.01).name('target x')
    // gui.add(spotLight.target.position, 'y').min(-10).max(10).step(0.01).name('target y')
    // gui.add(spotLight.target.position, 'z').min(-15).max(10).step(0.01).name('target z')
    //
    // gui.add(spotLight.rotation, 'x').min(-10).max(10).step(0.01).name('rotation x')
    // gui.add(spotLight.rotation, 'y').min(-10).max(10).step(0.01).name('rotation y')
    // gui.add(spotLight.rotation, 'z').min(-15).max(10).step(0.01).name('rotation z')
    //
    // gui.add(spotLight, 'intensity').min(0).max(100).step(0.01).name('intensity')
    // gui.add(spotLight, 'distance').min(0).max(10).step(0.01).name('distance')
    // gui.add(spotLight, 'angle').min(0).max(Math.PI / 2).step(0.01).name('angle')
    // gui.add(spotLight, 'penumbra').min(0).max(10).step(0.01).name('penumbra')
    // gui.add(spotLight, 'decay').min(0).max(10).step(0.01).name('decay')

}


function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
}
</script>
<template>
    <div id="container"></div>
</template>
