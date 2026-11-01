<script lang="ts">
	import { onMount } from 'svelte';

	type Cursor = { x: number; y: number; vx: number; vy: number };
	type TrailPoint = { x: number; y: number };
	const map = [
		'S..#.....#',
		'#..#.###.#',
		'#..#.....#',
		'##.#####.#',
		'#........#',
		'#.######.#',
		'########.G'
	];
	const columns = map[0].length;
	const rows = map.length;
	let screen = $state<'cursors' | 'maze'>('cursors');
	let cursorCanvas = $state<HTMLCanvasElement>();
	let mazeCanvas = $state<HTMLCanvasElement>();
	let cursors: Cursor[] = [];
	let cursorFrame = 0;
	let mazeFrame = 0;
	let mazePointerCell: number | null = null;
	let mazePointerPosition: TrailPoint | null = null;
	let mazeStarted = false;
	let mazeCompleted = false;
	let outsideRouteActive = false;
	let trailEdges = new Set<string>();
	let outsideTrail: TrailPoint[] = [];

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

	function animateCursors() {
		if (!cursorCanvas || screen !== 'cursors') return;
		const context = cursorCanvas.getContext('2d');
		if (!context) return;
		const bounds = cursorCanvas.getBoundingClientRect();
		const ratio = Math.min(window.devicePixelRatio || 1, 2);
		cursorCanvas.width = Math.round(bounds.width * ratio);
		cursorCanvas.height = Math.round(bounds.height * ratio);
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
				cursor.vx = Math.max(-3.5, Math.min(3.5, cursor.vx + (Math.random() - 0.5) * 2.5));
				cursor.vy = Math.max(-3.5, Math.min(3.5, cursor.vy + (Math.random() - 0.5) * 2.5));
			}
			cursor.x += cursor.vx;
			cursor.y += cursor.vy;
			if (cursor.x < 0 || cursor.x > bounds.width - 14) cursor.vx *= -1;
			if (cursor.y < 0 || cursor.y > bounds.height - 19) cursor.vy *= -1;
			cursor.x = Math.max(0, Math.min(bounds.width - 14, cursor.x));
			cursor.y = Math.max(0, Math.min(bounds.height - 19, cursor.y));
			drawCursor(context, cursor.x, cursor.y);
		}
		cursorFrame = window.requestAnimationFrame(animateCursors);
	}

	function mazeLayout() {
		if (!mazeCanvas) return null;
		const bounds = mazeCanvas.getBoundingClientRect();
		const ratio = Math.min(window.devicePixelRatio || 1, 2);
		mazeCanvas.width = Math.round(bounds.width * ratio);
		mazeCanvas.height = Math.round(bounds.height * ratio);
		const context = mazeCanvas.getContext('2d');
		if (!context) return null;
		context.setTransform(ratio, 0, 0, ratio, 0, 0);
		const cellWidth = bounds.width / (columns + 2);
		const cellHeight = bounds.height / (rows + 2);
		return { context, bounds, cellWidth, cellHeight, left: cellWidth, top: cellHeight };
	}

	function drawMaze() {
		const layout = mazeLayout();
		if (!layout) return;
		const { context, bounds, cellWidth, cellHeight, left, top } = layout;
		context.fillStyle = '#fff';
		context.fillRect(0, 0, bounds.width, bounds.height);
		for (let row = 0; row < rows; row += 1) {
			for (let column = 0; column < columns; column += 1) {
				if (map[row][column] === '#') {
					context.fillStyle = '#171717';
					context.fillRect(left + column * cellWidth, top + row * cellHeight, cellWidth, cellHeight);
				}
			}
		}
		if (trailEdges.size) {
			context.beginPath();
			context.strokeStyle = '#81d9ed';
			context.lineWidth = Math.max(2, Math.min(cellWidth, cellHeight) * 0.28);
			context.lineCap = 'round';
			for (const edge of trailEdges) {
				const [from, to] = edge.split(':').map(Number);
				context.moveTo(left + ((from % columns) + 0.5) * cellWidth, top + (Math.floor(from / columns) + 0.5) * cellHeight);
				context.lineTo(left + ((to % columns) + 0.5) * cellWidth, top + (Math.floor(to / columns) + 0.5) * cellHeight);
			}
			context.stroke();
		}
		if (outsideTrail.length > 1) {
			context.beginPath();
			context.strokeStyle = '#81d9ed';
			context.lineWidth = Math.max(2, Math.min(cellWidth, cellHeight) * 0.28);
			context.lineCap = 'round';
			context.moveTo(outsideTrail[0].x * bounds.width, outsideTrail[0].y * bounds.height);
			for (const point of outsideTrail.slice(1)) context.lineTo(point.x * bounds.width, point.y * bounds.height);
			context.stroke();
		}
		context.fillStyle = '#ffc928';
		context.fillRect(left + cellWidth * 0.32, top + cellHeight * 0.32, cellWidth * 0.36, cellHeight * 0.36);
		context.fillStyle = '#27d927';
		context.fillRect(left + (columns - 1) * cellWidth + cellWidth * 0.2, top + (rows - 1) * cellHeight + cellHeight * 0.2, cellWidth * 0.6, cellHeight * 0.6);
	}

	function handleMazeMove(event: PointerEvent) {
		if (!mazeCanvas || mazeCompleted) return;
		const layout = mazeLayout();
		if (!layout) return;
		const { bounds, cellWidth, cellHeight, left, top } = layout;
		const point = { x: event.clientX - bounds.left, y: event.clientY - bounds.top };
		const gridBottom = top + rows * cellHeight;
		const cellAt = (x: number, y: number) => {
			const column = Math.floor((x - left) / cellWidth);
			const row = Math.floor((y - top) / cellHeight);
			return column < 0 || row < 0 || column >= columns || row >= rows ? -1 : row * columns + column;
		};
		const current = cellAt(point.x, point.y);
		if (!mazeStarted) {
			if (current !== 0) return;
			mazeStarted = true;
			mazePointerCell = 0;
			mazePointerPosition = point;
			return;
		}
		if (!mazePointerPosition) {
			mazePointerPosition = point;
			mazePointerCell = current >= 0 && map[Math.floor(current / columns)][current % columns] !== '#' ? current : null;
			return;
		}
		const start = mazePointerPosition;
		const steps = Math.max(1, Math.ceil(Math.max(Math.abs(point.x - start.x), Math.abs(point.y - start.y)) / (Math.min(cellWidth, cellHeight) * 0.28)));
		let previous = start;
		for (let step = 1; step <= steps; step += 1) {
			const amount = step / steps;
			const sample = { x: start.x + (point.x - start.x) * amount, y: start.y + (point.y - start.y) * amount };
			const next = cellAt(sample.x, sample.y);
			if (outsideRouteActive) {
				if (next < 0) {
					outsideTrail.push({ x: sample.x / bounds.width, y: sample.y / bounds.height });
				} else if (next === map.length * columns - 1 && previous.y >= gridBottom && sample.y < gridBottom) {
					outsideTrail.push({ x: sample.x / bounds.width, y: sample.y / bounds.height });
					mazeCompleted = true;
					outsideRouteActive = false;
				} else {
					outsideRouteActive = false;
				}
			} else if (next < 0) {
				if (mazePointerCell === 0 && previous.y >= top && sample.y < top && sample.x <= left + cellWidth) {
					outsideRouteActive = true;
					outsideTrail.push({ x: previous.x / bounds.width, y: previous.y / bounds.height });
					outsideTrail.push({ x: sample.x / bounds.width, y: sample.y / bounds.height });
				}
			} else {
				const row = Math.floor(next / columns);
				const column = next % columns;
				if (map[row][column] !== '#') {
					if (mazePointerCell === null) mazePointerCell = next;
					else if (next !== mazePointerCell) {
						const fromRow = Math.floor(mazePointerCell / columns);
						const fromColumn = mazePointerCell % columns;
						if (Math.abs(fromRow - row) + Math.abs(fromColumn - column) === 1) {
							const edge = mazePointerCell < next ? `${mazePointerCell}:${next}` : `${next}:${mazePointerCell}`;
							trailEdges.add(edge);
							mazePointerCell = next;
							if (map[row][column] === 'G') mazeCompleted = true;
						}
					}
				}
			}
			previous = sample;
		}
		mazePointerPosition = point;
		mazeFrame = window.requestAnimationFrame(drawMaze);
	}

	function enterMaze() {
		window.cancelAnimationFrame(cursorFrame);
		screen = 'maze';
		mazePointerCell = null;
		mazePointerPosition = null;
		mazeStarted = false;
		mazeCompleted = false;
		outsideRouteActive = false;
		trailEdges.clear();
		outsideTrail = [];
		mazeFrame = window.requestAnimationFrame(drawMaze);
	}

	function handlePointerLeave() {
		mazePointerPosition = null;
		mazePointerCell = null;
		outsideRouteActive = false;
	}

	onMount(() => {
		cursorFrame = window.requestAnimationFrame(animateCursors);
		const resize = () => {
			if (screen === 'maze') drawMaze();
		};
		window.addEventListener('resize', resize);
		return () => {
			window.cancelAnimationFrame(cursorFrame);
			window.cancelAnimationFrame(mazeFrame);
			window.removeEventListener('resize', resize);
		};
	});
