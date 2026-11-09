
<script lang="ts">
    import { onMount } from "svelte";
    import * as THREE from "three";
    import { FontLoader, Font } from "three/addons/loaders/FontLoader.js";
    import { TextGeometry } from "three/addons/geometries/TextGeometry.js";
    import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
    import type { TMovie } from "../../types";
    import { addGround, onWindowResize, loadModels } from "$lib/Helper-3D";
    import { base } from "$app/paths";
    import * as d3 from "d3";

    let movies: TMovie[] = $state([]);

    // Stacked data: each year has counts per genre
    type TStackRow = {
        year: string;
        Comedy: number;
        Romance: number;
        Drama: number;
    };

    let stackData: TStackRow[] = $state([]);
    const genres = ["Comedy", "Romance", "Drama"];
    const genreColors: Record<string, number> = {
        Comedy: 0xf4a261,
        Romance: 0xe76f51,
        Drama: 0x2a9d8f,
    };

    // === TODO-1: Load Data ===
    async function loadCsv() {
        const response = await fetch(`${base}/summer_movies.csv`);

        if (!response.ok) {
            throw new Error(`Failed to load CSV: ${response.status}`);
        }

        const csvData = await response.text();

        const parsedMovies = d3.csvParse(csvData, (row) => ({
            ...row,
            year: new Date(row.year),
        })) as unknown as TMovie[];

        movies = parsedMovies;

        const yearGenreCounts = new Map<string, TStackRow>();

        movies.forEach((movie) => {
            if (
                !(movie.year instanceof Date) ||
                Number.isNaN(movie.year.getTime())
            ) {
                return;
            }

            const year = movie.year.getFullYear().toString();

            if (!yearGenreCounts.has(year)) {
                yearGenreCounts.set(year, {
                    year,
                    Comedy: 0,
                    Romance: 0,
                    Drama: 0,
                });
            }

            const row = yearGenreCounts.get(year)!;

            const movieGenres = String(movie.genres ?? "")
                .split(",")
                .map((genre) => genre.trim());

            genres.forEach((genre) => {
                if (movieGenres.includes(genre)) {
                    row[genre as keyof Omit<TStackRow, "year">]++;
                }
            });
        });

        stackData = Array.from(yearGenreCounts.values()).sort(
            (a, b) => Number(a.year) - Number(b.year),
        );

        console.log("Loaded movies:", movies.length);
        console.log("Stack data:", stackData);
    }
    // === END TODO-1 ===

    let container: HTMLElement;
    let camera: THREE.PerspectiveCamera;
    let scene: THREE.Scene;
    let renderer: THREE.WebGLRenderer;
    const FLOOR = -250;

    const morphs: Array<THREE.Mesh> = [];
    let mixer: THREE.AnimationMixer;
    const clock = new THREE.Clock();

    const horseMorphs: Array<THREE.Mesh> = [];
    let horseMixer: THREE.AnimationMixer;
    const ORBIT_RX = 850;
    const ORBIT_RZ = 400;

    let animating = $state(false);

    let sun: THREE.Mesh | undefined;

    onMount(async () => {
        await loadCsv();
        init(window.innerWidth, window.innerHeight);
    });

    function init(SCREEN_WIDTH: number, SCREEN_HEIGHT: number) {
        // Renderer
        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.setSize(SCREEN_WIDTH, SCREEN_HEIGHT);
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFShadowMap;
        container.appendChild(renderer.domElement);

        // Camera
        camera = new THREE.PerspectiveCamera(
            23,
            SCREEN_WIDTH / SCREEN_HEIGHT,
            10,
            5000,
        );
        camera.position.set(0, 100, 2800);

        // Scene
        scene = new THREE.Scene();

        // Animated summer sun
        const sunGeometry = new THREE.SphereGeometry(65, 32, 32);
        const sunMaterial = new THREE.MeshBasicMaterial({
            color: 0xffd166,
        });

        sun = new THREE.Mesh(sunGeometry, sunMaterial);
        sun.position.set(-850, FLOOR + 650, -300);
        scene.add(sun);

        new THREE.TextureLoader().load(`${base}/3d/sky.jpg`, (texture) => {
            texture.repeat.set(0.8, 1);
            scene.background = texture;
        });

        // Ambient Light
        const ambient = new THREE.AmbientLight(0xffffff);
        scene.add(ambient);

        // Directional Light
        const light = new THREE.DirectionalLight(0xffffff, 3);
        light.position.set(0, 1500, 1000);
        light.castShadow = true;
        Object.assign(light.shadow.camera, {
            top: 2000,
            bottom: -2000,
            left: -2000,
            right: 2000,
            near: 1200,
            far: 2500,
        });
        light.shadow.bias = 0.0001;
        light.shadow.mapSize.width = 2048;
        light.shadow.mapSize.height = 1024;
        scene.add(light);

        // Ground
        addGround(scene, FLOOR, `${base}/3d/grasslight-big.jpg`);

        // Title + Stacked Bars
        const fontLoader = new FontLoader();
        fontLoader.load(`${base}/3d/helvetiker_bold.typeface.json`, (font: Font) => {
            // Title
            const textGeo = new TextGeometry("summer movies", {
                font: font,
                size: 30,
                depth: 10,
            });
            textGeo.computeBoundingBox();
            const centerOffset =
                -0.5 *
                (textGeo!.boundingBox!.max.x - textGeo!.boundingBox!.min.x);
            const textMaterial = new THREE.MeshStandardMaterial({
                color: 0x449900,
                roughness: 0.5,
            });
            const titleMesh = new THREE.Mesh(textGeo, textMaterial);
            titleMesh.position.x = centerOffset;
            titleMesh.position.y = FLOOR + 620;
            titleMesh.castShadow = true;
            scene.add(titleMesh);

            createUnits(scene, font);
            createLegend(scene, font);
        });

        // === TODO-2: Add Expressive 3D Objects ===

        // Load bird models (linear motion)
        const birdModels = [
            {
                path: `${base}/3d/Flamingo.glb`,
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: 100,
                scale: 0.5,
            },
            {
                path: `${base}/3d/Flamingo.glb`,
                speed: 350,
                duration: 1,
                x: 300 - Math.random() * 500,
                y: FLOOR + 550,
                z: -100,
                scale: 0.5,
            },
            {
                path: `${base}/3d/Parrot.glb`,
                speed: 350,
                duration: 0.5,
                x: 500 - Math.random() * 500,
                y: FLOOR + 500,
                z: 700,
                scale: 0.5,
            },
        ];
        mixer = loadModels(birdModels, scene, mixer, morphs);

        // Load horse models (orbit around data)
        horseMixer = new THREE.AnimationMixer(scene);
        const gltfLoader = new GLTFLoader();
        const horseConfigs = [
            { startAngle: 0, orbitSpeed: 0.4 },
            { startAngle: Math.PI, orbitSpeed: 0.4 },
        ];
        horseConfigs.forEach((config) => {
            gltfLoader.load(`${base}/3d/Horse.glb`, (gltf) => {
                const mesh = gltf.scene.children[0].clone() as THREE.Mesh;
                (mesh as any).material = (mesh as any).material.clone();
                mesh.scale.set(0.5, 0.5, 0.5);
                mesh.castShadow = true;

                if (gltf.animations.length > 0) {
                    horseMixer
                        .clipAction(gltf.animations[0], mesh)
                        .setDuration(1)
                        .startAt(-Math.random())
                        .play();
                }

                mesh.userData.angle = config.startAngle;
                mesh.userData.orbitSpeed = config.orbitSpeed;
                mesh.position.set(
                    ORBIT_RX * Math.cos(config.startAngle),
                    FLOOR,
                    ORBIT_RZ * Math.sin(config.startAngle),
                );

                scene.add(mesh);
                horseMorphs.push(mesh);
            });
        });
        // === END TODO-2 ===

        // Handle resize
        window.addEventListener("resize", () =>
            onWindowResize(
                camera,
                renderer,
                window.innerWidth,
                window.innerHeight,
            ),
        );

        renderer.setAnimationLoop(animate);
    }

    // === TODO-3: Unit Visualization ===

    // === TODO-3a: Unique textured pole geometry per genre ===

    function createGenreGeometry(genre: string): THREE.BufferGeometry {
        switch (genre) {
            case "Comedy":
                // Round pole
                return new THREE.CylinderGeometry(2.5, 2.5, 5, 12);

            case "Romance":
                // Rounded, slightly tapered pole
                return new THREE.CylinderGeometry(3, 2, 5, 8);

            case "Drama":
                // Triangular pole
                return new THREE.CylinderGeometry(2.5, 2.5, 5, 3);

            default:
                return new THREE.CylinderGeometry(2.5, 2.5, 5, 8);
        }
    }

    function createGenreTexture(genre: string): THREE.CanvasTexture {
        const canvas = document.createElement("canvas");
        canvas.width = 128;
        canvas.height = 128;

        const ctx = canvas.getContext("2d")!;

        // Base colors and patterns
        if (genre === "Comedy") {
            ctx.fillStyle = "#e9a15b";
            ctx.fillRect(0, 0, 128, 128);

            // Orange stripes
            ctx.fillStyle = "#fff0c2";
            for (let y = 0; y < 128; y += 24) {
                ctx.fillRect(0, y, 128, 7);
            }
        } else if (genre === "Romance") {
            ctx.fillStyle = "#c95f50";
            ctx.fillRect(0, 0, 128, 128);

            // Subtle heart pattern
            ctx.fillStyle = "#f3a08c";
            ctx.font = "28px Arial";
            ctx.textAlign = "center";

            for (let y = 20; y < 128; y += 42) {
                for (let x = 20; x < 128; x += 42) {
                    ctx.fillText("♥", x, y);
                }
            }
        } else {
            ctx.fillStyle = "#168b80";
            ctx.fillRect(0, 0, 128, 128);

            // Teal diagonal wood-grain pattern
            ctx.strokeStyle = "#83cfc0";
            ctx.lineWidth = 3;

            for (let x = -128; x < 256; x += 18) {
                ctx.beginPath();
                ctx.moveTo(x, 0);
                ctx.lineTo(x + 128, 128);
                ctx.stroke();
            }
        }

        const texture = new THREE.CanvasTexture(canvas);
        texture.colorSpace = THREE.SRGBColorSpace;
        texture.wrapS = THREE.RepeatWrapping;
        texture.wrapT = THREE.RepeatWrapping;

        return texture;
    }

    function createGenreMaterial(genre: string): THREE.Material {
        return new THREE.MeshPhysicalMaterial({
            map: createGenreTexture(genre),
            roughness: 0.45,
            metalness: 0.05,
            clearcoat: 0.4,
        });
    }

    // === END TODO-3a ===

    // TODO-3b: Implement createUnits()

    function createUnits(scene: THREE.Scene, font: Font) {
        if (stackData.length === 0) {
            console.warn("No chart data available.");
            return;
        }

        const barMaxWidth = Math.min(stackData.length * 80, 1400);

        // Increase the vertical contribution of each movie.
        const unitSize = 6;

        // Raise the chart so it is more visible in the scene.
        const chartBaseY = FLOOR + 100;

        const xScale = d3
            .scaleBand()
            .domain(stackData.map((d) => d.year))
            .range([-barMaxWidth / 2, barMaxWidth / 2])
            .padding(0.15);

        const bandwidth = xScale.bandwidth();

        stackData.forEach((row) => {
            const xPos = xScale(row.year)! + bandwidth / 2;
            let currentY = chartBaseY;

            genres.forEach((genre) => {
                const count = row[genre as keyof TStackRow] as number;

                for (let i = 0; i < count; i++) {
                    const geometry = createGenreGeometry(genre);
                    const material = createGenreMaterial(genre);

                    const mesh = new THREE.Mesh(geometry, material);

                    mesh.position.set(
                        xPos,
                        currentY + unitSize / 2,
                        0,
                    );

                    mesh.castShadow = true;
                    mesh.receiveShadow = true;

                    scene.add(mesh);

                    currentY += unitSize;
                }
            });
        });

        // Year labels
        stackData.forEach((row, index) => {
            // Avoid overcrowding labels when there are many years.
            if (index % 2 !== 0 && stackData.length > 15) return;

            const x = xScale(row.year)! + bandwidth / 2;

            const textGeo = new TextGeometry(row.year, {
                font,
                size: 7,
                depth: 2,
            });

            textGeo.computeBoundingBox();

            const textWidth =
                textGeo.boundingBox!.max.x - textGeo.boundingBox!.min.x;

            // YEAR LABEL COLOR: WHITE
            const textMat = new THREE.MeshPhysicalMaterial({
                color: 0xffffff,
            });

            const textMesh = new THREE.Mesh(textGeo, textMat);

            textMesh.position.set(
                x - textWidth / 2,
                FLOOR + 65,
                15
            );

            textMesh.castShadow = true;
            scene.add(textMesh);
        });

        console.log("Chart rendered:", stackData);
    }

    // TODO-3c: Update createLegend() to use genre shapes and materials.

    function createBars(scene: THREE.Scene, font: Font) {
        if (stackData.length === 0) return;

        const barMaxWidth = Math.min(stackData.length * 80, 1400);

        // D3 scales
        const xScale = d3
            .scaleBand()
            .domain(stackData.map((d) => d.year))
            .range([-barMaxWidth / 2, barMaxWidth / 2])
            .padding(0.1);

        const bandwidth = xScale.bandwidth();
        const barWidth = Math.min(bandwidth * 0.7, 25);

        // One colored bar segment per genre, stacked per year
        stackData.forEach((row) => {
            const xPos = xScale(row.year)! + bandwidth / 2;
            let currentY = FLOOR;

            genres.forEach((genre) => {
                const count = row[genre as keyof TStackRow] as number;
                const barHeight = count * 10;
                if (barHeight === 0) return;

                const geometry = new THREE.BoxGeometry(barWidth, barHeight, barWidth);
                const material = new THREE.MeshPhysicalMaterial({ color: genreColors[genre] });
                const mesh = new THREE.Mesh(geometry, material);
                mesh.position.set(xPos, currentY + barHeight / 2, 0);
                mesh.castShadow = true;
                mesh.receiveShadow = true;
                scene.add(mesh);
                currentY += barHeight;
            });
        });

        // Year labels
        stackData.forEach((row) => {
            const x = xScale(row.year)! + bandwidth / 2;
            const textGeo = new TextGeometry(row.year, {
                font: font,
                size: 6,
                depth: 2,
            });
            textGeo.computeBoundingBox();
            const textWidth = textGeo.boundingBox!.max.x - textGeo.boundingBox!.min.x;
            const textMat = new THREE.MeshPhysicalMaterial({ color: 0x000000 });
            const textMesh = new THREE.Mesh(textGeo, textMat);
            textMesh.position.set(x - textWidth / 2, FLOOR + 5, barWidth / 2 + 5);
            textMesh.castShadow = true;
            scene.add(textMesh);
        });
    }

    // Update your new Legend code here — see TODO-3c
    function createLegend(scene: THREE.Scene, font: Font) {
        // MOVED LEGEND TO THE LEFT SIDE
        const legendX = -500;
        const legendStartY = FLOOR + 520;
        const spacing = 50;

        genres.forEach((genre, i) => {
            const y = legendStartY - i * spacing;

            // Use the same geometry and material as the visualization.
            const swatchGeo = createGenreGeometry(genre);
            const swatchMat = createGenreMaterial(genre);

            const swatch = new THREE.Mesh(swatchGeo, swatchMat);

            swatch.position.set(legendX, y, 0);
            swatch.scale.set(1.3, 1.3, 1.3);
            swatch.castShadow = true;

            scene.add(swatch);

            const textGeo = new TextGeometry(genre, {
                font,
                size: 10,
                depth: 3,
            });

            // GENRE LABEL COLOR: DARK GREEN
            const textMat = new THREE.MeshStandardMaterial({
                color: 0x14532d,
                roughness: 0.5,
            });

            const textMesh = new THREE.Mesh(textGeo, textMat);

            textMesh.position.set(legendX + 30, y - 5, 0);
            textMesh.castShadow = true;

            scene.add(textMesh);
        });
    }

    function animate() {
        const delta = clock.getDelta();

        // Sun animation
        if (animating && sun) {
            sun.rotation.y += delta * 0.3;
            sun.position.y = FLOOR + 650 + Math.sin(clock.elapsedTime) * 12;
        }

        if (animating) {
            if (mixer) mixer.update(delta);
            if (horseMixer) horseMixer.update(delta);

            // Birds - linear motion
            morphs.forEach((morph) => {
                morph.position.x += morph.speed * delta;
                if (morph.position.x > window.innerWidth / 2) {
                    morph.position.x = -window.innerWidth / 2 - Math.random() * 200;
                }
            });

            // Horses - orbit around the data
            horseMorphs.forEach((horse) => {
                horse.userData.angle += horse.userData.orbitSpeed * delta;
                const angle = horse.userData.angle;
                horse.position.x = ORBIT_RX * Math.cos(angle);
                horse.position.z = ORBIT_RZ * Math.sin(angle);

                // Face the direction of travel
                const lookAngle = angle + 0.1;
                horse.lookAt(
                    ORBIT_RX * Math.cos(lookAngle),
                    horse.position.y,
                    ORBIT_RZ * Math.sin(lookAngle),
                );
            });
        }

        renderer.render(scene, camera);
    }
</script>

<div bind:this={container} class="container"></div>

<button class="toggle-btn" onclick={() => animating = !animating}>
    {animating ? "Pause" : "Play"} Animation
</button>

<style>
    div.container {
        width: 100%;
        height: 100%;
    }

    .toggle-btn {
        position: fixed;
        bottom: 10px;
        right: 10px;
        padding: 8px 10px;
        font-size: 12px;
        border: none;
        border-radius: 20px;
        background: rgba(0, 0, 0, 0.30);
        color: white;
        cursor: pointer;
        z-index: 10;
    }

    .toggle-btn:hover {
        background: rgba(0, 0, 0, 0.8);
    }
</style>
