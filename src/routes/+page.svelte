<script lang="ts">
	import { base } from '$app/paths';
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { OBJLoader } from 'three/addons/loaders/OBJLoader.js';

	const targetFrameTime = 1000 / 60; //60fps

	class ThreeScene {
		scene: THREE.Scene;
		camera: THREE.PerspectiveCamera;
		renderer: THREE.WebGLRenderer;
		arrows: Arrows | null = null;

		constructor() {
			this.handleWindowResize = this.handleWindowResize.bind(this);
			this.selectElement = this.selectElement.bind(this);

			this.scene = new THREE.Scene();
			this.scene.fog = new THREE.Fog(0x000000, 1, 1500);

			this.camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
			this.camera.position.z = 15;

			this.renderer = new THREE.WebGLRenderer({
				antialias: true
			});
			this.renderer.setSize(window.innerWidth, window.innerHeight);
			this.renderer.setClearColor(0x000000, 1);
		}

		handleWindowResize(): void {
			this.camera.aspect = window.innerWidth / window.innerHeight;
			this.camera.updateProjectionMatrix();

			this.renderer.setSize(window.innerWidth, window.innerHeight);

			this.selectElement(document.getElementById('selected'));
		}

		//position the arrows next to element being hovered
		selectElement(element: HTMLElement | null): void {
			const currentSelection = document.getElementById('selected');
			if (currentSelection) {
				currentSelection.removeAttribute('id');
			}
			if (element && this.arrows) {
				element.id = 'selected';

				const boundingBox = element.getBoundingClientRect();
				let position = this.screenToWorld(boundingBox.left, (boundingBox.top + boundingBox.bottom) / 2);
				this.arrows.arrow1.position.x = position.x;
				this.arrows.arrow1.position.y = position.y;
				this.arrows.arrow1.position.z = 0;

				position = this.screenToWorld(boundingBox.right, (boundingBox.top + boundingBox.bottom) / 2);
				this.arrows.arrow2.position.x = position.x;
				this.arrows.arrow2.position.y = position.y;
				this.arrows.arrow2.position.z = 0;
			}
		}

		//utility function to convert screen coordinates to world equivalent
		screenToWorld(x: number, y: number): THREE.Vector3 {
			let vector = new THREE.Vector3();
			vector.set((x / window.innerWidth) * 2 - 1, -(y / window.innerHeight) * 2 + 1, 0);
			vector.unproject(this.camera);
			let direction = vector.sub(this.camera.position).normalize();
			let distance = -this.camera.position.z / direction.z;
			let position = this.camera.position.clone().add(direction.multiplyScalar(distance));
			return position;
		}

		render = (): void => {
			this.renderer.render(this.scene, this.camera);
		};
	}

	class InfinitePointCloud {
		private particleCount = 10000;
		private particles1: THREE.Points;
		private particles2: THREE.Points;

		constructor() {
			const pGeometry1 = new THREE.BufferGeometry();
			const pGeometry2 = new THREE.BufferGeometry();
			const vertices1 = [];
			const vertices2 = [];
			for (let i = 0; i < this.particleCount * 3; i++) {
				vertices1.push(Math.random() * 2000 - 1000);
				vertices2.push(Math.random() * 2000 - 1000);
			}

			pGeometry1.setAttribute('position', new THREE.Float32BufferAttribute(vertices1, 3));
			pGeometry2.setAttribute('position', new THREE.Float32BufferAttribute(vertices2, 3));

			const pMaterial = new THREE.PointsMaterial({
				color: 0xffffff,
				size: 5,
				map: new THREE.TextureLoader().load(`${base}/textures/star.png`),
				blending: THREE.AdditiveBlending,
				transparent: true
			});

			this.particles1 = new THREE.Points(pGeometry1, pMaterial);
			this.particles2 = new THREE.Points(pGeometry2, pMaterial);
			this.particles1.position.z = 15;
			//places the second point cloud behind the first one
			this.particles2.position.z = this.particles1.position.z - 2000;
		}

		animate(timeDelta: number): void {
			//move the stars, and wrap the point clouds once they are no longer on screen
			const timeMultiplier = timeDelta / targetFrameTime;
			this.particles1.position.z += 10 * timeMultiplier;
			this.particles2.position.z += 10 * timeMultiplier;
			if (this.particles1.position.z > 1015) {
				this.particles1.position.z = -3015;
			}
			if (this.particles2.position.z > 1015) {
				this.particles2.position.z = -3015;
			}
		}

		addToScene(scene: ThreeScene): void {
			scene.scene.add(this.particles1);
			scene.scene.add(this.particles2);
		}
	}

	class Arrows {
		arrow1: THREE.Object3D;
		arrow2: THREE.Object3D;

		private constructor(arrow1: THREE.Object3D, arrow2: THREE.Object3D) {
			this.arrow1 = arrow1;
			this.arrow2 = arrow2;
		}

		static async load(): Promise<Arrows> {
			const arrow1 = await new OBJLoader().loadAsync(`${base}/models/arrow.obj`);
			convertToWireFrame(arrow1);
			//hide arrows
			arrow1.position.z = -2000;

			//rotate the arrows to face the correct direction
			arrow1.rotation.x = Math.PI / 2;
			arrow1.rotation.y = Math.PI / 2;

			const arrow2 = arrow1.clone();
			// arrow2.rotation.x = Math.PI / 2;
			arrow2.rotation.y = (3 * Math.PI) / 2;

			return new Arrows(arrow1, arrow2);
		}

		addToScene(scene: ThreeScene): void {
			scene.scene.add(this.arrow1);
			scene.scene.add(this.arrow2);
			scene.arrows = this;
		}

		animate(timeDelta: number): void {
			const timeMultiplier = timeDelta / targetFrameTime;
			this.arrow1.rotation.z += 0.1 * timeMultiplier;
			this.arrow2.rotation.z -= 0.1 * timeMultiplier;
		}
	}

	class Logo {
		private logo: THREE.Object3D;
		private counter = 0;

		private constructor(logo: THREE.Object3D) {
			this.logo = logo;
		}

		static async load(): Promise<Logo> {
			const logo = await new OBJLoader().loadAsync(`${base}/models/leoland.obj`);
			convertToWireFrame(logo);
			return new Logo(logo);
		}

		addToScene(scene: ThreeScene): void {
			this.logo.position.y =
				Math.tan(((scene.camera.fov / 2) * Math.PI) / 180) * scene.camera.position.z * (1 - 2 * (9 / 32));
			scene.scene.add(this.logo);
		}

		animate(timeDelta: number): void {
			this.logo.rotation.y = Math.sin(this.counter) / 4;
			this.logo.rotation.z = Math.cos(this.counter) / 4;
			this.counter += 0.05 * (timeDelta / targetFrameTime);
		}
	}

	function convertToWireFrame(object: THREE.Object3D): void {
		object.traverse(function (child) {
			if (child instanceof THREE.Mesh) {
				child.material = new THREE.MeshStandardMaterial({
					transparent: true,
					opacity: 0.6
				});

				const edges = new THREE.EdgesGeometry(child.geometry);
				const lineMaterial = new THREE.LineBasicMaterial({ color: 0x00c6ff, linewidth: 2 });
				const wireframe = new THREE.LineSegments(edges, lineMaterial);
				wireframe.matrix = child.matrix;

				child.add(wireframe);
			}
		});
	}

	let threeScene: ThreeScene;
	let container: HTMLDivElement;
	onMount(async () => {
		threeScene = new ThreeScene();

		const stars = new InfinitePointCloud();
		stars.addToScene(threeScene);

		const leoland = await Logo.load();
		leoland.addToScene(threeScene);

		const arrows = await Arrows.load();
		arrows.addToScene(threeScene);

		container.appendChild(threeScene.renderer.domElement);

		let lastTime = performance.now();
		let timeDelta = 0;

		let frame = requestAnimationFrame(function loop() {
			const currentTime = performance.now();
			if (!document.hidden) {
				timeDelta = currentTime - lastTime;

				stars.animate(timeDelta);
				leoland.animate(timeDelta);
				arrows.animate(timeDelta);

				threeScene.render();
			}

			lastTime = currentTime;
			frame = requestAnimationFrame(loop);
		});

		return () => cancelAnimationFrame(frame);
	});
</script>

<svelte:window on:resize={threeScene.handleWindowResize} />
<svelte:head>
	<title>Welcome to leoland!</title>
</svelte:head>

<div id="menu">
	<nav>
		<a
			href={base}
			on:mouseover={(event) => threeScene.selectElement(event.target)}
			on:focus={(event) => threeScene.selectElement(event.target)}>COMING</a
		><br />
		<a
			href={base}
			on:mouseover={(event) => threeScene.selectElement(event.target)}
			on:focus={(event) => threeScene.selectElement(event.target)}>SOON</a
		><br />
	</nav>
</div>
<div bind:this={container} />

<style>
	@import url('https://fonts.cdnfonts.com/css/ds-digital');

	#menu {
		position: absolute;
		top: 55%;
		width: 100%;
	}
	#menu nav {
		font-size: 4em;
		text-align: center;
	}
	#menu nav a {
		display: inline-block;
		font-family: DS-Digital, sans-serif;
		font-weight: bold;
		font-style: italic;
		padding: 0 0.5em 0 0.5em;
		text-decoration: none;
		color: rgba(0, 198, 255, 0.6);
	}
</style>
