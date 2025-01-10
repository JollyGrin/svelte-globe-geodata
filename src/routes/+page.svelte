<script lang="ts">
	import * as THREE from 'three';
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
	import { onMount, onDestroy } from 'svelte';
	import { basePath } from '$lib/helpers';
	import Delaunator from 'delaunator';

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
		console.log({ ring });
		const points = ring.map((coord) => convertToSphereCoords(coord[1], coord[0], sphereRadius));
		const geometry = new THREE.BufferGeometry().setFromPoints(points);
		const line = new THREE.Line(geometry, countryMaterial);
		line.userData = properties;
		return line;
	}

	function addIntermediatePoints(ring: number[][], numPointsToAdd: number = 5) {
		const newRing: number[][] = [];
		for (let i = 0; i < ring.length; i++) {
			const current = ring[i];
			const next = ring[(i + 1) % ring.length]; // Wrap around for last point

			newRing.push(current);
			for (let j = 1; j <= numPointsToAdd; j++) {
				const t = j / (numPointsToAdd + 1);
				const lat = current[1] * (1 - t) + next[1] * t;
				const lon = current[0] * (1 - t) + next[0] * t;
				newRing.push([lon, lat]);
			}
		}
		return newRing;
	}

	function calculateAngle(p1: number[], p2: number[], p3: number[]): number {
		// Convert to 3D for angle calculation
		const v1 = convertToSphereCoords(p1[1], p1[0], 1);
		const v2 = convertToSphereCoords(p2[1], p2[0], 1);
		const v3 = convertToSphereCoords(p3[1], p3[0], 1);

		// Vectors from p2 to p1 and p2 to p3
		const vec1 = new THREE.Vector3().subVectors(v1, v2).normalize();
		const vec2 = new THREE.Vector3().subVectors(v3, v2).normalize();

		// Angle between vectors
		return Math.acos(vec1.dot(vec2));
	}

	// Helper function to map a value from one range to another
	function map(
		value: number,
		start1: number,
		stop1: number,
		start2: number,
		stop2: number
	): number {
		return start2 + (stop2 - start2) * ((value - start1) / (stop1 - start1));
	}

	function addPointsAdaptivelyWithConcavity(
		ring: number[][],
		minPointsToAdd: number = 3,
		maxPointsToAdd: number = 10
	) {
		const newRing: number[][] = [];
		for (let i = 0; i < ring.length; i++) {
			const current = ring[i];
			const next = ring[(i + 1) % ring.length];
			const prev = ring[(i - 1 + ring.length) % ring.length];

			newRing.push(current);

			// Calculate if the segment is concave
			const isConcave = isSegmentConcave(prev, current, next);
			const pointsToAdd = isConcave
				? maxPointsToAdd
				: Math.floor(
						map(calculateAngle(prev, current, next), 0, Math.PI, minPointsToAdd, maxPointsToAdd)
					);

			for (let j = 1; j <= pointsToAdd; j++) {
				const t = j / (pointsToAdd + 1);
				const lat = current[1] * (1 - t) + next[1] * t;
				const lon = current[0] * (1 - t) + next[0] * t;
				newRing.push([lon, lat]);
			}
		}
		return newRing;
	}

	function isSegmentConcave(p1: number[], p2: number[], p3: number[]): boolean {
		// Convert to 3D for cross product calculation
		const v1 = convertToSphereCoords(p1[1], p1[0], 1);
		const v2 = convertToSphereCoords(p2[1], p2[0], 1);
		const v3 = convertToSphereCoords(p3[1], p3[0], 1);

		// Vectors from p2 to p1 and p2 to p3
		const vec1 = new THREE.Vector3().subVectors(v1, v2);
		const vec2 = new THREE.Vector3().subVectors(v3, v2);

		// Cross product to determine if it's concave (assuming counter-clockwise winding)
		const cross = new THREE.Vector3().crossVectors(vec1, vec2);
		const normal = v2.clone().normalize();
		return cross.dot(normal) < 0;
	}
	function fillPolygonRing(
		ring: number[][],
		fillMaterial: THREE.MeshBasicMaterial,
		properties: any
	) {
		// Convert coordinates to 3D points on the sphere
		// const points = ring.map((coord) => convertToSphereCoords(coord[1], coord[0], sphereRadius));

		let detailedRing = addPointsAdaptivelyWithConcavity(ring as number[][], 100, 200);
		const points = detailedRing.map((coord) =>
			convertToSphereCoords(coord[1], coord[0], sphereRadius)
		);
		// Extract 2D coordinates for Delaunay triangulation (flatten the sphere for triangulation)
		const flatPoints = [];
		for (const point of points) {
			flatPoints.push(point.x, point.z);
		}

		// Use Delaunator for triangulation
		const delaunay = new Delaunator(flatPoints);
		const triangles = delaunay.triangles;

		// Create geometry for the filled polygon
		const geometry = new THREE.BufferGeometry();

		// Define vertices
		const vertices = new Float32Array(triangles.length * 3);
		for (let i = 0; i < triangles.length; i += 3) {
			const p1 = points[triangles[i]];
			const p2 = points[triangles[i + 1]];
			const p3 = points[triangles[i + 2]];

			vertices[i * 3] = p1.x;
			vertices[i * 3 + 1] = p1.y;
			vertices[i * 3 + 2] = p1.z;

			vertices[i * 3 + 3] = p2.x;
			vertices[i * 3 + 4] = p2.y;
			vertices[i * 3 + 5] = p2.z;

			vertices[i * 3 + 6] = p3.x;
			vertices[i * 3 + 7] = p3.y;
			vertices[i * 3 + 8] = p3.z;
		}

		geometry.setAttribute('position', new THREE.BufferAttribute(vertices, 3));

		// Create mesh
		const mesh = new THREE.Mesh(geometry, fillMaterial);
		mesh.userData = properties;

		return mesh;
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
				// color: 0x00ff00,
				side: THREE.DoubleSide, // Ensure visibility from both sides
				opacity: 1
			});

			data.features.forEach((feature: GeoJSONFeature) => {
				if (feature.geometry.type === 'Polygon')
					feature.geometry.coordinates.forEach((ring) => {
						//wireframe
						scene?.add(mapPolygonRing(ring as number[][], countryMaterial, feature.properties));
						//fill
						scene?.add(fillPolygonRing(ring as number[][], fillMaterial, feature.properties));
					});

				if (feature.geometry.type === 'MultiPolygon')
					feature.geometry.coordinates.forEach((polygon) =>
						polygon.forEach((ring) => {
							scene?.add(mapPolygonRing(ring as number[][], countryMaterial, feature.properties));
							//fill
							scene?.add(fillPolygonRing(ring as number[][], fillMaterial, feature.properties));
						})
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
