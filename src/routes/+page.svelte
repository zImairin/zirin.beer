<!-- <script lang="ts">
	import { resolve } from '$app/paths';

	let darkMode = $state(false);
	let clicked = $state(false);

	function toggleTheme() {
		darkMode = !darkMode;
	}
</script>

<main class:dark={darkMode}>
	<header class="topbar">
		<a class="logo" href={resolve('/')} aria-label="На главную">sosite</a>
		<button
			class="theme-toggle"
			type="button"
			aria-label={darkMode ? 'Включить светлую тему' : 'Включить тёмную тему'}
			aria-pressed={darkMode}
			onclick={toggleTheme}
		>
			<span aria-hidden="true">{darkMode ? '☀' : '☾'}</span>
			{darkMode ? 'Светлая тема' : 'Тёмная тема'}
		</button>
	</header>

	<section class="hero" aria-labelledby="hero-title">
		<div class="eyebrow">Простое начало</div>
		<h1 id="hero-title">Добро пожаловать.</h1>
		<p>Минимальная страница, на которой всё по делу.</p>
		<button class="primary-button" type="button" onclick={() => (clicked = true)}>
			{clicked ? 'Готово!' : 'Нажать кнопку'}
		</button>
	</section>
</main>

<style>
	main {
		--background: #f5f5f0;
		--surface: #ffffff;
		--text: #171717;
		--muted: #737373;
		--accent: #171717;
		--accent-text: #ffffff;
		min-height: 100vh;
		background: var(--background);
		color: var(--text);
		transition:
			background 180ms ease,
			color 180ms ease;
	}

	main.dark {
		--background: #171717;
		--surface: #242424;
		--text: #f5f5f0;
		--muted: #a3a3a3;
		--accent: #f5f5f0;
		--accent-text: #171717;
	}

	.topbar {
		display: flex;
		align-items: center;
		justify-content: space-between;
		max-width: 1080px;
		margin: 0 auto;
		padding: 28px 32px;
	}

	.logo {
		color: var(--text);
		font-size: 1.15rem;
		font-weight: 700;
		letter-spacing: -0.04em;
		text-decoration: none;
	}

	button {
		font: inherit;
		cursor: pointer;
	}

	.theme-toggle {
		display: inline-flex;
		align-items: center;
		gap: 8px;
		border: 1px solid color-mix(in srgb, var(--text) 18%, transparent);
		border-radius: 999px;
		padding: 9px 14px;
		background: var(--surface);
		color: var(--text);
		font-size: 0.85rem;
		transition:
			background 180ms ease,
			border-color 180ms ease;
	}

	.theme-toggle:hover {
		border-color: var(--text);
	}

	.hero {
		display: flex;
		flex-direction: column;
		align-items: flex-start;
		max-width: 1080px;
		margin: 0 auto;
		padding: clamp(110px, 20vh, 220px) 32px;
	}

	.eyebrow {
		margin-bottom: 20px;
		color: var(--muted);
		font-size: 0.78rem;
		font-weight: 700;
		letter-spacing: 0.12em;
		text-transform: uppercase;
	}

	h1 {
		max-width: 700px;
		margin: 0;
		font-size: clamp(3.2rem, 8vw, 7rem);
		font-weight: 700;
		letter-spacing: -0.08em;
		line-height: 0.95;
	}

	p {
		margin: 28px 0 36px;
		color: var(--muted);
		font-size: clamp(1.05rem, 2vw, 1.25rem);
	}

	.primary-button {
		border: 0;
		border-radius: 999px;
		padding: 14px 22px;
		background: var(--accent);
		color: var(--accent-text);
		font-weight: 700;
		transition:
			transform 180ms ease,
			opacity 180ms ease;
	}

	.primary-button:hover {
		transform: translateY(-2px);
		opacity: 0.86;
	}

	.primary-button:focus-visible,
	.theme-toggle:focus-visible,
	.logo:focus-visible {
		outline: 3px solid #70a1ff;
		outline-offset: 4px;
	}

	@media (max-width: 600px) {
		.topbar {
			padding: 22px 20px;
		}

		.theme-toggle {
			padding: 8px 11px;
		}

		.hero {
			padding: 120px 20px 80px;
		}
	}
</style> -->



<script>
   import { Canvas } from '@threlte/core';
   import SceneOne from '../lib/scene/SceneOne.svelte';
</script>

<main>
    <section class="canvas-wrapper ring h-[700px]">
        <Canvas>
            <SceneOne />
        </Canvas>
    </section>
</main>