</script>

<svelte:head>
	<title>sosite — нажми сюда</title>
</svelte:head>

{#if screen === 'cursors'}
	<main class="cursor-screen">
		<canvas bind:this={cursorCanvas} class="cursor-canvas" aria-hidden="true"></canvas>
		<div class="browser-bar"><div class="tab">● &nbsp; Новая вкладка <span>×</span></div><div class="address">‹ &nbsp; http://sosite.ru/нажми-сюда &nbsp; ↻</div></div>
		<button class="click-me" type="button" onclick={enterMaze}>Click me</button>
		<div class="corner-note">Лучше нажать мышкой</div>
	</main>
{:else}
	<main class="maze-screen">
		<div class="maze-toolbar"><span>ПРОВЕРКА ПОДКЛЮЧЕНИЯ</span><span>ЗАГРУЗКА... НЕ ВЫКЛЮЧАЙТЕ</span></div>
		<canvas
			bind:this={mazeCanvas}
			class="maze-canvas"
			aria-label="Лабиринт от оранжевого входа до зелёного выхода"
			onpointermove={handleMazeMove}
			onpointerleave={handlePointerLeave}
		></canvas>
		{#if mazeCompleted}
			<div class="maze-done"><span>ЗАГРУЗКА ЗАВЕРШЕНА</span><a href="/shawarma">Продолжить</a></div>
		{/if}
		<div class="maze-footer">Пожалуйста, подождите. Пожалуйста, подождите. Пожалуйста, подождите.</div>
	</main>
{/if}

<style>
	:global(*) { box-sizing: border-box; }
	:global(body) { margin: 0; background: #fff; color: #111; font-family: Arial, Helvetica, sans-serif; }
	.cursor-screen, .maze-screen { position: relative; min-height: 100dvh; width: 100%; overflow: hidden; }
	.cursor-screen { background: #fff; }
	.cursor-canvas { position: absolute; top: 54px; left: 0; width: 100vw; height: calc(100dvh - 54px); pointer-events: none; }
	.browser-bar { position: relative; z-index: 3; height: 54px; background: #080808; color: #ddd; font: 10px Arial, sans-serif; }
	.tab { display: flex; width: 190px; height: 23px; align-items: center; padding: 0 9px; background: #191919; }
	.tab span { margin-left: auto; }
	.address { height: 31px; padding: 8px; background: #111; color: #aaa; }
	.click-me { position: absolute; z-index: 5; top: 48%; left: 50%; padding: 3px 6px; border: 1px solid #9f00ae; border-radius: 0; background: #d000cf; color: #fff; font-size: 15px; cursor: pointer; transform: translate(-50%, -50%); }
	.maze-screen { display: flex; height: 100dvh; flex-direction: column; background: #fff; }
	.maze-toolbar { display: flex; justify-content: space-between; gap: 12px; padding: 5px 8px; background: #101010; color: #eee; font: 9px 'Courier New', monospace; }
	.maze-canvas { display: block; width: 100%; min-height: 0; flex: 1; border: 1px solid #262626; background: #fff; touch-action: none; }
	.maze-footer { min-height: 20px; padding: 4px 8px; overflow: hidden; background: #ececec; color: #555; font: 9px 'Courier New', monospace; white-space: nowrap; }
	.maze-done { position: fixed; z-index: 4; bottom: 36px; left: 50%; display: flex; align-items: center; gap: 12px; padding: 10px; background: #111; color: #fff; font: bold 11px 'Courier New', monospace; transform: translateX(-50%); }
	.maze-done a { color: #ffdc31; }
	.corner-note { position: absolute; right: 4px; bottom: 3px; z-index: 2; color: #999; font-size: 8px; }
</style>
