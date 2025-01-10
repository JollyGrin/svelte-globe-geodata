<script lang="ts">
	import * as THREE from 'three';
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
	import { onMount, onDestroy } from 'svelte';

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

	let container: HTMLDivElement;
	let scene: THREE.Scene | null = null;
	let camera: THREE.PerspectiveCamera | null = null;
	let renderer: THREE.WebGLRenderer | null = null;
	let controls: OrbitControls | null = null;
	let animationFrameId: number | null = null;
	let mounted = false;

	// INTERACTIVITY
	let raycaster = new THREE.Raycaster();
	let mouse = new THREE.Vector2();
	let selectedObject: THREE.Object3D | null = null;

	// Add these to your existing state
	let hoveredObject: THREE.Object3D | null = null;
	let previousMaterial: THREE.Material | null = null;

	function onMouseMove(event: MouseEvent) {
		if (!camera || !scene) return;

		// Calculate mouse position in normalized device coordinates
		mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
		mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

		raycaster.setFromCamera(mouse, camera);

		// Check if the mouse is over any line representing a country
		const intersects = raycaster.intersectObjects(scene.children, true);

		if (intersects.length > 0) {
			if (hoveredObject !== intersects[0].object) {
				if (hoveredObject) {
					if (previousMaterial) hoveredObject.material = previousMaterial;
				}
				hoveredObject = intersects[0].object;
				previousMaterial = hoveredObject.material;
				hoveredObject.material = new THREE.LineBasicMaterial({
					color: 0xff0000, // Red for hover effect
					transparent: false,
					opacity: 1
				});
			}
		} else {
			if (hoveredObject) {
				hoveredObject.material = previousMaterial!;
				hoveredObject = null;
			}
		}
	}

	function onMouseClick(event: MouseEvent) {
		if (!camera || !scene || !controls) return;

		mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
		mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

		raycaster.setFromCamera(mouse, camera);
		const intersects = raycaster.intersectObjects(scene.children, true);
		if (intersects.length > 0) {
			selectedObject = intersects[0].object;
			// Zoom into the selected country
			const targetPosition = selectedObject.position.clone();
			const distance = camera!.position.distanceTo(targetPosition);
			const newPosition = targetPosition
				.clone()
				.add(targetPosition.clone().normalize().multiplyScalar(distance));

			console.log({ selectedObject });

			// // Animate camera movement
			// new Tween(camera!.position)
			// 	.to(newPosition, 1000)
			// 	.easing(TWEEN.Easing.Quadratic.Out)
			// 	.start();
			// new TWEEN.Tween(controls!.target)
			// 	.to(targetPosition, 1000)
			// 	.easing(TWEEN.Easing.Quadratic.Out)
			// 	.onUpdate(() => {
			// 		controls!.update();
			// 	})
			// 	.start();

			// Here you would typically update OrbitControls or add more animation if needed
		}
	}

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
		container.addEventListener('mousemove', onMouseMove, false);
		container.addEventListener('click', onMouseClick, false);

		controls = new OrbitControls(camera, renderer.domElement);

		controls!.maxDistance = 5;
		controls!.minDistance = 3;
		controls.enableDamping = true;

		const geometry = new THREE.SphereGeometry(2);
		const lineMat = new THREE.LineBasicMaterial({
			color: 0xffffff,
			transparent: true,
			opacity: 0.2
		});

		const edges = new THREE.EdgesGeometry(geometry, 1);
		const line = new THREE.LineSegments(edges, lineMat);
		// scene.add(line);

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

	async function loadGeoJSON() {
		try {
			const response = await fetch('/geo.json');
			const data: GeoJSON = await response.json();
			console.log({ data });

			const radius = 2;
			const countryMaterial = new THREE.LineBasicMaterial({
				color: 0x00ff00,
				transparent: false,
				opacity: 1
			});

			data.features.forEach((feature: GeoJSONFeature) => {
				if (feature.geometry.type === 'Polygon') {
					//@ts-expect-error: type array mess
					feature.geometry.coordinates.forEach((ring: number[][]) => {
						const points = ring.map((coord) => convertToSphereCoords(coord[1], coord[0], radius));
						const geometry = new THREE.BufferGeometry().setFromPoints(points);
						const line = new THREE.Line(geometry, countryMaterial);
						line.userData = feature.properties;
						scene?.add(line);
					});
				} else if (feature.geometry.type === 'MultiPolygon') {
					//@ts-expect-error: type array mess
					feature.geometry.coordinates.forEach((polygon: number[][][]) => {
						polygon.forEach((ring: number[][]) => {
							const points = ring.map((coord) => convertToSphereCoords(coord[1], coord[0], radius));
							const geometry = new THREE.BufferGeometry().setFromPoints(points);
							const line = new THREE.Line(geometry, countryMaterial);
							line.userData = feature.properties;
							scene?.add(line);
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
		renderer?.render(scene!, camera!);
		controls?.update();
		console.log(controls?.getDistance());
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
		{selectedObject?.userData.admin}
	</p>
</div>
<div bind:this={container} class="globe-container"></div>

<style>
	.globe-container {
		width: 100vw;
		height: calc(100vh - 2rem);
	}
</style>
