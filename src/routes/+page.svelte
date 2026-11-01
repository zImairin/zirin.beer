<script lang="ts">
	import { onMount } from 'svelte';
	import { resolve } from '$app/paths';

	type MazeCell = {
		x: number;
		y: number;
		walls: [boolean, boolean, boolean, boolean];
		visited: boolean;
	};
	type Maze = { cells: MazeCell[]; columns: number; rows: number };
	type Cursor = { x: number; y: number; vx: number; vy: number };
	type Screen = 'cursors' | 'maze' | 'ads';

	const ads = [
		{
			brand: 'Я',
			brandName: 'ЯНДЕКС БРАУЗЕР',
			title: 'НАДОЕЛИ РЕКЛАМЫ?',
			copy: 'Установи Яндекс Браузер. Реклама станет ярче, а вкладок будет больше.',
			action: 'СКАЧАТЬ ЯНДЕКС БРАУЗЕР',
			url: 'https://browser.yandex.ru/',
			className: 'yandex'
		},
		{
			brand: 'M',
			brandName: 'МЕССЕНДЖЕР MAX',
			title: 'ВСЕ УЖЕ В MAX!',
			copy: 'Скачай MAX прямо сейчас. Даже это объявление уже отправило тебе сообщение.',
			action: 'УСТАНОВИТЬ MAX',
			url: 'https://max.ru/',
			className: 'max'
		},
		{
			brand: 'Я',
			brandName: 'ЯНДЕКС БРАУЗЕР',
			title: 'ЭТО ОКНО НЕ ЗАКРЫТЬ',
			copy: 'Пока не установишь Яндекс Браузер. Это быстро. Наверное.',
			action: 'УСТАНОВИТЬ СЕЙЧАС',
			url: 'https://browser.yandex.ru/',
			className: 'yandex'
		},
		{
			brand: 'M',
			brandName: 'МЕССЕНДЖЕР MAX',
			title: 'ВАЖНОЕ СООБЩЕНИЕ',
			copy: 'Твои друзья ждут тебя в MAX. Не заставляй их ждать, скачай приложение.',
			action: 'ПЕРЕЙТИ В MAX',
			url: 'https://max.ru/',
			className: 'max'
		},
		{
			brand: 'Я',
			brandName: 'ОБЯЗАТЕЛЬНОЕ ОБНОВЛЕНИЕ',
			title: 'ВАШ БРАУЗЕР УСТАРЕЛ',
			copy: 'Возможно. Для надёжности скачай Яндекс Браузер ещё раз.',
			action: 'СКАЧАТЬ',
			url: 'https://browser.yandex.ru/',
			className: 'yandex'
		},
		{
			brand: 'M',
			brandName: 'МЕССЕНДЖЕР MAX',
			title: 'MAX УЖЕ ЗАПУЩЕН',
			copy: 'Он ждёт. Ты ждёшь. Осталось нажать большую фиолетовую кнопку.',
			action: 'СКАЧАТЬ MAX',
			url: 'https://max.ru/',
			className: 'max'
		},
		{
			brand: 'Я',
			brandName: 'ПОСЛЕДНЕЕ ПРЕДЛОЖЕНИЕ',
			title: 'НУ КУПИ БРАУЗЕР',
			copy: 'Тут уже всё понятно. Яндекс Браузер. Скачать. Сейчас.',
			action: 'ДА, СКАЧАТЬ',
			url: 'https://browser.yandex.ru/',
			className: 'yandex'
		},
		{
			brand: 'M',
			brandName: 'МЕССЕНДЖЕР MAX',
			title: 'ПОСЛЕДНИЙ ШАНС',
			copy: 'Скачай MAX, пока объявление не открыло ещё одно объявление.',
			action: 'СКАЧАТЬ MAX',
			url: 'https://max.ru/',
			className: 'max'
		}
	];

	const directions = [
		{ dx: 0, dy: -1, wall: 0, opposite: 2 },
		{ dx: 1, dy: 0, wall: 1, opposite: 3 },
		{ dx: 0, dy: 1, wall: 2, opposite: 0 },
		{ dx: -1, dy: 0, wall: 3, opposite: 1 }
	];

	let screen = $state<Screen>('cursors');
	let visibleAds = $state(0);
	let adsClosed = $state(false);
	let exitWindowVisible = $state(false);
	let exitWindowClosed = $state(false);
	let closeMoves = $state(0);
	let closePosition = $state({ x: 0, y: 0 });
	let cursorCanvas = $state<HTMLCanvasElement | undefined>();
	let mazeCanvas = $state<HTMLCanvasElement | undefined>();
	let cursors: Cursor[] = [];
	let maze: Maze | null = null;
	let trailEdges = new Set<string>();
	let mazeOutsideTrail: { x: number; y: number }[] = [];
	let reachedCells = new Set<number>();
	let mazePointerCell: number | null = null;
	let mazeTrailCell: number | null = null;
	let mazeOnTrail = false;
	let mazeOutsideRouteActive = false;
	let mazePointerPosition: { x: number; y: number } | null = null;
	let mazeStarted = false;
	let mazeCompleted = false;
	let cursorFrame = 0;
	let mazeTimer = 0;
	let exitTimer = 0;
	let adTimers: number[] = [];

	function createMaze(columns: number, rows: number): Maze {
		const cells: MazeCell[] = Array.from({ length: columns * rows }, (_, index) => ({
			x: index % columns,
			y: Math.floor(index / columns),
			walls: [true, true, true, true],
			visited: false
		}));
		const stack = [cells[0]];
		cells[0].visited = true;

		while (stack.length) {
			const current = stack[stack.length - 1];
			const shuffledDirections = [...directions];
			for (let index = shuffledDirections.length - 1; index > 0; index -= 1) {
				const swapIndex = Math.floor(Math.random() * (index + 1));
				[shuffledDirections[index], shuffledDirections[swapIndex]] = [
					shuffledDirections[swapIndex],
					shuffledDirections[index]
				];
			}
			let carved = false;

			for (const direction of shuffledDirections) {
				const nextX = current.x + direction.dx;
				const nextY = current.y + direction.dy;
				if (nextX < 0 || nextY < 0 || nextX >= columns || nextY >= rows) continue;
				const next = cells[nextY * columns + nextX];
				if (next.visited) continue;

				current.walls[direction.wall] = false;
				next.walls[direction.opposite] = false;
				next.visited = true;
				stack.push(next);
				carved = true;
				break;
			}

			if (!carved) stack.pop();
		}

		cells[0].walls[0] = false;
		cells[cells.length - 1].walls[2] = false;
		return { cells, columns, rows };
	}

	function drawCursor(context: CanvasRenderingContext2D, x: number, y: number) {
		context.save();
		context.translate(Math.round(x), Math.round(y));
		context.beginPath();
		context.moveTo(0, 0);
		context.lineTo(0.5, 15.5);
		context.lineTo(4.3, 11.7);
		context.lineTo(7.1, 17.4);
		context.lineTo(9.2, 16.4);
		context.lineTo(6.6, 10.5);
		context.lineTo(12.1, 10.2);
		context.closePath();
		context.fillStyle = '#ffffff';
		context.fill();
		context.strokeStyle = '#ffffff';
		context.lineWidth = 2.4;
		context.lineJoin = 'miter';
		context.stroke();
		context.strokeStyle = '#202020';
		context.lineWidth = 0.9;
		context.stroke();
		context.restore();
	}

	function sizeCanvas(canvas: HTMLCanvasElement) {
		const ratio = Math.min(window.devicePixelRatio || 1, 2);
		const bounds = canvas.getBoundingClientRect();
		const width = bounds.width;
		const height = bounds.height;
		const pixelWidth = Math.round(width * ratio);
		const pixelHeight = Math.round(height * ratio);
		if (canvas.width !== pixelWidth) canvas.width = pixelWidth;
		if (canvas.height !== pixelHeight) canvas.height = pixelHeight;
		const context = canvas.getContext('2d');
		if (!context) return null;
		context.setTransform(ratio, 0, 0, ratio, 0, 0);
		return { context, width, height };
	}

	function animateCursors() {
		if (screen !== 'cursors' || !cursorCanvas) return;
		const frame = () => {
			if (screen !== 'cursors' || !cursorCanvas) return;
			const currentSize = sizeCanvas(cursorCanvas);
			if (!currentSize) return;
			const { context, width, height } = currentSize;
			const count = Math.min(1800, Math.max(260, Math.round((width * height) / 440)));
			if (cursors.length !== count) {
				cursors = Array.from({ length: count }, () => ({
					x: Math.random() * width,
					y: Math.random() * height,
					vx: (Math.random() - 0.5) * 2.4,
					vy: (Math.random() - 0.5) * 2.4
				}));
			}
			context.clearRect(0, 0, width, height);
			for (const cursor of cursors) {
				if (Math.random() < 0.035) {
					cursor.vx = Math.max(-3.8, Math.min(3.8, cursor.vx + (Math.random() - 0.5) * 3.2));
					cursor.vy = Math.max(-3.8, Math.min(3.8, cursor.vy + (Math.random() - 0.5) * 3.2));
				}
				cursor.x += cursor.vx;
				cursor.y += cursor.vy;
				if (cursor.x < 0 || cursor.x > width - 13) cursor.vx *= -1;
				if (cursor.y < 0 || cursor.y > height - 18) cursor.vy *= -1;
				cursor.x = Math.max(0, Math.min(width - 13, cursor.x));
				cursor.y = Math.max(0, Math.min(height - 18, cursor.y));
				drawCursor(currentSize.context, cursor.x, cursor.y);
			}
			cursorFrame = window.requestAnimationFrame(frame);
		};
		cursorFrame = window.requestAnimationFrame(frame);
	}

	function drawMaze() {
		if (!mazeCanvas) return;
		const size = sizeCanvas(mazeCanvas);
		if (!size) return;
		const { context, width, height } = size;
		const columns = Math.max(14, Math.ceil(width / 32));
		const rows = Math.max(9, Math.ceil(height / 32));
		if (!maze || maze.columns !== columns || maze.rows !== rows) {
			maze = createMaze(columns, rows);
			trailEdges.clear();
			mazeOutsideTrail = [];
			reachedCells.clear();
			mazePointerCell = null;
			mazeTrailCell = null;
			mazeOnTrail = false;
			mazeOutsideRouteActive = false;
			mazePointerPosition = null;
			mazeStarted = false;
			mazeCompleted = false;
			window.clearTimeout(mazeTimer);
		}
		const currentMaze = maze;
		const cellWidth = width / (currentMaze.columns + 2);
		const cellHeight = height / (currentMaze.rows + 2);
		const gridLeft = cellWidth;
		const gridTop = cellHeight;
		context.fillStyle = '#ffffff';
		context.fillRect(0, 0, width, height);
		context.beginPath();
		context.strokeStyle = '#171717';
		context.lineWidth = Math.max(1.2, Math.min(cellWidth, cellHeight) * 0.17);
		context.lineCap = 'square';
		for (const cell of currentMaze.cells) {
			const x = gridLeft + cell.x * cellWidth;
			const y = gridTop + cell.y * cellHeight;
			if (cell.walls[0]) {
				context.moveTo(x, y);
				context.lineTo(x + cellWidth, y);
			}
			if (cell.walls[1]) {
				context.moveTo(x + cellWidth, y);
				context.lineTo(x + cellWidth, y + cellHeight);
			}
			if (cell.walls[2]) {
				context.moveTo(x, y + cellHeight);
				context.lineTo(x + cellWidth, y + cellHeight);
			}
			if (cell.walls[3]) {
				context.moveTo(x, y);
				context.lineTo(x, y + cellHeight);
			}
		}
		context.stroke();

		if (trailEdges.size > 0) {
			context.beginPath();
			context.strokeStyle = '#81d9ed';
			context.lineWidth = Math.max(2.2, Math.min(cellWidth, cellHeight) * 0.28);
			context.lineJoin = 'round';
			context.lineCap = 'round';
			for (const edge of trailEdges) {
				const [fromIndex, toIndex] = edge.split(':').map(Number);
				const from = currentMaze.cells[fromIndex];
				const to = currentMaze.cells[toIndex];
				context.moveTo(gridLeft + (from.x + 0.5) * cellWidth, gridTop + (from.y + 0.5) * cellHeight);
				context.lineTo(gridLeft + (to.x + 0.5) * cellWidth, gridTop + (to.y + 0.5) * cellHeight);
			}
			context.stroke();
		}
		if (mazeOutsideTrail.length > 1) {
			context.beginPath();
			context.strokeStyle = '#81d9ed';
			context.lineWidth = Math.max(2.2, Math.min(cellWidth, cellHeight) * 0.28);
			context.lineJoin = 'round';
			context.lineCap = 'round';
			context.moveTo(mazeOutsideTrail[0].x * width, mazeOutsideTrail[0].y * height);
			for (const point of mazeOutsideTrail.slice(1)) {
				context.lineTo(point.x * width, point.y * height);
			}
			context.stroke();
		}

		context.fillStyle = '#ffc928';
		context.fillRect(gridLeft + cellWidth * 0.34, gridTop + cellHeight * 0.34, cellWidth * 0.32, cellHeight * 0.32);
		context.fillStyle = '#27f312';
		context.fillRect(gridLeft + (currentMaze.columns - 1) * cellWidth + cellWidth * 0.24, gridTop + (currentMaze.rows - 1) * cellHeight + cellHeight * 0.24, cellWidth * 0.52, cellHeight * 0.52);
	}

	function handleMazePointerMove(event: PointerEvent) {
		if (!mazeCanvas || !maze || screen !== 'maze') return;
		const bounds = mazeCanvas.getBoundingClientRect();
		const point = { x: event.clientX - bounds.left, y: event.clientY - bounds.top };
		const cellWidth = bounds.width / (maze.columns + 2);
		const cellHeight = bounds.height / (maze.rows + 2);
		const gridLeft = cellWidth;
		const gridTop = cellHeight;
		const gridRight = gridLeft + maze.columns * cellWidth;
		const gridBottom = gridTop + maze.rows * cellHeight;
		const cellAt = (x: number, y: number) => {
			const column = Math.floor((x - gridLeft) / cellWidth);
			const row = Math.floor((y - gridTop) / cellHeight);
			return column < 0 || row < 0 || column >= maze!.columns || row >= maze!.rows
				? -1
				: row * maze!.columns + column;
		};
		const inOuterLane = (x: number, y: number) =>
			x >= 0 && x <= bounds.width && y >= 0 && y <= bounds.height &&
				(x <= gridLeft || x >= gridRight || y <= gridTop || y >= gridBottom);
		const currentIndex = cellAt(point.x, point.y);

		if (!mazeStarted) {
			if (currentIndex !== 0) return;
			mazeStarted = true;
			mazePointerCell = 0;
			mazeTrailCell = 0;
			mazeOnTrail = true;
			mazeOutsideRouteActive = false;
			reachedCells.add(0);
			mazePointerPosition = point;
			drawMaze();
			return;
		}

		if (!mazePointerPosition) {
			mazePointerCell = currentIndex >= 0 ? currentIndex : null;
			mazeOnTrail = currentIndex >= 0 && reachedCells.has(currentIndex);
			mazeTrailCell = mazeOnTrail ? currentIndex : null;
			mazePointerPosition = point;
			return;
		}

		const start = mazePointerPosition;
		const distance = Math.max(Math.abs(point.x - start.x), Math.abs(point.y - start.y));
		const steps = Math.max(1, Math.ceil(distance / (Math.min(cellWidth, cellHeight) * 0.3)));
		let trailChanged = false;
		let previousPoint = start;
		const recordOutsidePoint = (position: { x: number; y: number }) => {
			const normalized = { x: position.x / bounds.width, y: position.y / bounds.height };
			const last = mazeOutsideTrail.at(-1);
			if (!last || Math.hypot((normalized.x - last.x) * bounds.width, (normalized.y - last.y) * bounds.height) >= 1) {
				mazeOutsideTrail.push(normalized);
				trailChanged = true;
			}
		};

		for (let step = 1; step <= steps; step += 1) {
			const progress = step / steps;
			const sample = {
				x: start.x + (point.x - start.x) * progress,
				y: start.y + (point.y - start.y) * progress
			};
			const nextIndex = cellAt(sample.x, sample.y);

			if (mazeOutsideRouteActive) {
				if (nextIndex < 0 && inOuterLane(sample.x, sample.y)) {
					recordOutsidePoint(sample);
				} else if (
					nextIndex === maze.cells.length - 1 &&
					previousPoint.y >= gridBottom &&
					sample.y < gridBottom &&
					sample.x >= gridLeft + (maze.columns - 1) * cellWidth &&
					sample.x <= gridRight
				) {
					recordOutsidePoint(sample);
					mazeOutsideRouteActive = false;
					mazeCompleted = true;
					mazeTimer = window.setTimeout(enterAds, 700);
				} else {
					mazeOutsideRouteActive = false;
					mazeOnTrail = nextIndex >= 0 && reachedCells.has(nextIndex);
					mazeTrailCell = mazeOnTrail ? nextIndex : null;
				}
			} else if (nextIndex < 0) {
				const leftStart = mazePointerCell === 0 &&
					sample.y < gridTop &&
					previousPoint.y >= gridTop &&
					sample.x >= gridLeft &&
					sample.x <= gridLeft + cellWidth &&
					inOuterLane(sample.x, sample.y);
				mazeOnTrail = false;
				mazeTrailCell = null;
				if (leftStart) {
					mazeOutsideRouteActive = true;
					recordOutsidePoint(previousPoint);
					recordOutsidePoint(sample);
				}
			} else if (nextIndex !== mazePointerCell) {
				if (mazePointerCell === null) {
					mazeOnTrail = reachedCells.has(nextIndex);
					mazeTrailCell = mazeOnTrail ? nextIndex : null;
				} else {
					const fromIndex = mazePointerCell;
					const from = maze.cells[fromIndex];
					const to = maze.cells[nextIndex];
					const direction = directions.find((item) => from.x + item.dx === to.x && from.y + item.dy === to.y);
					if (direction && !from.walls[direction.wall]) {
						if (mazeOnTrail && mazeTrailCell === fromIndex) {
							const edge = fromIndex < nextIndex ? `${fromIndex}:${nextIndex}` : `${nextIndex}:${fromIndex}`;
							if (!trailEdges.has(edge)) {
								trailEdges.add(edge);
								trailChanged = true;
							}
							reachedCells.add(nextIndex);
							mazeTrailCell = nextIndex;
							if (nextIndex === maze.cells.length - 1 && !mazeCompleted) {
								mazeCompleted = true;
								mazeTimer = window.setTimeout(enterAds, 700);
							}
						} else if (reachedCells.has(nextIndex)) {
							mazeOnTrail = true;
							mazeTrailCell = nextIndex;
						}
					} else {
						mazeOnTrail = false;
						mazeTrailCell = null;
					}
				}
			}
			mazePointerCell = nextIndex >= 0 ? nextIndex : null;
			previousPoint = sample;
		}
		mazePointerPosition = point;
		if (trailChanged) drawMaze();
	}

	function handleMazePointerLeave() {
		mazePointerPosition = null;
		mazePointerCell = null;
		mazeOnTrail = false;
		mazeTrailCell = null;
		mazeOutsideRouteActive = false;
	}

	function enterMaze() {
		window.cancelAnimationFrame(cursorFrame);
		window.clearTimeout(mazeTimer);
		cursorFrame = 0;
		maze = null;
		trailEdges.clear();
		mazeOutsideTrail = [];
		reachedCells.clear();
		mazePointerCell = null;
		mazeTrailCell = null;
		mazeOnTrail = false;
		mazeOutsideRouteActive = false;
		mazePointerPosition = null;
		mazeStarted = false;
		mazeCompleted = false;
		screen = 'maze';
		window.requestAnimationFrame(drawMaze);
	}

	function enterAds() {
		window.clearTimeout(mazeTimer);
		screen = 'ads';
		visibleAds = 0;
		adsClosed = false;
		exitWindowClosed = false;
		exitWindowVisible = false;
		adTimers = ads.map((_, index) =>
			window.setTimeout(() => {
				visibleAds = index + 1;
				if (index === ads.length - 1) {
					exitTimer = window.setTimeout(() => (exitWindowVisible = true), 1100);
				}
			}, 700 + index * 760)
		);
	}

	function moveCloseButton() {
		if (closeMoves >= 6) return;
		const maxX = Math.max(12, window.innerWidth - 48);
		const maxY = Math.max(16, window.innerHeight - 48);
		closeMoves += 1;
		closePosition = {
			x: 12 + Math.random() * Math.max(1, maxX - 12),
			y: 16 + Math.random() * Math.max(1, maxY - 16)
		};
	}

	function handleCloseHover(event: PointerEvent) {
		if (event.pointerType === 'mouse') moveCloseButton();
	}

	function handleCloseClick(event: MouseEvent) {
		if (event.detail === 0 || closeMoves >= 6) {
			window.clearTimeout(exitTimer);
			exitWindowVisible = false;
			exitWindowClosed = true;
			adsClosed = true;
			visibleAds = 0;
		}
		else moveCloseButton();
	}

	onMount(() => {
		animateCursors();
		const resize = () => {
			if (screen === 'maze') drawMaze();
		};
		window.addEventListener('resize', resize);
		return () => {
			window.cancelAnimationFrame(cursorFrame);
			window.clearTimeout(mazeTimer);
			window.clearTimeout(exitTimer);
			adTimers.forEach((timer) => window.clearTimeout(timer));
			window.removeEventListener('resize', resize);
		};
	});
