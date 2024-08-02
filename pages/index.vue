<script setup>
import { onMounted, ref } from 'vue';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { ARButton } from 'three/addons/webxr/ARButton.js';
import { TextGeometry } from 'three/addons/geometries/TextGeometry.js';
import { FontLoader } from 'three/addons/loaders/FontLoader.js';
import * as THREE from 'three';

const container = ref(null);
let camera, scene, renderer;
let controller;
let reticle;
let object;
let hitTestSource = null;
let hitTestSourceRequested = false;
let textMesh;
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

onMounted(() => {
    init();
    animate();
});

function init() {
    scene = new THREE.Scene();

    camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.01, 1000);
    camera.position.z = 5;

    const light = new THREE.HemisphereLight(0xffffff, 0xbbbbff, 3);
    light.position.set(0.5, 1, 0.25);
    scene.add(light);

    renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.xr.enabled = true;
    container.value.appendChild(renderer.domElement);

    document.body.appendChild(ARButton.createButton(renderer, { requiredFeatures: ['hit-test'] }));

    const loader = new GLTFLoader();
    loader.load(
        '/food/scene.gltf',
        (gltf) => {
            object = gltf.scene;
            object.visible = true;
        },
        undefined,
        (error) => {
            console.error('An error happened while loading the GLTF model:', error);
        }
    );

    const LoaderFont = new FontLoader();
    LoaderFont.load('/fonts/roboto-regular.json', (font) => {
        const textGeometry = new TextGeometry('Click Me!', {
            font: font,
            size: 1,
            height: 0.2,
        });

        // Create a material and mesh
        const textMaterial = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
        textMesh = new THREE.Mesh(textGeometry, textMaterial);
        scene.add(textMesh);

        // Center the text
        textGeometry.center();
    }, undefined, (error) => {
        console.error('Error loading font:', error);
    });

    function onSelect() {
        if (reticle.visible && object) {
            const placedObject = object.clone();
            placedObject.position.setFromMatrixPosition(reticle.matrix);
            placedObject.visible = true; // Ensure the cloned object is visible
            placedObject.scale.set(0.1, 0.1, 0.1);
            scene.add(placedObject);
        }
    }

    controller = renderer.xr.getController(0);
    controller.addEventListener('select', onSelect);
    scene.add(controller);

    reticle = new THREE.Mesh(
        new THREE.RingGeometry(0.15, 0.2, 32).rotateX(-Math.PI / 2),
        new THREE.MeshBasicMaterial({ color: 0x00ff00 })
    );
    reticle.matrixAutoUpdate = false;
    reticle.visible = false;
    scene.add(reticle);

    window.addEventListener('resize', onWindowResize);
    renderer.domElement.addEventListener('click', onMouseClick);
}

function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
}

function onMouseClick(event) {
    // Convert mouse position to normalized device coordinates
    const rect = renderer.domElement.getBoundingClientRect();
    mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

    raycaster.updateMatrix();
    raycaster.ray.origin.setFromMatrixPosition(camera.matrixWorld);
    raycaster.ray.direction.set(mouse.x, mouse.y, 1).unproject(camera).sub(raycaster.ray.origin).normalize();

    const intersects = raycaster.intersectObject(textMesh);

    if (intersects.length > 0) {
        alert('Text clicked!');
    }
}

function animate() {
    renderer.setAnimationLoop(render);
}

function render(timestamp, frame) {
    if (frame) {
        const referenceSpace = renderer.xr.getReferenceSpace();
        const session = renderer.xr.getSession();

        if (!hitTestSourceRequested) {
            session.requestReferenceSpace('viewer').then((referenceSpace) => {
                session.requestHitTestSource({ space: referenceSpace }).then((source) => {
                    hitTestSource = source;
                });
            });

            session.addEventListener('end', () => {
                hitTestSourceRequested = false;
                hitTestSource = null;
            });

            hitTestSourceRequested = true;
        }

        if (hitTestSource) {
            const hitTestResults = frame.getHitTestResults(hitTestSource);

            if (hitTestResults.length) {
                const hit = hitTestResults[0];
                reticle.visible = true;
                reticle.matrix.fromArray(hit.getPose(referenceSpace).transform.matrix);
            } else {
                reticle.visible = false;
            }
        }
    }

    renderer.render(scene, camera);
}
</script>
<template>
    <div ref="container"></div>
</template>
