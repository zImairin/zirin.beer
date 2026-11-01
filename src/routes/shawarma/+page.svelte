<script lang="ts">
	import { Canvas } from '@threlte/core';
	import SceneOne from '../../lib/scene/SceneOne.svelte';
</script>

<svelte:head>
	<title>Шаурма 3D</title>
</svelte:head>

<main>
	<section class="canvas-wrapper ring h-[700px]" aria-label="Вращающаяся модель шаурмы">
		<Canvas>
			<SceneOne />
		</Canvas>
	</section>
</main>