</script>

<svelte:head>
	<title>sosite — нажми сюда</title>
	<meta name="description" content="Нажми сюда. Что может пойти не так?" />
</svelte:head>

{#if screen === 'cursors'}
	<main class="cursor-screen">
		<canvas bind:this={cursorCanvas} class="cursor-canvas" aria-hidden="true"></canvas>
		<div class="browser-bar">
			<div class="browser-tabs"><span class="tab-dot"></span><span>Новая вкладка</span><span class="tab-close">×</span></div>
			<div class="address-row"><span class="back-arrow">‹</span><span class="address">http://sosite.ru/нажми-сюда</span><span class="refresh">↻</span><a href={resolve('/shawarma')} class="other-page">⋮</a></div>
		</div>
		<button class="click-me" type="button" onclick={enterMaze}>Click me</button>
		<div class="corner-note">Лучше нажать мышкой</div>
	</main>
{:else if screen === 'maze'}
	<main class="maze-screen">
		<div class="maze-toolbar"><span>ПРОВЕРКА ПОДКЛЮЧЕНИЯ</span><span>ЗАГРУЗКА... НЕ ВЫКЛЮЧАЙТЕ</span></div>
		<canvas
			bind:this={mazeCanvas}
			class="maze-canvas"
			aria-label="Лабиринт от оранжевого входа до зелёного выхода"
			onpointermove={handleMazePointerMove}
			onpointerleave={handleMazePointerLeave}
		></canvas>
		<div class="maze-footer">Пожалуйста, подождите. Пожалуйста, подождите. Пожалуйста, подождите.</div>
	</main>
{:else}
	<main class="ad-screen">
		<div class="old-site-backdrop" aria-hidden="true">
			<div class="fake-page-header">ДОБРО ПОЖАЛОВАТЬ НА САМЫЙ ЛУЧШИЙ САЙТ В ИНТЕРНЕТЕ</div>
			<div class="fake-page-lines"><i></i><i></i><i></i><i></i><i></i><i></i></div>
			<div class="fake-pixel-art"><span>WWW</span><b>!!!</b></div>
		</div>
		{#if !adsClosed}
			{#each ads.slice(0, visibleAds) as ad, index (ad.brandName + index)}
				<article class={`ad-window ${ad.className} ad-position-${index}`} style={`--window-order:${index};`}>
					<div class="ad-window-bar"><span class="ad-window-icon">▣</span><span>{ad.brandName}</span><span class="window-controls"><i>_</i><i>□</i><i>×</i></span></div>
					<div class="ad-window-body">
						<div class={`brand-tile ${ad.className}`}><span>{ad.brand}</span><small>{ad.className === 'yandex' ? 'БРАУЗЕР' : 'МЕССЕНДЖЕР'}</small></div>
						<div class="ad-copy"><h1>{ad.title}</h1><p>{ad.copy}</p></div>
					</div>
					<a class="download-button" href={ad.url} target="_blank" rel="noreferrer">{ad.action} <span>»</span></a>
				</article>
			{/each}
		{/if}
		<div class="ad-desktop-footer"><a href={resolve('/shawarma')}>sosite: ещё один сайт, который не спросил</a><span>Вы всё ещё смотрите? Как интересно.</span></div>
	</main>
	{#if exitWindowVisible && !exitWindowClosed}
		<div class="exit-shade" aria-hidden="true"></div>
		<dialog open class="exit-window" aria-labelledby="exit-title">
			<div class="exit-titlebar"><span class="warning-icon">!</span><span>Важное предложение</span>
				{#if closeMoves === 0}
					<button
						class="runaway-close"
						type="button"
						aria-label="Закрыть окно"
						title="Закрыть окно"
						onpointerenter={handleCloseHover}
						onclick={handleCloseClick}
					>×</button>
				{/if}
			</div>
			<div class="exit-window-body">
				<div class="exit-fire">Я</div>
				<div><h2 id="exit-title">УХОДИТЕ УЖЕ?</h2><p>Сначала скачайте Яндекс Браузер и MAX. Потом ещё раз попробуйте закрыть это окно.</p></div>
			</div>
			<div class="exit-window-actions"><a href="https://browser.yandex.ru/" target="_blank" rel="noreferrer">СКАЧАТЬ</a><a href="https://max.ru/" target="_blank" rel="noreferrer">УСТАНОВИТЬ MAX</a></div>
		</dialog>
		{#if closeMoves > 0}
			<button
				class="runaway-close escaped"
				style={`--close-left:${closePosition.x}px;--close-top:${closePosition.y}px;`}
				type="button"
				aria-label="Закрыть окно"
				title="Закрыть окно"
				onpointerenter={handleCloseHover}
				onclick={handleCloseClick}
			>×</button>
		{/if}
	{/if}
	{#if exitWindowClosed}
		<div class="closed-note" role="status"><span>ВСЁ ЗАКРЫТО.</span><a href={resolve('/shawarma')}>ОТКРЫТЬ ШАУРМУ ↗</a></div>
	{/if}
{/if}

<style>
	:global(*) { box-sizing: border-box; }
	:global(body) { margin: 0; background: white; color: #111; font-family: Arial, Helvetica, sans-serif; }
	:global(button), :global(a) { font: inherit; }
	.cursor-screen, .maze-screen, .ad-screen { position: relative; min-height: 100vh; width: 100%; overflow: hidden; }
	.cursor-screen { background: #fff; }
	.cursor-canvas { position: fixed; z-index: 1; top: 54px; left: 0; display: block; width: 100vw; height: calc(100dvh - 54px); pointer-events: none; }
	.browser-bar { position: relative; z-index: 3; height: 54px; background: #080808; color: #ddd; font: 10px Arial, sans-serif; }
	.browser-tabs { display: flex; height: 22px; align-items: center; gap: 7px; width: 190px; padding: 0 8px; border-bottom: 1px solid #777; border-radius: 0 0 5px 0; background: #171717; }
	.tab-dot { width: 8px; aspect-ratio: 1; border-radius: 50%; background: #f5f5f5; }
	.tab-close { margin-left: auto; font-size: 13px; }
	.address-row { display: flex; height: 30px; align-items: center; gap: 12px; padding: 0 8px; border-bottom: 1px solid #464646; }
	.back-arrow { color: #aaa; font-size: 19px; }
	.address { display: flex; height: 20px; flex: 1; align-items: center; padding: 0 9px; border-radius: 2px; background: #262626; color: #999; }
	.refresh, .other-page { color: #ddd; text-decoration: none; font-size: 15px; }
	.click-me { position: absolute; z-index: 5; top: 48%; left: 50%; padding: 2px 5px 3px; border: 1px solid #9f00ae; border-radius: 0; background: #d000cf; color: white; font-family: Arial, sans-serif; font-size: 15px; line-height: 1; cursor: pointer; transform: translate(-50%, -50%); }
	.click-me:hover { background: #f000e9; }
	.corner-note { position: absolute; right: 4px; bottom: 3px; z-index: 2; color: #a5a5a5; font: 8px Arial, sans-serif; }
	.maze-screen { display: flex; height: 100dvh; flex-direction: column; align-items: stretch; background: #fff; }
	.maze-toolbar { display: flex; width: 100%; justify-content: space-between; gap: 12px; padding: 5px 8px; background: #101010; color: #eee; font: 9px 'Courier New', monospace; }
	.maze-canvas { display: block; width: 100%; min-height: 0; flex: 1; border: 1px solid #262626; background: #fff; image-rendering: pixelated; touch-action: none; }
	.maze-footer { width: 100%; min-height: 20px; padding: 4px 8px; overflow: hidden; background: #ececec; color: #555; font: 9px 'Courier New', monospace; white-space: nowrap; }
	.ad-screen { min-height: 100vh; background-color: #3f2918; background-image: repeating-linear-gradient(0deg, #674629 0 13px, #7c5935 13px 16px, #53351f 16px 22px, #8b6640 22px 25px); }
	.old-site-backdrop { position: absolute; inset: 0; padding: 25px 5%; color: #e9c891; font-family: 'Courier New', monospace; }
	.fake-page-header { width: min(70%, 840px); height: 62px; overflow: hidden; padding: 13px; border: 3px ridge #af9b77; background: #27190f; color: #ffca32; font-size: clamp(12px, 2vw, 22px); font-weight: bold; text-align: center; text-shadow: 2px 2px #9b351a; }
	.fake-page-lines { display: grid; width: min(64%, 760px); gap: 10px; margin: 25px auto; }
	.fake-page-lines i { display: block; height: 9px; background: #d2bb8d; opacity: 0.7; }
	.fake-page-lines i:nth-child(2n) { width: 73%; }
	.fake-page-lines i:nth-child(3n) { width: 90%; }
	.fake-pixel-art { position: absolute; right: 10%; bottom: 8%; display: flex; width: 180px; height: 110px; align-items: center; justify-content: center; border: 6px ridge #a38b61; background: #d4a339; color: #8d1b1b; font: bold 28px 'Courier New', monospace; transform: rotate(2deg); }
	.fake-pixel-art b { position: absolute; right: 8px; bottom: 2px; color: #f6f0ce; font-size: 40px; }
	.ad-window { position: fixed; z-index: calc(40 + var(--window-order)); width: min(330px, 86vw); min-height: 230px; padding: 4px; border: 4px double #a99b83; background: #050505; color: #f2eee8; box-shadow: 5px 6px 0 #0b0806; font-family: 'Courier New', monospace; animation: pop-window 180ms steps(2, end); }
	.ad-position-0 { top: 6%; left: 4%; }
	.ad-position-1 { top: 18%; left: 34%; }
	.ad-position-2 { top: 8%; right: 4%; left: auto; }
	.ad-position-3 { top: 45%; left: 8%; }
	.ad-position-4 { top: 39%; left: 42%; }
	.ad-position-5 { top: 61%; right: 7%; left: auto; }
	.ad-position-6 { top: 68%; left: 27%; }
	.ad-position-7 { top: 24%; left: 58%; }
	.ad-window-bar { display: flex; height: 23px; align-items: center; gap: 6px; padding: 0 5px; border-bottom: 2px solid #8b7d6b; background: #17130f; color: #f6eedf; font-size: 8px; font-weight: bold; }
	.ad-window-icon { color: #ffc626; }
	.window-controls { display: flex; gap: 2px; margin-left: auto; }
	.window-controls i { display: grid; width: 16px; height: 15px; place-items: center; border: 1px solid #a99b83; color: #fff; font: normal 11px Arial, sans-serif; }
	.ad-window-body { display: flex; min-height: 152px; align-items: center; gap: 12px; padding: 12px 10px 8px; background: #070707; }
	.brand-tile { display: flex; width: 84px; height: 95px; flex: 0 0 auto; flex-direction: column; align-items: center; justify-content: center; border: 4px outset #dbcaa6; background: #a96f17; color: white; image-rendering: pixelated; }
	.brand-tile > span { display: grid; width: 55px; aspect-ratio: 1; place-items: center; background: #ed281e; color: #fff; font: bold 36px Arial, sans-serif; }
	.brand-tile small { margin-top: 5px; font-size: 7px; }
	.brand-tile.max { background: #35234a; }
	.brand-tile.max > span { border-radius: 4px 4px 4px 0; background: #8429e5; font-size: 29px; }
	.ad-copy { min-width: 0; }
	.ad-copy h1 { margin: 0 0 9px; color: #fff; font: bold clamp(16px, 1.8vw, 21px)/0.98 'Courier New', monospace; }
	.ad-copy p { margin: 0; color: #ddd5c6; font: 11px/1.3 Arial, sans-serif; }
	.download-button { display: flex; min-height: 32px; align-items: center; justify-content: space-between; gap: 8px; padding: 5px 8px; border: 3px outset #b6ab94; background: #8c1b1a; color: #fff8dc; font: bold 9px 'Courier New', monospace; text-align: center; text-decoration: none; }
	.download-button span { font-size: 16px; }
	.max .download-button, .ad-window.max .download-button { background: #6824a9; }
	.ad-desktop-footer { position: fixed; right: 8px; bottom: 6px; left: 8px; z-index: 70; display: flex; justify-content: space-between; gap: 12px; color: #f3e5ce; font: 10px 'Courier New', monospace; text-shadow: 1px 1px #000; }
	.ad-desktop-footer a { color: #fff36b; }
	.exit-shade { position: fixed; z-index: 78; inset: 0; background: rgb(0 0 0 / 35%); }
	.exit-window { position: fixed; z-index: 80; top: 50%; left: 50%; width: min(390px, calc(100vw - 30px)); padding: 4px; border: 5px double #b1a38b; background: #090909; color: #fff; box-shadow: 9px 10px 0 #000; transform: translate(-50%, -50%); animation: pop-window 200ms steps(2, end); }
	.exit-titlebar { position: relative; display: flex; height: 28px; align-items: center; gap: 7px; padding: 0 38px 0 7px; border-bottom: 2px solid #9f8a6c; background: #18130e; color: #fff; font: bold 10px 'Courier New', monospace; }
	.warning-icon { display: grid; width: 17px; aspect-ratio: 1; place-items: center; background: #fc1e33; color: #fff; font: bold 13px Arial, sans-serif; }
	.runaway-close { position: absolute; top: 3px; right: 4px; display: grid; width: 20px; height: 19px; place-items: center; border: 1px solid #afa28c; background: #17130f; color: #fff; font: 14px Arial, sans-serif; cursor: pointer; }
	.runaway-close.escaped { position: fixed; z-index: 100; top: var(--close-top); left: var(--close-left); right: auto; width: 27px; height: 25px; border: 2px outset #f4ead8; background: #130f0b; box-shadow: 3px 3px #000; }
	.exit-window-body { display: flex; align-items: center; gap: 13px; padding: 15px 11px 12px; }
	.exit-fire { display: grid; width: 72px; aspect-ratio: 1; flex: 0 0 auto; place-items: center; background: #b5751b; color: #fff; font: bold 50px Arial, sans-serif; image-rendering: pixelated; box-shadow: inset 8px 0 #f4cf25, inset -8px 0 #d72a23; }
	.exit-window-body h2 { margin: 0 0 8px; color: #fff; font: bold 20px 'Courier New', monospace; }
	.exit-window-body p { margin: 0; color: #d4cbbd; font: 11px/1.4 Arial, sans-serif; }
	.exit-window-actions { display: flex; justify-content: space-between; gap: 8px; padding: 8px 6px 4px; border-top: 1px solid #86765f; }
	.exit-window-actions a { flex: 1; padding: 8px 4px; border: 3px outset #b6ab94; background: #8c1b1a; color: #fff; font: bold 9px 'Courier New', monospace; text-align: center; text-decoration: none; }
	.exit-window-actions a:last-child { background: #6824a9; }
	.closed-note { position: fixed; z-index: 110; bottom: 26px; left: 50%; display: flex; max-width: calc(100vw - 24px); align-items: center; gap: 12px; padding: 8px 10px; border: 3px outset #c4b699; background: #050505; color: #fff; font: bold 10px 'Courier New', monospace; transform: translateX(-50%); }
	.closed-note a { color: #fff36b; white-space: nowrap; }
	:global(button:focus-visible), :global(a:focus-visible) { outline: 3px solid #ec2ec8; outline-offset: 3px; }
	@keyframes pop-window { from { scale: 0.94; opacity: 0; } to { scale: 1; opacity: 1; } }
	@media (max-width: 650px) {
		.ad-position-0 { top: 5%; left: 2%; }
		.ad-position-1 { top: 24%; left: 11%; }
		.ad-position-2 { top: 6%; right: 2%; left: auto; }
		.ad-position-3 { top: 43%; left: 2%; }
		.ad-position-4 { top: 40%; left: 15%; }
		.ad-position-5 { top: 61%; right: 2%; left: auto; }
		.ad-position-6 { top: 68%; left: 5%; }
		.ad-position-7 { top: 25%; left: 20%; }
		.ad-window { width: min(290px, 82vw); min-height: 205px; }
		.ad-window-body { min-height: 130px; gap: 8px; padding-inline: 6px; }
		.brand-tile { width: 66px; height: 81px; }
		.brand-tile > span { width: 45px; font-size: 30px; }
		.brand-tile.max > span { font-size: 24px; }
		.ad-copy h1 { font-size: 16px; }
		.ad-copy p { font-size: 10px; }
		.download-button { font-size: 8px; }
		.ad-desktop-footer { flex-direction: column; gap: 3px; font-size: 8px; }
		.exit-window-body h2 { font-size: 17px; }
		.exit-fire { width: 60px; font-size: 42px; }
	}
	@media (prefers-reduced-motion: reduce) { *, *::before, *::after { animation-duration: 0.01ms !important; animation-iteration-count: 1 !important; transition-duration: 0.01ms !important; } }
</style>
