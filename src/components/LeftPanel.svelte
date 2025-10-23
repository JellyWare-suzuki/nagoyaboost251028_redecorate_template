<script>
    import { onMount } from 'svelte';
    import * as THREE from 'three';
    import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
    import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
    import { SpotLightHelper } from 'three'; // ← ★ ここを examples ではなく core から
    import { gridState } from '$lib/stores/gridState';

    let container;
    let scene, camera, renderer, controls;
    let gridObjects = []; // グリッド上のオブジェクトを格納する配列
    let isInitialized = false; // 初期化フラグ

    // モデルURL
    const placementModelUrls = {
        model1: 'https://9lr9bdlzhrgweq2j.public.blob.vercel-storage.com/redecorate/04f28cb21ce9_orange_rice__3d_asset_0_glb.glb',
        model2: 'https://9lr9bdlzhrgweq2j.public.blob.vercel-storage.com/redecorate/08f541aa2b91_orange_bread__3d_asset_0_glb.glb',
        model3: 'https://9lr9bdlzhrgweq2j.public.blob.vercel-storage.com/redecorate/fa49e19f6df7_orange_pumpkin__3d_asset_0_glb.glb'
    };

    function loadModel(url) {
        return new Promise((resolve, reject) => {
            const loader = new GLTFLoader();
            loader.load(
                url,
                (gltf) => resolve(gltf.scene),
                undefined,
                (error) => {
                    console.error('モデルロード中にエラーが発生しました:', error);
                    reject(error);
                }
            );
        });
    }

    const modelCache = { model1: null, model2: null, model3: null };

    async function preloadModels() {
        try {
            const [m1, m2, m3] = await Promise.all([
                loadModel(placementModelUrls.model1),
                loadModel(placementModelUrls.model2),
                loadModel(placementModelUrls.model3)
            ]);
            modelCache.model1 = m1;
            modelCache.model2 = m2;
            modelCache.model3 = m3;
            console.log('すべてのモデルをプリロードしました');
        } catch (error) {
            console.error('モデルのプリロード中にエラーが発生しました:', error);
        }
    }

    async function createStateObject(state) {
        if (state === 0) return null;

        let modelType;
        let rotationDegrees;
        if (state >= 1 && state <= 4) {
            modelType = 'model1';
            rotationDegrees = (state - 1) * 90;
        } else if (state >= 5 && state <= 8) {
            modelType = 'model2';
            rotationDegrees = (state - 5) * 90;
        } else if (state >= 9 && state <= 12) {
            modelType = 'model3';
            rotationDegrees = (state - 9) * 90;
        } else {
            return null;
        }

        const base = modelCache[modelType] ?? (modelCache[modelType] = await loadModel(placementModelUrls[modelType]));
        const model = base.clone();

        // 影の設定
        model.traverse((obj) => {
            if (obj.isMesh) {
                obj.castShadow = true;
                obj.receiveShadow = true;
            }
        });

        model.rotation.y = THREE.MathUtils.degToRad(rotationDegrees);
        return model;
    }

    async function updateGridObjects(states) {
        if (!isInitialized || !scene) return;

        const gridSize = 8;
        for (let i = 0; i < states.length; i++) {
            const state = states[i];
            const x = i % gridSize;
            const z = Math.floor(i / gridSize);

            if (gridObjects[i]) {
                scene.remove(gridObjects[i]);
                gridObjects[i] = null;
            }

            const newObject = await createStateObject(state);
            if (newObject) {
                newObject.scale.set(0.5, 0.5, 0.5);

                const bbox = new THREE.Box3().setFromObject(newObject);
                const modelHeight = bbox.max.y - bbox.min.y;

                newObject.position.x = x - gridSize / 2 + 0.5;
                newObject.position.y = modelHeight * 0.5;
                newObject.position.z = z - gridSize / 2 + 0.5;

                scene.add(newObject);
                gridObjects[i] = newObject;
            }
        }
    }

    $: if (isInitialized) {
        updateGridObjects($gridState);
    }

    onMount(() => {
        scene = new THREE.Scene();
        scene.background = new THREE.Color(0xf5f2e8);

        camera = new THREE.PerspectiveCamera(75, 1, 0.1, 1000);

        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(container.clientWidth, container.clientHeight);
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFSoftShadowMap;
        container.appendChild(renderer.domElement);

        camera.position.set(0, 10, 10);
        camera.lookAt(0, 0, 0);

        controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.05;

        // 🌙 ムーディな紫色のスポットライト
        const spotlight = new THREE.SpotLight(0x9b59b6, 1.2);
        spotlight.position.set(5, 10, 5);
        spotlight.angle = Math.PI / 6;
        spotlight.penumbra = 0.5;
        spotlight.decay = 2;
        spotlight.distance = 30;
        spotlight.castShadow = true;
        scene.add(spotlight);

        spotlight.target.position.set(0, 0, 0);
        scene.add(spotlight.target);

        // デバッグ可視化（不要なら次の2行を削除）
        const helper = new SpotLightHelper(spotlight);
        scene.add(helper);

        // 環境光は控えめにしてムードを維持
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.45);
        scene.add(ambientLight);

        // グリッド
        const gridSize = 8;
        const gridHelper = new THREE.GridHelper(gridSize, gridSize, 0x8f8f8f, 0x8f8f8f);
        scene.add(gridHelper);

        // 床
        const planeGeometry = new THREE.PlaneGeometry(gridSize, gridSize);
        const planeMaterial = new THREE.MeshStandardMaterial({
            color: 0xffffff,
            side: THREE.DoubleSide,
            roughness: 0.85,
            metalness: 0.1
        });
        const plane = new THREE.Mesh(planeGeometry, planeMaterial);
        plane.rotation.x = Math.PI / 2;
        plane.position.y = -0.01;
        plane.receiveShadow = true;
        scene.add(plane);

        // 8x8セル
        const cellSize = 1;
        const cellGeometry = new THREE.PlaneGeometry(cellSize * 0.95, cellSize * 0.95);
        const colors = [0xf0f0f0, 0xe0e0e0];
        for (let x = 0; x < gridSize; x++) {
            for (let z = 0; z < gridSize; z++) {
                const index = z * gridSize + x;
                const colorIndex = (x + z) % 2;
                const cellMaterial = new THREE.MeshStandardMaterial({
                    color: colors[colorIndex],
                    side: THREE.DoubleSide,
                    roughness: 0.75,
                    metalness: 0.1
                });
                const cell = new THREE.Mesh(cellGeometry, cellMaterial);
                cell.rotation.x = Math.PI / 2;
                cell.position.x = x - gridSize / 2 + 0.5;
                cell.position.y = 0;
                cell.position.z = z - gridSize / 2 + 0.5;
                cell.receiveShadow = true;
                scene.add(cell);
                gridObjects[index] = null;
            }
        }

        preloadModels();
        isInitialized = true;
        updateGridObjects($gridState);

        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            renderer.render(scene, camera);
        }

        function handleResize() {
            camera.aspect = container.clientWidth / container.clientHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(container.clientWidth, container.clientHeight);
        }
        window.addEventListener('resize', handleResize);

        animate();

        return () => {
            window.removeEventListener('resize', handleResize);
            if (container && container.contains(renderer.domElement)) {
                container.removeChild(renderer.domElement);
            }
            isInitialized = false;
        };
    });
</script>

<div class="left-content">
    <div class="canvas-container" bind:this={container}></div>
</div>

<style>
    .left-content {
        padding: 20px;
        height: 80%;
        display: flex;
        flex-direction: column;
    }
    .canvas-container {
        flex-grow: 1;
        min-height: 300px;
        width: 100%;
    }
</style>
