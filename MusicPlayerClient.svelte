<script lang="ts">
	import Icon from "@iconify/svelte";
	import { onMount } from "svelte";

	export type Song = {
		id: number;
		name: string;
		artist: string;
		cover: string;
		duration: number;
	};

	let { songs }: { songs: Song[] } = $props();

	let index = $state(0);
	let playing = $state(false);
	let currentTime = $state(0);
	let realDuration = $state(0);
	let volume = $state(0.7);
	let muted = $state(false);
	let mode = $state<"list" | "one" | "shuffle">("list");
	let listOpen = $state(false);

	// 非响应式标记：用户主动切歌时置 true，由 src 的 effect 消费
	let autoplay = false;
	// 连续加载失败计数，避免所有歌都失败时无限跳
	let failCount = 0;

	let audioEl: HTMLAudioElement | undefined = $state();

	const song = $derived(songs[index] ?? songs[0]);
	const duration = $derived(realDuration || (song?.duration ?? 0) / 1000);
	const progress = $derived(duration > 0 ? Math.min(currentTime / duration, 1) : 0);
	const shownVolume = $derived(muted ? 0 : volume);

	// 网易云外链：浏览器直接向它的 CDN 取流，会 302 跳转，不需要 CORS
	const src = $derived(
		song ? `https://music.163.com/song/media/outer/url?id=${song.id}.mp3` : "",
	);

	function fmt(sec: number): string {
		if (!Number.isFinite(sec) || sec < 0) sec = 0;
		const m = Math.floor(sec / 60);
		const s = Math.floor(sec % 60);
		return `${m}:${String(s).padStart(2, "0")}`;
	}

	function toggle() {
		if (!audioEl) return;
		if (playing) {
			audioEl.pause();
		} else {
			audioEl.play().catch(() => {});
		}
	}

	function pick(i: number) {
		autoplay = true;
		index = i;
	}

	function step(delta: number) {
		if (songs.length === 0) return;
		const n =
			mode === "shuffle"
				? Math.floor(Math.random() * songs.length)
				: (index + delta + songs.length) % songs.length;
		autoplay = true;
		index = n;
	}

	function onEnded() {
		if (mode === "one" && audioEl) {
			audioEl.currentTime = 0;
			audioEl.play().catch(() => {});
			return;
		}
		step(1);
	}

	// 某首歌取不到（下架/VIP/地区限制）时跳到下一首；全失败则停下
	function onError() {
		failCount += 1;
		if (failCount >= songs.length) {
			playing = false;
			return;
		}
		step(1);
	}

	function toggleMute() {
		if (muted) {
			muted = false;
			// 音量本来就是 0 时，取消静音要恢复一个默认值，否则按钮会「卡住」
			if (volume === 0) volume = 0.7;
		} else {
			muted = true;
		}
	}

	function cycleMode() {
		mode = mode === "list" ? "one" : mode === "one" ? "shuffle" : "list";
	}

	function applyVolume(v: number) {
		volume = Math.min(Math.max(v, 0), 1);
		muted = volume === 0;
	}

	function seekRatio(r: number) {
		if (!audioEl || duration <= 0) return;
		audioEl.currentTime = Math.min(Math.max(r, 0), 1) * duration;
		currentTime = audioEl.currentTime;
	}

	// 按下即跳转，按住可拖动。用 pointer 事件 + 指针捕获，鼠标和触屏通用。
	function startDrag(
		e: PointerEvent,
		el: HTMLElement,
		onMove: (ratio: number) => void,
	) {
		e.preventDefault();
		const rect = el.getBoundingClientRect();
		const fire = (ev: PointerEvent) =>
			onMove(Math.min(Math.max((ev.clientX - rect.left) / rect.width, 0), 1));
		fire(e);
		try {
			el.setPointerCapture(e.pointerId);
		} catch {
			/* 某些环境不支持捕获，忽略即可 */
		}
		const stop = () => {
			el.removeEventListener("pointermove", fire);
			el.removeEventListener("pointerup", stop);
			el.removeEventListener("pointercancel", stop);
		};
		el.addEventListener("pointermove", fire);
		el.addEventListener("pointerup", stop);
		el.addEventListener("pointercancel", stop);
	}

	// 键盘：→/↑ 前进，←/↓ 后退，Home/End 到首尾；按住 Shift 步长更大
	function progressKey(e: KeyboardEvent) {
		if (!audioEl || duration <= 0) return;
		const step = e.shiftKey ? 30 : 5;
		let t: number | null = null;
		if (e.key === "ArrowRight" || e.key === "ArrowUp") t = currentTime + step;
		else if (e.key === "ArrowLeft" || e.key === "ArrowDown") t = currentTime - step;
		else if (e.key === "Home") t = 0;
		else if (e.key === "End") t = duration;
		if (t === null) return;
		e.preventDefault();
		seekRatio(t / duration);
	}

	function volumeKey(e: KeyboardEvent) {
		const step = 0.05;
		let v: number | null = null;
		if (e.key === "ArrowRight" || e.key === "ArrowUp") v = shownVolume + step;
		else if (e.key === "ArrowLeft" || e.key === "ArrowDown") v = shownVolume - step;
		else if (e.key === "Home") v = 0;
		else if (e.key === "End") v = 1;
		if (v === null) return;
		e.preventDefault();
		applyVolume(v);
	}

	onMount(() => {
		if (audioEl) audioEl.volume = volume;
	});

	// 音量变化同步到 audio
	$effect(() => {
		if (audioEl) audioEl.volume = volume;
	});

	// 切歌：只在「曲目变了」时重置进度。
	// 注意这个 effect 只能依赖 src —— 之前误读了响应式的 playing，
	// 导致一暂停就重跑、把进度清零。现在用非响应式的 autoplay 标记。
	$effect(() => {
		const _ = src;
		if (!audioEl) return;
		currentTime = 0;
		realDuration = 0;
		if (autoplay) audioEl.play().catch(() => {});
	});

	const modeLabel = $derived(
		mode === "list" ? "列表循环" : mode === "one" ? "单曲循环" : "随机播放",
	);
	const modeIcon = $derived(
		mode === "one"
			? "material-symbols:repeat-one-rounded"
			: mode === "shuffle"
				? "material-symbols:shuffle-rounded"
				: "material-symbols:repeat-rounded",
	);
