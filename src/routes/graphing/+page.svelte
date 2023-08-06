<script lang="ts">
	import * as THREE from 'three';
	import { onMount } from 'svelte';

	let canvas: HTMLCanvasElement;

	onMount(() => {
		const scene = new THREE.Scene();
		const camera = new THREE.PerspectiveCamera(
			75,
			window.innerWidth / window.innerHeight,
			0.1,
			1000
		);

		const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
		// renderer.setClearColor(0xffffff)

		const geometry = new THREE.BoxGeometry(1, 1, 1);
		const material = new THREE.MeshNormalMaterial();
		const cube = new THREE.Mesh(geometry, material);
		scene.add(cube);

		camera.position.z = 5;

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

		let timeStamp: DOMHighResTimeStamp;
		function animate(t: DOMHighResTimeStamp) {
			if (!timeStamp) timeStamp = t;

			resizeCanvasToDisplaySize();

			if (t - timeStamp >= 16) {
				cube.rotation.x += 0.01;
				cube.rotation.y += 0.01;

				timeStamp = t;
			}
			requestAnimationFrame(animate);
			renderer.render(scene, camera);
		}
		requestAnimationFrame(animate);
	});
</script>

<div id="container">
	<h1>Three.js Test</h1>
	<canvas id="three-canvas" bind:this={canvas} />
</div>

<style>
	h1 {
		margin-left: 1.5vw;
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
</style>
