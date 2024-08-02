<script setup>
import * as THREE from 'three';
import { TextGeometry } from 'three/addons/geometries/TextGeometry.js';
import { FontLoader } from 'three/addons/loaders/FontLoader.js';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { onMounted } from 'vue';

const container3D = ref(null);

onMounted(() => {
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.01, 1000);
    camera.position.z = 5;

    const renderer = new THREE.WebGLRenderer({ alpha: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    container3D.value.appendChild(renderer.domElement);

    const LoaderFont = new FontLoader();
    LoaderFont.load('/fonts/roboto-regular.json', (font) => {
        const textGeometry = new TextGeometry('Hello Dunia', {
            font: font,
            size:1,
            height: 0.2,
        });

        // Create a material and mesh
        const textMaterial = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
        const textMesh = new THREE.Mesh(textGeometry, textMaterial);
        scene.add(textMesh);

        // Center the text
        textGeometry.center();

        // Render the scene
        animate();
    }, undefined, (error) => {
        console.error('Error loading font:', error);
    });

    const controls = new OrbitControls(camera, renderer.domElement);
    function animate() {
        requestAnimationFrame(animate);
        controls.update();
        renderer.render(scene, camera);
    }

    window.addEventListener('resize', onWindowResize);

    function onWindowResize() {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
    }

    animate();

    onBeforeUnmount(() => {
        window.removeEventListener('resize', onWindowResize);
        if (mixer) mixer.stopAllAction();
        renderer.dispose();
    });
})
</script>

<template>
    <div ref="container3D"></div>
</template>