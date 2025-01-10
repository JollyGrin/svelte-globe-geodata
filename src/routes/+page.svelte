<script>
	import * as THREE from 'three';
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

	import { onMount, onDestroy } from 'svelte';
	import { drawThreeGeo } from '$lib/threeGeo';

	let container;
	let scene;
	let camera;
	let renderer;
	let controls;
	let animationFrameId;
	let mounted = false;

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
		controls.enableDamping = true;

		const geometry = new THREE.SphereGeometry(2);
		const lineMat = new THREE.LineBasicMaterial({
			color: 0xffffff,
			transparent: true,
			opacity: 0.4
		});

		const edges = new THREE.EdgesGeometry(geometry, 1);
		const line = new THREE.LineSegments(edges, lineMat);
		scene.add(line);

		loadGeoJSON();
		animate();
	}

	function convertToSphereCoords(lat, lon, radius) {
		const phi = (90 - lat) * (Math.PI / 180);
		const theta = (lon + 180) * (Math.PI / 180);

		return new THREE.Vector3(
			-radius * Math.sin(phi) * Math.cos(theta),
			radius * Math.cos(phi),
			radius * Math.sin(phi) * Math.sin(theta)
		);
	}

	async function loadGeoJSON() {
		try {
			const response = await fetch('/geo.json');
			const data = await response.json();
			console.log({ data });

			const radius = 2;
			const countryMaterial = new THREE.LineBasicMaterial({
				color: 0x00ff00,
				transparent: true,
				opacity: 0.6
			});

			data.features.forEach((feature) => {
				if (feature.geometry.type === 'Polygon') {
					feature.geometry.coordinates.forEach((ring) => {
						const points = [];
						ring.forEach((coord) => {
							points.push(convertToSphereCoords(coord[1], coord[0], radius));
						});
						const geometry = new THREE.BufferGeometry().setFromPoints(points);
						const line = new THREE.Line(geometry, countryMaterial);
						scene.add(line);
					});
				} else if (feature.geometry.type === 'MultiPolygon') {
					feature.geometry.coordinates.forEach((polygon) => {
						polygon.forEach((ring) => {
							const points = [];
							ring.forEach((coord) => {
								points.push(convertToSphereCoords(coord[1], coord[0], radius));
							});
							const geometry = new THREE.BufferGeometry().setFromPoints(points);
							const line = new THREE.Line(geometry, countryMaterial);
							scene.add(line);
						});
					});
				}
			});
		} catch (error) {
			console.error('Error loading GeoJSON:', error);
		}
	}

	function animate() {
		if (!mounted) return;
		animationFrameId = requestAnimationFrame(animate);
		renderer.render(scene, camera);
		controls.update();
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

<div bind:this={container} class="globe-container"></div>

<style>
	.globe-container {
		width: 100vw;
		height: 100vh;
	}
</style>
