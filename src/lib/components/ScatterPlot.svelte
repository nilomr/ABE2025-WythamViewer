<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { TrackballControls } from 'three/examples/jsm/controls/TrackballControls.js';
	import * as d3 from 'd3-dsv';
	import { interpolateSpectral } from 'd3-scale-chromatic';
	import Controls from './Controls.svelte';
	import gsap from 'gsap';

	let container: HTMLDivElement;
	let data: any[] = [];
	let speciesCounts: Record<string, number> = {};

	// UI state
	let selectedSpecies = 'All';
	let selectedSite = 'All';
	let selectedHour: [number, number] = [0, 47];
	let colorMode: 'time' | 'species' | 'site' | '' = '';

	// Options for selectors
	let speciesOptions: string[] = [];
	let siteOptions: string[] = [];
	let hourOptions: number[] = [];

	// For updating colors
	let pointsGeometry: THREE.BufferGeometry | null = null;

	// Select items arrays
	$: speciesSelectItems = [
		{ value: 'All', label: 'All' },
		...speciesOptions.map((s) => ({ value: s, label: s }))
	];
	$: siteSelectItems = [
		{ value: 'All', label: 'All' },
		...siteOptions.map((s) => ({ value: s, label: s }))
	];

	let sceneLoaded = false;

	let timePalette: string[] = [];
	let uniqueTimes: string[] = [];

	// Store camera target for animation
	let cameraTarget = new THREE.Vector3(0, 0, 0);

	function updateTimePalette() {
		uniqueTimes = Array.from(new Set(data.map((d) => d.time))).sort();
		const n = Math.floor(uniqueTimes.length / 2);
		let palette = [];
		for (let i = 0; i <= n; ++i) {
			palette.push(interpolateSpectral(i / n));
		}
		palette = palette.reverse();
		timePalette = palette.concat(palette.slice().reverse());
	}

	function createCircleTexture(size = 64) {
		const canvas = document.createElement('canvas');
		canvas.width = canvas.height = size;
		const ctx = canvas.getContext('2d')!;
		ctx.clearRect(0, 0, size, size);
		ctx.beginPath();
		ctx.arc(size / 2, size / 2, size / 2, 0, 2 * Math.PI, false);
		ctx.closePath();
		ctx.fillStyle = '#fff';
		ctx.globalAlpha = 1;
		ctx.fill();
		return new THREE.CanvasTexture(canvas);
	}

	async function fetchData() {
		const response = await fetch('/2025-dataset.csv');
		const csvData = await response.text();
		data = d3
			.csvParse(csvData, (d: any, i: number) => {
				if (!d.umap_x || !d.umap_y) return null;
				const umap_x = Number(d.umap_x);
				const umap_y = Number(d.umap_y);
				const umap_z = d.umap_z !== undefined && d.umap_z !== '' ? Number(d.umap_z) : 0;
				const confidence =
					d.confidence !== undefined && d.confidence !== '' ? parseFloat(d.confidence) : null;
				if (!isFinite(umap_x) || !isFinite(umap_y) || !isFinite(umap_z)) return null;
				let hour = 0;
				let halfHour = 0;
				if (d.time) {
					const match = /^(\d{1,2}):(\d{2})/.exec(d.time);
					if (match) {
						hour = parseInt(match[1], 10);
						const minute = parseInt(match[2], 10);
						if (isNaN(hour) || hour < 0 || hour > 23) hour = 0;
						halfHour = hour * 2 + (minute >= 30 ? 1 : 0);
					}
				}
				return {
					...d,
					umap_x,
					umap_y,
					umap_z,
					confidence, // Add confidence here
					hour,
					halfHour
				};
			})
			.filter(Boolean);

		speciesCounts = {};
		data.forEach((d) => {
			const name = d.common_name || 'No detection';
			speciesCounts[name] = (speciesCounts[name] ?? 0) + 1;
		});

		speciesOptions = Array.from(Object.entries(speciesCounts))
			.filter(([_, count]) => count > 2)
			.map(([name]) => name)
			.sort();

		siteOptions = Array.from(new Set(data.map((d) => d.site || 'No detection'))).sort();

		updateHourOptions();
		selectedSpecies = 'All';
		selectedSite = 'All';
	}

	function updateHourOptions() {
		const oldMin = selectedHour[0];
		const oldMax = selectedHour[1];

		hourOptions = Array.from(new Set(data.map((d) => d.halfHour))).sort((a, b) => a - b);

		updateTimePalette();

		const newMinPossible = hourOptions[0] ?? 0;
		const newMaxPossible = hourOptions[hourOptions.length - 1] ?? 47;

		let newSelectedMin = Math.max(newMinPossible, Math.min(oldMin, newMaxPossible));
		let newSelectedMax = Math.max(newMinPossible, Math.min(oldMax, newMaxPossible));

		if (newSelectedMin > newSelectedMax) {
			newSelectedMin = newMinPossible;
			newSelectedMax = newMaxPossible;
		}

		selectedHour = [newSelectedMin, newSelectedMax];
	}

	function updatePointColors() {
		if (!pointsGeometry || !data || data.length === 0) return;

		let colorAttribute = pointsGeometry.attributes.color as THREE.BufferAttribute | undefined;

		if (!colorAttribute || colorAttribute.array.length !== data.length * 3) {
			const newColorsArray = new Float32Array(data.length * 3);
			pointsGeometry.setAttribute('color', new THREE.Float32BufferAttribute(newColorsArray, 3));
			colorAttribute = pointsGeometry.attributes.color as THREE.BufferAttribute;
		}

		const colorsArray = colorAttribute.array as Float32Array;

		const [minH, maxH] = selectedHour && selectedHour.length === 2 ? selectedHour : [null, null];

		const timeToIdx = Object.fromEntries(uniqueTimes.map((t, i) => [t, i]));
		const paletteLen = timePalette.length;

		const speciesList = ['All', ...speciesOptions];
		const siteList = ['All', ...siteOptions];
		const speciesPalette = speciesList.map((_, i) =>
			interpolateSpectral(i / Math.max(1, speciesList.length - 1))
		);
		const sitePalette = siteList.map((_, i) =>
			interpolateSpectral(i / Math.max(1, siteList.length - 1))
		);

		data.forEach((point, i) => {
			let isActive = true;

			if (selectedSpecies !== 'All' && point.common_name !== selectedSpecies) {
				isActive = false;
			}

			if (isActive && selectedSite !== 'All' && point.site !== selectedSite) {
				isActive = false;
			}

			if (isActive && minH !== null && maxH !== null) {
				const pointHourValue = point.halfHour;
				if (pointHourValue < minH || pointHourValue > maxH) {
					isActive = false;
				}
			}

			let colorInstance: THREE.Color;
			if (isActive) {
				if (colorMode === 'time') {
					const idx = timeToIdx[point.time] ?? 0;
					const paletteIdx =
						paletteLen > 1 && uniqueTimes.length > 1
							? Math.round((idx * (paletteLen - 1)) / (uniqueTimes.length - 1))
							: 0;
					const hex = timePalette[paletteIdx] || '#e4ebed';
					colorInstance = new THREE.Color(hex);
					const hsl = { h: 0, s: 0, l: 0 };
					colorInstance.getHSL(hsl);
					colorInstance.setHSL(
						hsl.h,
						Math.min(1, hsl.s * 1.25 + 0.1),
						Math.min(1, hsl.l * 1.1 + 0.05)
					);
				} else if (colorMode === 'species') {
					if (!point.common_name || point.common_name === 'Unknown') {
						colorInstance = new THREE.Color(0x1c2426);
					} else {
						const idx = speciesList.indexOf(point.common_name ?? 'All');
						const hex = speciesPalette[idx >= 0 ? idx : 0] || '#e4ebed';
						colorInstance = new THREE.Color(hex);
					}
				} else if (colorMode === 'site') {
					const idx = siteList.indexOf(point.site ?? 'All');
					const hex = sitePalette[idx >= 0 ? idx : 0] || '#e4ebed';
					colorInstance = new THREE.Color(hex);
				} else {
					colorInstance = new THREE.Color('#e4ebed');
				}
			} else {
				colorInstance = new THREE.Color(0x1c2426);
			}

			colorsArray[i * 3] = colorInstance.r;
			colorsArray[i * 3 + 1] = colorInstance.g;
			colorsArray[i * 3 + 2] = colorInstance.b;
		});

		pointsGeometry.attributes.color.needsUpdate = true;
	}

	let tooltip = {
		visible: false,
		x: 0,
		y: 0,
		species: '',
		site: '',
		time: '',
		confidence: null as number | null // Add confidence to tooltip state
	};

	function getRelativeMousePos(event: MouseEvent, element: HTMLElement) {
		const rect = element.getBoundingClientRect();
		return {
			x: event.clientX - rect.left,
			y: event.clientY - rect.top
		};
	}

	let raycaster = new THREE.Raycaster();
	let mouse = new THREE.Vector2();

	function onPointerMove(event: MouseEvent) {
		if (!container || !pointsGeometry) return;
		const canvas = container.querySelector('canvas');
		if (!canvas) return;

		const rect = canvas.getBoundingClientRect();
		mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
		mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

		if (!scene || !camera || !points) return;
		raycaster.setFromCamera(mouse, camera);
		const intersects = raycaster.intersectObject(points);

		if (intersects.length > 0) {
			const idx = intersects[0].index;
			if (idx != null && data[idx]) {
				tooltip.visible = true;
				tooltip.x = event.clientX;
				tooltip.y = event.clientY;
				tooltip.species = data[idx].common_name ?? '';
				tooltip.site = data[idx].site ?? '';
				tooltip.time = data[idx].time ?? '';
				tooltip.confidence = data[idx].confidence ?? null; // Populate confidence
			} else {
				tooltip.visible = false;
			}
		} else {
			tooltip.visible = false;
		}
	}

	function onPointerLeave() {
		tooltip.visible = false;
	}

	let scene: THREE.Scene;
	let camera: THREE.PerspectiveCamera;
	let points: THREE.Points;
	let controls: TrackballControls | null = null; // Make controls module-level

	// Helper: get median center and bounding sphere for densest 95% of points of a species
	function getSpeciesMedianCenterAndBoundingSphere(species: string) {
		const filtered = data.filter((d) => d.common_name === species);
		if (filtered.length === 0) return null;

		// Compute median for each axis
		const xs = filtered.map((d) => d.umap_x).sort((a, b) => a - b);
		const ys = filtered.map((d) => d.umap_y).sort((a, b) => a - b);
		const zs = filtered.map((d) => d.umap_z).sort((a, b) => a - b);
		const mid = Math.floor(filtered.length / 2);
		const median = new THREE.Vector3(
			xs.length % 2 ? xs[mid] : (xs[mid - 1] + xs[mid]) / 2,
			ys.length % 2 ? ys[mid] : (ys[mid - 1] + ys[mid]) / 2,
			zs.length % 2 ? zs[mid] : (zs[mid - 1] + zs[mid]) / 2
		);

		// Find 95% of points closest to the median
		const pointsWithDist = filtered.map((d) => {
			const dx = d.umap_x - median.x;
			const dy = d.umap_y - median.y;
			const dz = d.umap_z - median.z;
			return {
				d,
				dist: Math.sqrt(dx * dx + dy * dy + dz * dz)
			};
		});
		pointsWithDist.sort((a, b) => a.dist - b.dist);
		const keepCount = Math.max(1, Math.floor(filtered.length * 0.95));
		const kept = pointsWithDist.slice(0, keepCount).map((p) => p.d);

		// Compute "center of mass" of these kept points
		let sumX = 0,
			sumY = 0,
			sumZ = 0;
		for (const d of kept) {
			sumX += d.umap_x;
			sumY += d.umap_y;
			sumZ += d.umap_z;
		}
		const numKept = kept.length > 0 ? kept.length : 1;
		const center = new THREE.Vector3(sumX / numKept, sumY / numKept, sumZ / numKept);

		// Compute bounding sphere radius for kept points
		let maxR = 0;
		for (const d of kept) {
			const r = Math.sqrt(
				Math.pow(d.umap_x - center.x, 2) +
					Math.pow(d.umap_y - center.y, 2) +
					Math.pow(d.umap_z - center.z, 2)
			);
			if (r > maxR) maxR = r;
		}
		return { center, radius: maxR };
	}

	// Helper: compute camera distance to fit bounding sphere
	function getCameraDistanceToFitSphere(
		radius: number,
		camera: THREE.PerspectiveCamera,
		fitRatio = 1.2
	): number {
		const fov = camera.fov * (Math.PI / 180);
		const aspect = camera.aspect;
		const fitRadius = radius * fitRatio;

		if (fitRadius < 0.01) return 0.5;

		const verticalDist = fitRadius / Math.sin(fov / 2);
		const horizontalFov = 2 * Math.atan(Math.tan(fov / 2) * aspect);
		const horizontalDist = fitRadius / Math.sin(horizontalFov / 2);

		return Math.max(verticalDist, horizontalDist);
	}

	// Animate camera to center/radius of densest 95% of selected species
	function flyToSpecies(species: string) {
		if (!camera || !controls || !scene || !data || data.length === 0) return;

		gsap.killTweensOf(camera.position);
		gsap.killTweensOf(cameraTarget);

		let newCamPos: THREE.Vector3;
		let newTarget: THREE.Vector3;
		const duration = 1.5;
		const ease = 'power2.inOut';

		if (species === 'All') {
			newCamPos = new THREE.Vector3(-6, 3, -10);
			newTarget = new THREE.Vector3(0, 0, 0);
		} else {
			const result = getSpeciesMedianCenterAndBoundingSphere(species);
			if (!result) return;
			const { center, radius } = result;

			const distance = getCameraDistanceToFitSphere(radius, camera, 1.25);

			const viewOffsetDirection = new THREE.Vector3(0.4, 0.3, 0.8).normalize();

			newCamPos = center.clone().add(viewOffsetDirection.multiplyScalar(distance));
			newTarget = center.clone();
		}

		const controlsWereEnabled = controls.enabled;
		controls.enabled = false;

		gsap.to(camera.position, {
			x: newCamPos.x,
			y: newCamPos.y,
			z: newCamPos.z,
			duration,
			ease
		});

		gsap.to(cameraTarget, {
			x: newTarget.x,
			y: newTarget.y,
			z: newTarget.z,
			duration,
			ease,
			onComplete: () => {
				if (controls) {
					controls.target.copy(newTarget);
					if (controlsWereEnabled) {
						controls.enabled = true;
					}
				}
			}
		});
	}

	$: if (sceneLoaded && selectedSpecies) {
		flyToSpecies(selectedSpecies);
	}

	function createScene() {
		if (!container || data.length === 0) return;

		scene = new THREE.Scene();
		scene.background = new THREE.Color(0x2c3336);

		const validData = data.filter(
			(d) => isFinite(d.umap_x) && isFinite(d.umap_y) && isFinite(d.umap_z)
		);
		if (validData.length === 0) {
			container.innerHTML = '<p style="color:red">No valid data points to display.</p>';
			return;
		}

		camera = new THREE.PerspectiveCamera(
			75,
			container.clientWidth / container.clientHeight,
			0.1,
			1000
		);
		camera.position.set(0, 0, 3);

		const renderer = new THREE.WebGLRenderer({ antialias: true });
		renderer.setSize(container.clientWidth, container.clientHeight);
		container.appendChild(renderer.domElement);

		raycaster.params.Points.threshold = 0.02;

		controls = new TrackballControls(camera, renderer.domElement);
		controls.noPan = true;
		controls.noZoom = false;
		controls.dynamicDampingFactor = 0.1;

		pointsGeometry = new THREE.BufferGeometry();
		const positions: number[] = [];
		data.forEach((point) => {
			positions.push(point.umap_x, point.umap_y, point.umap_z);
		});
		pointsGeometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));

		updatePointColors();

		const circleTexture = createCircleTexture();

		const pointsMaterial = new THREE.PointsMaterial({
			size: 0.045,
			vertexColors: true,
			transparent: true,
			opacity: 0.7,
			map: circleTexture,
			alphaTest: 0.4,
			sizeAttenuation: true
		});
		points = new THREE.Points(pointsGeometry, pointsMaterial);
		scene.add(points);

		let revealing = true;
		let revealProgress = 0;
		let startPos = new THREE.Vector3(0, 20, 40);
		let endPos = new THREE.Vector3(-6, 3, -10);

		setTimeout(() => {
			sceneLoaded = true;
		}, 1000);

		gsap.to(
			{ t: 0 },
			{
				t: 1,
				duration: 4,
				ease: 'power2.out',
				onUpdate: function () {
					revealProgress = this.targets()[0].t;
				},
				onComplete: function () {
					revealing = false;
					if (controls) {
						controls.enabled = true;
						controls.target.copy(cameraTarget);
					}
				}
			}
		);

		if (controls) controls.enabled = false;

		function animate() {
			requestAnimationFrame(animate);

			if (revealing) {
				camera.position.lerpVectors(startPos, endPos, revealProgress);
				camera.lookAt(cameraTarget);
				if (controls) controls.target.copy(cameraTarget);
			} else {
				camera.lookAt(cameraTarget);
			}

			if (controls && controls.enabled) {
				controls.update();
			}
			renderer.render(scene, camera);
		}

		animate();

		window.addEventListener('resize', () => {
			camera.aspect = container.clientWidth / container.clientHeight;
			camera.updateProjectionMatrix();
			renderer.setSize(container.clientWidth, container.clientHeight);
			controls.handleResize();
		});

		const canvas = renderer.domElement;
		canvas.addEventListener('pointermove', onPointerMove);
		canvas.addEventListener('pointerleave', onPointerLeave);
	}

	function resetAll() {
		selectedSpecies = 'All';
		selectedSite = 'All';
		const minTime = hourOptions[0] ?? 0;
		const maxTime = hourOptions[hourOptions.length - 1] ?? 47;
		selectedHour = [minTime, maxTime];
		colorMode = '';
	}

	$: (selectedSpecies, selectedSite, selectedHour, data, pointsGeometry, colorMode),
		updatePointColors();

	onMount(async () => {
		await fetchData();
		createScene();
	});
</script>

<div bind:this={container} class="scatter-container"></div>

<Controls
	bind:selectedSpecies
	bind:selectedSite
	bind:selectedHour
	{speciesSelectItems}
	{siteSelectItems}
	{hourOptions}
	{speciesCounts}
	{sceneLoaded}
	bind:colorMode
	onReset={resetAll}
	{tooltip}
/>

<style>
	.scatter-container {
		width: 100%;
		height: 100%;
		position: relative;
		overflow: visible;
	}
</style>
