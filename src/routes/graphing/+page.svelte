<script lang="ts">
	import * as THREE from 'three';
	import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
	import { ParametricGeometry } from 'three/addons/geometries/ParametricGeometry.js';
	import { onMount } from 'svelte';
	import { Parser, type Expression } from 'expr-eval';

	// Bound elements
	let canvas: HTMLCanvasElement;

	// User input
	let expr = 'sin(sqrt(x^2  + y^2))';
	let domain: Domain = {
		x: {
			start: -10,
			end: 10
		},
		y: {
			start: -10,
			end: 10
		}
	};
	let resolution = 45;
	let showGrid = false;

	interface Domain {
		x: {
			start: number;
			end: number;
		};
		y: {
			start: number;
			end: number;
		};
	}

	// Parse expr
	const parser = new Parser();
	let validExpr = true;
	let parsedExpr: Expression = parser.parse(expr);
	$: {
		try {
			const newExpr = parser.parse(expr);
			const newExprVars = newExpr.variables();

			if (
				(newExprVars.length === 2 && newExprVars.includes('x') && newExprVars.includes('y')) ||
				(newExprVars.length === 1 && (newExprVars.includes('x') || newExprVars.includes('y'))) ||
				newExprVars.length === 0
			) {
				parsedExpr = newExpr;
				validExpr = true;
			} else {
				validExpr = false;
			}
		} catch {
			validExpr = false;
		}
	}

	// Three vars
	const scene = new THREE.Scene();
	const camera = new THREE.PerspectiveCamera(
		50,
		0, // this gets set by resizeCanvasToDisplaySize()
		0.1,
		2000
	);
	camera.position.z = 19;
	camera.position.y = -21;

	let surface: THREE.Mesh;
	$: {
		scene.remove(surface);
		surface = createSurfaceMesh(parsedExpr, resolution, domain);
		scene.add(surface);
	}

	let xGrid: THREE.GridHelper;
	$: {
		scene.remove(xGrid);

		if (showGrid) {
			const rangeX = Math.abs(domain.x.end - domain.x.start);

			const gridSize = Math.ceil(Math.max(rangeX)) + 2;
			xGrid = new THREE.GridHelper(gridSize, gridSize);

			const boundBox = new THREE.Box3().setFromObject(surface);
			boundBox.getCenter(xGrid.position);
			xGrid.position.setComponent(1, domain.y.end + 1);

			scene.add(xGrid);
		}
	}

	let yGrid: THREE.GridHelper;
	$: {
		scene.remove(yGrid);

		if (showGrid) {
			const rangeY = Math.abs(domain.y.end - domain.y.start);

			const gridSize = Math.ceil(Math.max(rangeY)) + 2;
			yGrid = new THREE.GridHelper(gridSize, gridSize);

			const boundBox = new THREE.Box3().setFromObject(surface);
			yGrid.rotateZ(Math.PI / 2);
			boundBox.getCenter(yGrid.position);
			yGrid.position.setComponent(0, domain.x.start - 1);

			scene.add(yGrid);
		}
	}

	function createSurfaceMesh(exp: Expression, resolution: number, domain: Domain) {
		const geometry = new ParametricGeometry(getParametricExpr(exp, domain), resolution, resolution);
		const material = new THREE.MeshNormalMaterial();
		material.side = THREE.DoubleSide;
		const surface = new THREE.Mesh(geometry, material);

		return surface;
	}

	// Three uses u and v parameters from 0 to 1.
	// We need to scale this to the x-y coord system.
	function getParametricExpr(
		exp: Expression,
		domain: Domain
	): (u: number, v: number, target: THREE.Vector3) => void {
		return function parametricExprHelper(u: number, v: number, outVec: THREE.Vector3): void {
			const rangeX = Math.abs(domain.x.end - domain.x.start);
			const rangeY = Math.abs(domain.y.end - domain.y.start);

			const x = rangeX * u + domain.x.start;
			const y = rangeY * v + domain.y.start;
			const z = exp.evaluate({ x, y });

			outVec.set(x, y, z);
		};
	}

	onMount(() => {
		const controls = new OrbitControls(camera, canvas);
		const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
		renderer.setPixelRatio(devicePixelRatio);
		renderer.setClearColor(0xffffff);

		function resizeCanvasToDisplaySize() {
			const canvas = renderer.domElement;
			// look up the size the canvas is being displayed
			const width = canvas.clientWidth;
			const height = canvas.clientHeight;

			// adjust displayBuffer size to match
			if (canvas.width !== width || canvas.height !== height) {
				// you must pass false here or three.js sadly fights the browser
				renderer.setSize(width, height, false);
				camera.aspect = width / height;
				camera.updateProjectionMatrix();

				// update any render target sizes here
			}
		}

		function animate() {
			window.requestAnimationFrame(animate);

			resizeCanvasToDisplaySize();
			controls.update();
			renderer.render(scene, camera);
		}
		window.requestAnimationFrame(animate);
	});
</script>

<div id="container">
	<div id="header">
		<h1>
			Surface Graphing Tool - <a
				href="https://github.com/mymindstorm/website/blob/master/src/routes/graphing/%2Bpage.svelte"
				>Source Code</a
			>
		</h1>
		<div id="controls">
			<div id="expression">
				<div>Expression</div>
				<input bind:value={expr} />
			</div>
			<div id="resolution">
				<div>Resolution</div>
				<input type="range" min="1" max="100" bind:value={resolution} />
			</div>
			<div id="grid">
				<div>Show Grid</div>
				<input type="checkbox" bind:checked={showGrid} />
			</div>
			<div id="domain">
				<div style="margin-bottom: 8px">
					Graph x from
					<input type="number" bind:value={domain.x.start} />
					to
					<input type="number" bind:value={domain.x.end} />
				</div>
				<div>
					Graph y from
					<input type="number" bind:value={domain.y.start} />
					to
					<input type="number" bind:value={domain.y.end} />
				</div>
			</div>
		</div>
	</div>
	<canvas id="three-canvas" bind:this={canvas} />
</div>

<style>
	#header {
		margin-left: 16px;
		margin-bottom: 8px;
	}

	#three-canvas {
		height: 100%;
		width: 100%;
	}

	#container {
		display: flex;
		flex-flow: column;
		height: 100%;
	}

	#controls {
		display: grid;
		grid-template-columns: repeat(auto-fill, 33%);
		grid-auto-rows: min-content;
		gap: 8px;
		grid-auto-flow: row;
	}

	#expression {
		grid-column: span 1;
		grid-row: span 1;
	}

	#resolution {
		grid-column: span 1;
		grid-row: span 1;
	}

	#grid {
		grid-column: span 1;
		grid-row: span 1;
	}
	#domain {
		grid-column: span 3;
		grid-row: span 1;
	}
</style>