</script>

<div class="mp">
	<div class="mp-head">
		<div class="mp-cover" class:spinning={playing}>
			{#if song?.cover}
				<img src={song.cover} alt={song.name} draggable="false" />
			{:else}
				<div class="mp-cover-fallback">
					<Icon icon="material-symbols:music-note-rounded" />
				</div>
			{/if}
		</div>
		<div class="mp-meta">
			<div class="mp-title" title={song?.name}>{song?.name ?? "—"}</div>
			<div class="mp-artist" title={song?.artist}>{song?.artist ?? ""}</div>

			<div class="mp-row">
				<span class="mp-time">{fmt(currentTime)} / {fmt(duration)}</span>
				<div class="mp-vol-wrap">
					<button
						class="mp-icon-btn"
						aria-label={muted ? "取消静音" : "静音"}
						onclick={toggleMute}
					>
						<Icon
							icon={shownVolume === 0
								? "material-symbols:volume-off-rounded"
								: "material-symbols:volume-up-rounded"}
						/>
					</button>
					<div
						class="mp-vol-bar"
						role="slider"
						tabindex="0"
						aria-label="音量"
						aria-valuenow={Math.round(shownVolume * 100)}
						aria-valuemin="0"
						aria-valuemax="100"
						onpointerdown={(e) => startDrag(e, e.currentTarget, applyVolume)}
						onkeydown={volumeKey}
					>
						<div class="mp-vol-fill" style="width:{shownVolume * 100}%"></div>
					</div>
				</div>
			</div>
		</div>
	</div>

	<div
		class="mp-progress"
		role="slider"
		tabindex="0"
		aria-label="播放进度"
		aria-valuenow={Math.round(currentTime)}
		aria-valuemin="0"
		aria-valuemax={Math.round(duration)}
		onpointerdown={(e) => startDrag(e, e.currentTarget, seekRatio)}
		onkeydown={progressKey}
	>
		<div class="mp-progress-fill" style="width:{progress * 100}%"></div>
	</div>

	<div class="mp-controls">
		<button class="mp-icon-btn mp-mode" aria-label={modeLabel} title={modeLabel} onclick={cycleMode}>
			<Icon icon={modeIcon} />
		</button>
		<button class="mp-icon-btn" aria-label="上一首" onclick={() => step(-1)}>
			<Icon icon="material-symbols:skip-previous-rounded" />
		</button>
		<button class="mp-play" aria-label={playing ? "暂停" : "播放"} onclick={toggle}>
			<Icon icon={playing ? "material-symbols:pause-rounded" : "material-symbols:play-arrow-rounded"} />
		</button>
		<button class="mp-icon-btn" aria-label="下一首" onclick={() => step(1)}>
			<Icon icon="material-symbols:skip-next-rounded" />
		</button>
		<button
			class="mp-icon-btn mp-list-btn"
			class:active={listOpen}
			aria-label={listOpen ? "收起歌单" : "展开歌单"}
			aria-expanded={listOpen}
			onclick={() => (listOpen = !listOpen)}
		>
			<Icon icon="material-symbols:queue-music-rounded" />
		</button>
	</div>

	<!-- 歌单抽屉：grid-template-rows 0fr→1fr 的高度过渡，比 max-height 更平滑 -->
	<div class="mp-drawer" class:open={listOpen}>
		<div class="mp-drawer-inner">
			<div class="mp-list">
				{#each songs as s, i (s.id)}
					<button class="mp-item" class:current={i === index} onclick={() => pick(i)}>
						<span class="mp-item-cover">
							{#if s.cover}<img src={s.cover} alt="" draggable="false" />{/if}
						</span>
						<span class="mp-item-text">
							<span class="mp-item-title" class:active={i === index}>{s.name}</span>
							<span class="mp-item-artist" class:active={i === index}>{s.artist}</span>
						</span>
					</button>
				{/each}
			</div>
		</div>
	</div>

	<!-- 隐藏的音频元素 -->
	<audio
		bind:this={audioEl}
		src={src}
		preload="metadata"
		onplay={() => {
			playing = true;
			failCount = 0;
		}}
		onpause={() => (playing = false)}
		ontimeupdate={() => audioEl && (currentTime = audioEl.currentTime)}
		onloadedmetadata={() => audioEl && (realDuration = audioEl.duration)}
		onended={onEnded}
		onerror={onError}
	></audio>
</div>

<style>
	.mp {
		padding: 0.5rem 0 0.25rem;
	}

	/* 封面 + 曲目信息 */
	.mp-head {
		display: flex;
		align-items: center;
		gap: 0.7rem;
		margin-bottom: 0.5rem;
	}
	.mp-cover {
		width: 3.5rem;
		height: 3.5rem;
		flex: 0 0 3.5rem;
		border-radius: 9999px;
		overflow: hidden;
		background: var(--btn-regular-bg);
	}
	.mp-cover img {
		width: 100%;
		height: 100%;
		object-fit: cover;
		display: block;
	}
	.mp-cover img {
		/* 动画常驻，暂停只切 play-state —— 否则移除 animation 会让角度重置回 0 */
		animation: mp-spin 12s linear infinite;
		animation-play-state: paused;
		transform-origin: center;
	}
	.mp-cover.spinning img {
		animation-play-state: running;
	}
	.mp-cover-fallback {
		width: 100%;
		height: 100%;
		display: flex;
		align-items: center;
		justify-content: center;
		font-size: 1.5rem;
		color: var(--btn-content);
	}
	@keyframes mp-spin {
		from {
			transform: rotate(0);
		}
		to {
			transform: rotate(360deg);
		}
	}

	.mp-meta {
		min-width: 0;
		flex: 1;
	}
	.mp-title {
		font-size: 0.875rem;
		font-weight: 700;
		color: var(--deep-text);
		line-height: 1.35;
		display: -webkit-box;
		-webkit-line-clamp: 2;
		line-clamp: 2;
		-webkit-box-orient: vertical;
		overflow: hidden;
	}
	:global(:root.dark) .mp-title {
		color: #f5f5f5;
	}
	.mp-artist {
		font-size: 0.75rem;
		line-height: 1.35;
		color: var(--btn-content);
		margin-top: 0.05rem;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}

	/* 时间 + 音量 */
	.mp-row {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 0.4rem;
		/* 行高锁成一行文字的高度，使「歌手→时间」的间距与「标题→歌手」一致 */
		height: 1.0125rem;
		margin-top: 0.05rem;
	}
	.mp-vol-wrap {
		height: 100%;
	}
	/* 只缩小时间行里的音量按钮，不影响下面控制栏 */
	.mp-vol-wrap .mp-icon-btn {
		width: 1.0125rem;
		height: 1.0125rem;
		font-size: 0.9rem;
		border-radius: 0.35rem;
	}
	.mp-time {
		font-size: 0.75rem;
		color: var(--btn-content);
		white-space: nowrap;
		font-variant-numeric: tabular-nums;
	}
	.mp-vol-wrap {
		display: flex;
		align-items: center;
		gap: 0.3rem;
		min-width: 0;
	}
	.mp-vol-bar {
		position: relative;
		touch-action: none;
		width: 2.75rem;
		height: 0.25rem;
		border-radius: 9999px;
		background: var(--btn-regular-bg);
		overflow: hidden;
		cursor: pointer;
		transition: height 0.15s ease;
	}
	.mp-vol-bar:hover {
		height: 0.375rem;
	}
	.mp-vol-fill {
		height: 100%;
		background: var(--primary);
		border-radius: inherit;
		transition: width 0.1s linear;
	}

	/* 进度条 */
	.mp-progress {
		position: relative;
		touch-action: none;
		width: 100%;
		height: 0.375rem;
		border-radius: 9999px;
		background: var(--btn-regular-bg);
		overflow: hidden;
		cursor: pointer;
		margin-top: 0.6rem;
		margin-bottom: 0.6rem;
	}
	.mp-progress-fill {
		height: 100%;
		background: var(--primary);
		border-radius: inherit;
		transition: width 0.1s linear;
	}

	/* 控制栏 */
	.mp-controls {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 0.15rem;
	}
	.mp-icon-btn {
		display: flex;
		align-items: center;
		justify-content: center;
		width: 1.9rem;
		height: 1.9rem;
		border-radius: 0.5rem;
		color: var(--btn-content);
		font-size: 1.15rem;
		transition:
			color 0.15s ease,
			transform 0.15s ease,
			background-color 0.15s ease;
	}
	.mp-icon-btn:hover {
		color: var(--primary);
		background: var(--btn-plain-bg-hover);
	}
	.mp-icon-btn:active {
		transform: scale(0.94);
	}
	.mp-icon-btn.active {
		color: var(--primary);
	}
	.mp-play {
		display: flex;
		align-items: center;
		justify-content: center;
		width: 2.5rem;
		height: 2.5rem;
		border-radius: 9999px;
		background: var(--btn-regular-bg);
		color: var(--primary);
		font-size: 1.5rem;
		flex: 0 0 2.5rem;
		transition:
			background-color 0.15s ease,
			transform 0.15s ease;
	}
	.mp-play:hover {
		background: var(--btn-regular-bg-hover);
	}
	.mp-play:active {
		transform: scale(0.94);
	}

	/* 歌单抽屉 —— 关键动画 */
	.mp-drawer {
		display: grid;
		grid-template-rows: 0fr;
		opacity: 0;
		transition:
			grid-template-rows 0.3s cubic-bezier(0.4, 0, 0.2, 1),
			opacity 0.3s cubic-bezier(0.4, 0, 0.2, 1),
			margin-top 0.3s cubic-bezier(0.4, 0, 0.2, 1);
	}
	.mp-drawer.open {
		grid-template-rows: 1fr;
		opacity: 1;
		margin-top: 0.5rem;
	}
	.mp-drawer-inner {
		overflow: hidden;
		min-height: 0;
	}
	.mp-list {
		max-height: 11rem;
		overflow-y: auto;
		display: flex;
		flex-direction: column;
		gap: 0.2rem;
		padding-top: 0.5rem;
		border-top: 1px solid var(--line-divider);
		scrollbar-width: none;
	}
	.mp-list::-webkit-scrollbar {
		display: none;
	}
	.mp-item {
		display: flex;
		align-items: center;
		gap: 0.6rem;
		padding: 0.4rem;
		border-radius: 0.6rem;
		text-align: left;
		transition: background-color 0.18s ease;
	}
	.mp-item:hover {
		background: var(--btn-plain-bg-hover);
	}
	.mp-item.current {
		background: var(--btn-regular-bg);
	}
	.mp-item-cover {
		width: 2rem;
		height: 2rem;
		flex: 0 0 2rem;
		border-radius: 0.4rem;
		overflow: hidden;
		background: var(--btn-regular-bg);
	}
	.mp-item-cover img {
		width: 100%;
		height: 100%;
		object-fit: cover;
		display: block;
	}
	.mp-item-text {
		min-width: 0;
		display: flex;
		flex-direction: column;
	}
	.mp-item-title {
		font-size: 0.75rem;
		font-weight: 700;
		color: var(--deep-text);
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}
	.mp-item-artist {
		font-size: 0.625rem;
		color: var(--btn-content);
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}
	:global(:root.dark) .mp-item-title {
		color: #e5e5e5;
	}
	.mp-item-title.active,
	.mp-item-artist.active {
		color: var(--primary);
	}
</style>
