<script lang="ts">
	import { onMount } from 'svelte';

	type Cursor = { x: number; y: number; vx: number; vy: number };
	let canvas = $state<HTMLCanvasElement>();
	let cursors: Cursor[] = [];
	let frame = 0;

	function drawCursor(context: CanvasRenderingContext2D, x: number, y: number) {
		context.save();
		context.translate(Math.round(x), Math.round(y));
		context.beginPath();
		context.moveTo(0, 0);
		context.lineTo(1, 16);
		context.lineTo(5, 12);
		context.lineTo(8, 18);
		context.lineTo(10, 17);
		context.lineTo(7, 11);
		context.lineTo(13, 10);
		context.closePath();
		context.fillStyle = '#fff';
		context.fill();
		context.strokeStyle = '#171717';
		context.lineWidth = 1;
		context.stroke();
		context.restore();
	}

	function animate() {
		if (!canvas) return;
		const context = canvas.getContext('2d');
		if (!context) return;
		const ratio = Math.min(window.devicePixelRatio || 1, 2);
		const bounds = canvas.getBoundingClientRect();
		canvas.width = Math.round(bounds.width * ratio);
		canvas.height = Math.round(bounds.height * ratio);
		context.setTransform(ratio, 0, 0, ratio, 0, 0);
		const count = Math.min(1200, Math.max(180, Math.round((bounds.width * bounds.height) / 520)));
		if (cursors.length !== count) {
			cursors = Array.from({ length: count }, () => ({
				x: Math.random() * bounds.width,
				y: Math.random() * bounds.height,
				vx: (Math.random() - 0.5) * 2.5,
				vy: (Math.random() - 0.5) * 2.5
			}));
		}
		context.clearRect(0, 0, bounds.width, bounds.height);
		for (const cursor of cursors) {
			if (Math.random() < 0.04) {
				cursor.vx += (Math.random() - 0.5) * 2.5;
				cursor.vy += (Math.random() - 0.5) * 2.5;
				cursor.vx = Math.max(-3.5, Math.min(3.5, cursor.vx));
				cursor.vy = Math.max(-3.5, Math.min(3.5, cursor.vy));
			}
			cursor.x += cursor.vx;
			cursor.y += cursor.vy;
			if (cursor.x < 0 || cursor.x > bounds.width - 14) cursor.vx *= -1;
			if (cursor.y < 0 || cursor.y > bounds.height - 19) cursor.vy *= -1;
			cursor.x = Math.max(0, Math.min(bounds.width - 14, cursor.x));
			cursor.y = Math.max(0, Math.min(bounds.height - 19, cursor.y));
			drawCursor(context, cursor.x, cursor.y);
		}
		frame = window.requestAnimationFrame(animate);
	}

	onMount(() => {
		frame = window.requestAnimationFrame(animate);
		return () => window.cancelAnimationFrame(frame);
	});
</script>

<svelte:head>
	<title>sosite — нажми сюда</title>
</svelte:head>

<main>
	<canvas bind:this={canvas} class="cursor-field" aria-hidden="true"></canvas>
	<div class="browser-bar">
		<div class="tab">● &nbsp; Новая вкладка <span>×</span></div>
		<div class="address">‹ &nbsp; http://sosite.ru/нажми-сюда &nbsp; ↻</div>
	</div>
	<a class="click-me" href="/shawarma">Click me</a>
	<div class="corner-note">Лучше нажать мышкой</div>
</main>

<style>
	:global(*) { box-sizing: border-box; }
	:global(body) { margin: 0; background: #fff; font-family: Arial, Helvetica, sans-serif; }
	main { position: relative; min-height: 100dvh; overflow: hidden; background: #fff; }
	.cursor-field { position: absolute; inset: 54px 0 0; width: 100%; height: calc(100dvh - 54px); pointer-events: none; }
	.browser-bar { position: relative; z-index: 2; height: 54px; background: #080808; color: #ddd; font-size: 10px; }
	.tab { display: flex; width: 190px; height: 23px; align-items: center; padding: 0 9px; background: #191919; }
	.tab span { margin-left: auto; }
	.address { height: 31px; padding: 8px; background: #111; color: #aaa; }
	.click-me { position: absolute; z-index: 3; top: 48%; left: 50%; padding: 3px 6px; border: 1px solid #9f00ae; background: #d000cf; color: #fff; font-size: 15px; text-decoration: none; transform: translate(-50%, -50%); }
	.click-me:hover { background: #f000e9; }
	.corner-note { position: absolute; right: 4px; bottom: 3px; z-index: 2; color: #999; font-size: 8px; }
</style>
