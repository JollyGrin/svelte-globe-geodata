<script lang="ts">
	import * as THREE from 'three';
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
	import { onMount, onDestroy } from 'svelte';
	import { basePath } from '$lib/helpers';

	// Define types for better type checking
	type GeoJSONFeature = {
		type: 'Feature';
		geometry: {
			type: 'Polygon' | 'MultiPolygon';
			coordinates: (number[][] | number[][][])[];
		};
		properties?: any;
	};

	type GeoJSON = {
		type: 'FeatureCollection';
		features: GeoJSONFeature[];
	};

	const sphereRadius = 2;

	let container: HTMLDivElement;
	let scene: THREE.Scene | null = null;
	let camera: THREE.PerspectiveCamera | null = null;
	let renderer: THREE.WebGLRenderer | null = null;
	let controls: OrbitControls | null = null;
	let animationFrameId: number | null = null;
	let mounted = false;

	let selectedObject: THREE.Object3D | null = null;

	function initScene() {
		if (!mounted || !container) return;

		const w = window.innerWidth;
		const h = window.innerHeight;

		scene = new THREE.Scene();
		scene.fog = new THREE.FogExp2(0x000000, 0.3);

		camera = new THREE.PerspectiveCamera(75, w / h, 1, 100);
		camera.position.z = 5;

		renderer = new THREE.WebGLRenderer({ antialias: true });
		renderer.setSize(w, h);
		container.appendChild(renderer.domElement);

		controls = new OrbitControls(camera, renderer.domElement);

		controls!.maxDistance = 5;
		controls!.minDistance = 3;
		controls.enableDamping = true;

		const geometry = new THREE.SphereGeometry(sphereRadius);
		const lineMat = new THREE.LineBasicMaterial({
			color: 0xffffff,
			transparent: true,
			opacity: 0.1
		});

		const edges = new THREE.EdgesGeometry(geometry, 1);
		const line = new THREE.LineSegments(edges, lineMat);
		scene.add(line);

		loadGeoJSON();
		animate();
	}

	function convertToSphereCoords(lat: number, lon: number, radius: number): THREE.Vector3 {
		const phi = (90 - lat) * (Math.PI / 180);
		const theta = (lon + 180) * (Math.PI / 180);

		return new THREE.Vector3(
			-radius * Math.sin(phi) * Math.cos(theta),
			radius * Math.cos(phi),
			radius * Math.sin(phi) * Math.sin(theta)
		);
	}

	function mapPolygonRing(
		ring: number[][],
		countryMaterial: THREE.LineBasicMaterial,
		properties: any
	) {
		const points = ring.map((coord) => convertToSphereCoords(coord[1], coord[0], sphereRadius));
		const geometry = new THREE.BufferGeometry().setFromPoints(points);
		const line = new THREE.Line(geometry, countryMaterial);
		line.userData = properties;
		return line;
	}

	function fillPolygonRing(
		ring: number[][],
		fillMaterial: THREE.MeshBasicMaterial,
		properties: any
	) {
		// fill
	}

	async function loadGeoJSON() {
		try {
			const response = await fetch(basePath + '/geo.json');
			const data: GeoJSON = await response.json();
			console.log({ data });

			const countryMaterial = new THREE.LineBasicMaterial({
				color: 0x00ff00,
				transparent: false,
				opacity: 1
			});

			const fillMaterial = new THREE.MeshBasicMaterial({
				color: 0xff0000, // Different color for filling
				side: THREE.DoubleSide // Ensure visibility from both sides
			});

			data.features.forEach((feature: GeoJSONFeature) => {
				if (feature.geometry.type === 'Polygon')
					feature.geometry.coordinates.forEach((ring) => {
						//wireframe
						scene?.add(mapPolygonRing(ring as number[][], countryMaterial, feature.properties));
						//fill
						// scene?.add(fillPolygonRing(ring as number[][], fillMaterial, feature.properties));
					});

				if (feature.geometry.type === 'MultiPolygon')
					feature.geometry.coordinates.forEach((polygon) =>
						polygon.forEach((ring) =>
							scene?.add(mapPolygonRing(ring as number[][], countryMaterial, feature.properties))
						)
					);
			});
		} catch (error) {
			console.error('Error loading GeoJSON:', error);
		}
	}

	function animate() {
		if (!mounted) return;
		animationFrameId = requestAnimationFrame(animate);
		renderer?.render(scene!, camera!);
		controls?.update();
	}

	function handleWindowResize() {
		if (camera && renderer) {
			camera.aspect = window.innerWidth / window.innerHeight;
			camera.updateProjectionMatrix();
			renderer.setSize(window.innerWidth, window.innerHeight);
		}
	}

	onMount(() => {
		// Check if we're in a browser environment
		if (typeof window !== 'undefined') {
			mounted = true;
			initScene();
			window.addEventListener('resize', handleWindowResize);
		}
	});

	onDestroy(() => {
		mounted = false;

		if (animationFrameId) {
			cancelAnimationFrame(animationFrameId);
		}

		if (renderer) {
			renderer.dispose();
		}

		if (controls) {
			controls.dispose();
		}

		if (typeof window !== 'undefined') {
			window.removeEventListener('resize', handleWindowResize);
		}
	});
</script>

<div class="h-[2rem]">
	<p>
		<!-- {selectedObject?.userData.admin} -->
	</p>
</div>
<div bind:this={container} class="globe-container"></div>

<style>
	.globe-container {
		width: 100vw;
		height: calc(100vh - 2rem);
	}
</style>
