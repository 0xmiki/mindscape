<script lang="ts">
	import { browser } from '$app/environment';
	import createDOMPurify from 'dompurify';
	import { marked } from 'marked';
	import { onMount, tick } from 'svelte';

	marked.setOptions({ gfm: true, breaks: true });
	const markdownPurifier = browser ? createDOMPurify(window) : null;

	type Mind = {
		id: string;
		image: string;
		mediaType?: 'image' | 'video';
		idea: string;
		worldX: number;
		worldY: number;
		width: number;
		aspectRatio: number;
		borderRadius: number;
		tilt: number;
		createdAt: number;
	};

	type LegacyMind = Partial<Mind> &
		Pick<Mind, 'id' | 'image' | 'idea' | 'tilt' | 'createdAt'> & {
			x?: number;
			y?: number;
			size?: number;
		};

	const DB_NAME = 'mindscape';
	const STORE_NAME = 'ideas';
	const DB_VERSION = 1;
	const EDGE_OVERLAP = 12;
	const MIN_MEDIA_WIDTH = 120;
	const MAX_MEDIA_WIDTH = 560;
	const DEFAULT_BORDER_RADIUS_PERCENT = 4;
	const MAX_BORDER_RADIUS_PERCENT = 100;
	const BASE_CANVAS_WIDTH = 1800;
	const BASE_ORIGIN_X = 900;
	const LEGACY_BASE_CANVAS_HEIGHT = 1200;
	const LEGACY_BASE_ORIGIN_Y = 600;
	const CANVAS_EXPANSION = 280;
	const MARKDOWN_PLACEHOLDER =
		'# Name this idea\n\nWrite what it means, why it matters, and where it might lead…';

	let minds: Mind[] = $state([]);
	let composing = $state(false);
	let selected: Mind | null = $state(null);
	let editingMindId: string | null = $state(null);
	let draftImage = $state('');
	let draftMediaType: 'image' | 'video' = $state('image');
	let draftAspectRatio = $state(4 / 3);
	let draftBorderRadius = $state(DEFAULT_BORDER_RADIUS_PERCENT);
	let draftIdea = $state('');
	let radiusPreviewVisible = $state(false);
	let isDragging = $state(false);
	let isSaving = $state(false);
	let loadError = $state('');
	let canvasWidth = $state(BASE_CANVAS_WIDTH);
	let baseCanvasHeight = $state(0);
	let canvasHeight = $state(0);
	let originX = $state(BASE_ORIGIN_X);
	let originY = $state(0);
	let resizeState: { id: string; startX: number; startWidth: number } | null = $state(null);
	let layoutSave: Promise<void> = Promise.resolve();
	let savedScrollX: number | null = null;
	let savedScrollY: number | null = null;
	let scrollSaveFrame = 0;
	let radiusPreviewTimer: ReturnType<typeof setTimeout> | undefined;
	let fileInput: HTMLInputElement = $state()!;
	let ideaInput: HTMLTextAreaElement = $state()!;
	let canvasViewport: HTMLElement = $state()!;
	let draftPreview = $derived(renderMarkdown(draftIdea));

	function renderMarkdown(source: string) {
		if (!markdownPurifier || !source.trim()) return '';
		return markdownPurifier.sanitize(String(marked.parse(source)));
	}

	function openDatabase(): Promise<IDBDatabase> {
		return new Promise((resolve, reject) => {
			const request = indexedDB.open(DB_NAME, DB_VERSION);
			request.onupgradeneeded = () => {
				const database = request.result;
				if (!database.objectStoreNames.contains(STORE_NAME)) {
					database.createObjectStore(STORE_NAME, { keyPath: 'id' });
				}
			};
			request.onsuccess = () => resolve(request.result);
			request.onerror = () => reject(request.error);
		});
	}

	async function readMinds() {
		const database = await openDatabase();
		return new Promise<Mind[]>((resolve, reject) => {
			const transaction = database.transaction(STORE_NAME, 'readonly');
			const request = transaction.objectStore(STORE_NAME).getAll();
			request.onsuccess = () => {
				const saved = (request.result as LegacyMind[]).sort((a, b) => a.createdAt - b.createdAt);
				resolve(saved.map(normalizeMind));
			};
			request.onerror = () => reject(request.error);
			transaction.oncomplete = () => database.close();
		});
	}

	async function storeMind(mind: Mind) {
		const database = await openDatabase();
		return new Promise<void>((resolve, reject) => {
			const transaction = database.transaction(STORE_NAME, 'readwrite');
			transaction.objectStore(STORE_NAME).put(mind);
			transaction.oncomplete = () => {
				database.close();
				resolve();
			};
			transaction.onerror = () => reject(transaction.error);
		});
	}

	async function removeMind(id: string) {
		const database = await openDatabase();
		return new Promise<void>((resolve, reject) => {
			const transaction = database.transaction(STORE_NAME, 'readwrite');
			transaction.objectStore(STORE_NAME).delete(id);
			transaction.oncomplete = () => {
				database.close();
				resolve();
			};
			transaction.onerror = () => reject(transaction.error);
		});
	}

	async function storeMinds(entries: Mind[]) {
		if (!entries.length) return;
		const database = await openDatabase();
		return new Promise<void>((resolve, reject) => {
			const transaction = database.transaction(STORE_NAME, 'readwrite');
			const store = transaction.objectStore(STORE_NAME);
			for (const entry of entries) store.put(entry);
			transaction.oncomplete = () => {
				database.close();
				resolve();
			};
			transaction.onerror = () => reject(transaction.error);
		});
	}

	function queueLayoutSave(entries: Mind[], errorMessage: string) {
		const snapshot = entries.map((mind) => ({ ...mind }));
		layoutSave = layoutSave
			.then(() => storeMinds(snapshot))
			.catch(() => {
				loadError = errorMessage;
			});
	}

	function normalizeMind(mind: LegacyMind, index: number): Mind {
		const legacyX = typeof mind.x === 'number' ? ((mind.x - 50) / 100) * 1200 : index * 24;
		const legacyY = typeof mind.y === 'number' ? ((mind.y - 50) / 100) * 760 : index * 18;
		return {
			id: mind.id,
			image: mind.image,
			mediaType: mind.mediaType,
			idea: mind.idea,
			worldX: Number.isFinite(mind.worldX) ? Number(mind.worldX) : legacyX,
			worldY: Number.isFinite(mind.worldY) ? Number(mind.worldY) : legacyY,
			width: Number.isFinite(mind.width) ? Number(mind.width) : mind.size || 220,
			aspectRatio:
				Number.isFinite(mind.aspectRatio) && Number(mind.aspectRatio) > 0
					? Number(mind.aspectRatio)
					: 4 / 3,
			borderRadius: Number.isFinite(mind.borderRadius)
				? Math.max(0, Math.min(MAX_BORDER_RADIUS_PERCENT, Number(mind.borderRadius)))
				: DEFAULT_BORDER_RADIUS_PERCENT,
			tilt: mind.tilt,
			createdAt: mind.createdAt
		};
	}

	function isVideo(media: Pick<Mind, 'image' | 'mediaType'>) {
		return (
			media.mediaType === 'video' || (!media.mediaType && media.image.startsWith('data:video/'))
		);
	}

	function measureMedia(source: string, mediaType: 'image' | 'video') {
		return new Promise<number>((resolve) => {
			if (mediaType === 'video') {
				const video = document.createElement('video');
				video.preload = 'metadata';
				video.onloadedmetadata = () =>
					resolve(
						video.videoWidth && video.videoHeight ? video.videoWidth / video.videoHeight : 4 / 3
					);
				video.onerror = () => resolve(4 / 3);
				video.src = source;
				return;
			}

			const image = new Image();
			image.onload = () => resolve(image.naturalWidth / image.naturalHeight || 4 / 3);
			image.onerror = () => resolve(4 / 3);
			image.src = source;
		});
	}

	function readMedia(file?: File) {
		if (!file || (!file.type.startsWith('image/') && !file.type.startsWith('video/'))) return;
		loadError = '';
		const reader = new FileReader();
		reader.onload = async () => {
			const source = String(reader.result);
			const mediaType = file.type.startsWith('video/') ? 'video' : 'image';
			draftImage = source;
			draftMediaType = mediaType;
			draftAspectRatio = await measureMedia(source, mediaType);
			setTimeout(() => ideaInput?.focus(), 50);
		};
		reader.onerror = () => {
			loadError = 'This file could not be read. Try another image or video.';
		};
		reader.readAsDataURL(file);
	}

	function onPaste(event: ClipboardEvent) {
		const mediaItem = Array.from(event.clipboardData?.items ?? []).find(
			(item) => item.type.startsWith('image/') || item.type.startsWith('video/')
		);
		const mediaFile =
			mediaItem?.getAsFile() ??
			Array.from(event.clipboardData?.files ?? []).find(
				(file) => file.type.startsWith('image/') || file.type.startsWith('video/')
			);

		if (!mediaFile) return;

		event.preventDefault();
		if (!composing) openComposer();
		closeViewer();
		readMedia(mediaFile);
	}

	function onDrop(event: DragEvent) {
		event.preventDefault();
		isDragging = false;
		readMedia(event.dataTransfer?.files[0]);
	}

	function openComposer() {
		editingMindId = null;
		draftImage = '';
		draftMediaType = 'image';
		draftAspectRatio = 4 / 3;
		draftBorderRadius = DEFAULT_BORDER_RADIUS_PERCENT;
		draftIdea = '';
		hideRadiusPreview();
		loadError = '';
		composing = true;
		void tick().then(() => ideaInput?.focus());
	}

	function closeComposer() {
		if (isSaving) return;
		hideRadiusPreview();
		composing = false;
		editingMindId = null;
	}

	function holdRadiusPreview() {
		if (radiusPreviewTimer) clearTimeout(radiusPreviewTimer);
		radiusPreviewVisible = true;
	}

	function showRadiusPreview(event: Event) {
		draftBorderRadius = Number((event.currentTarget as HTMLInputElement).value);
		holdRadiusPreview();
		radiusPreviewTimer = setTimeout(() => {
			radiusPreviewVisible = false;
		}, 850);
	}

	function releaseRadiusPreview() {
		if (radiusPreviewTimer) clearTimeout(radiusPreviewTimer);
		radiusPreviewTimer = setTimeout(() => {
			radiusPreviewVisible = false;
		}, 450);
	}

	function hideRadiusPreview() {
		if (radiusPreviewTimer) clearTimeout(radiusPreviewTimer);
		radiusPreviewTimer = undefined;
		radiusPreviewVisible = false;
	}

	function displayedAspectRatio(aspectRatio: number, borderRadius: number) {
		return borderRadius >= MAX_BORDER_RADIUS_PERCENT ? 1 : aspectRatio;
	}

	function bounds(mind: Mind) {
		const height = mind.width / displayedAspectRatio(mind.aspectRatio, mind.borderRadius);
		return {
			left: mind.worldX - mind.width / 2,
			right: mind.worldX + mind.width / 2,
			top: mind.worldY - height / 2,
			bottom: mind.worldY + height / 2
		};
	}

	function overlap(a: Mind, b: Mind) {
		const aBounds = bounds(a);
		const bBounds = bounds(b);
		const horizontal =
			Math.min(aBounds.right, bBounds.right) - Math.max(aBounds.left, bBounds.left);
		const vertical = Math.min(aBounds.bottom, bBounds.bottom) - Math.max(aBounds.top, bBounds.top);
		return horizontal > EDGE_OVERLAP && vertical > EDGE_OVERLAP;
	}

	function findOpenPosition(width: number, aspectRatio: number, borderRadius: number) {
		const centerX = canvasViewport
			? canvasViewport.scrollLeft + canvasViewport.clientWidth / 2 - originX
			: 0;
		const centerY = canvasViewport
			? canvasViewport.scrollTop + canvasViewport.clientHeight / 2 - originY
			: 0;
		const height = width / displayedAspectRatio(aspectRatio, borderRadius);
		const minX = -originX + width / 2 + 36;
		const maxX = canvasWidth - originX - width / 2 - 36;
		const minY = -originY + height / 2 + 36;
		const maxY = canvasHeight - originY - height / 2 - 36;

		for (let index = 0; index < 2400; index += 1) {
			const radius = 42 * Math.sqrt(index);
			const angle = index * 2.3999632297;
			const candidate: Mind = {
				id: '__candidate__',
				image: '',
				idea: '',
				worldX: Math.max(minX, Math.min(maxX, centerX + Math.cos(angle) * radius)),
				worldY: Math.max(minY, Math.min(maxY, centerY + Math.sin(angle) * radius)),
				width,
				aspectRatio,
				borderRadius,
				tilt: 0,
				createdAt: 0
			};
			if (!minds.some((mind) => overlap(candidate, mind))) {
				return { worldX: candidate.worldX, worldY: candidate.worldY };
			}
		}

		return {
			worldX: Math.max(minX, Math.min(maxX, centerX)),
			worldY: Math.max(minY, Math.min(maxY, centerY + minds.length * 120))
		};
	}

	function resolveCollisions(entries: Mind[], fixedId?: string) {
		const settled = entries.map((mind) => ({ ...mind }));

		for (let iteration = 0; iteration < 600; iteration += 1) {
			let moved = false;
			for (let first = 0; first < settled.length; first += 1) {
				for (let second = first + 1; second < settled.length; second += 1) {
					const a = settled[first];
					const b = settled[second];
					if (!overlap(a, b)) continue;

					const aBounds = bounds(a);
					const bBounds = bounds(b);
					const pushX =
						Math.min(aBounds.right, bBounds.right) -
						Math.max(aBounds.left, bBounds.left) -
						EDGE_OVERLAP +
						1;
					const pushY =
						Math.min(aBounds.bottom, bBounds.bottom) -
						Math.max(aBounds.top, bBounds.top) -
						EDGE_OVERLAP +
						1;
					const aIsFixed = a.id === fixedId;
					const bIsFixed = b.id === fixedId;
					const aShare = aIsFixed ? 0 : bIsFixed ? 1 : 0.5;
					const bShare = bIsFixed ? 0 : aIsFixed ? 1 : 0.5;

					if (pushX < pushY) {
						const direction = b.worldX >= a.worldX ? 1 : -1;
						a.worldX -= direction * pushX * aShare;
						b.worldX += direction * pushX * bShare;
					} else {
						const direction = b.worldY >= a.worldY ? 1 : -1;
						a.worldY -= direction * pushY * aShare;
						b.worldY += direction * pushY * bShare;
					}
					moved = true;
				}
			}
			if (!moved) break;
		}

		return settled;
	}

	async function saveDocument() {
		if (!draftImage || isSaving) return;
		isSaving = true;
		loadError = '';
		const existing = editingMindId ? minds.find((mind) => mind.id === editingMindId) : undefined;

		if (existing) {
			const updated: Mind = {
				...existing,
				image: draftImage,
				mediaType: draftMediaType,
				idea: draftIdea.trim(),
				aspectRatio: draftAspectRatio,
				borderRadius: draftBorderRadius
			};
			const updatedMinds = resolveCollisions(
				minds.map((mind) => (mind.id === updated.id ? updated : mind)),
				updated.id
			);

			try {
				await storeMinds(updatedMinds);
				minds = updatedMinds;
				selected = updatedMinds.find((mind) => mind.id === updated.id) ?? updated;
				composing = false;
				editingMindId = null;
			} catch {
				loadError = 'This document could not be updated. Please try again.';
			} finally {
				isSaving = false;
			}
			return;
		}

		const width = 190 + ((minds.length * 37) % 110);
		const placement = findOpenPosition(width, draftAspectRatio, draftBorderRadius);
		const mind: Mind = {
			id: crypto.randomUUID(),
			image: draftImage,
			mediaType: draftMediaType,
			idea: draftIdea.trim(),
			...placement,
			width,
			aspectRatio: draftAspectRatio,
			borderRadius: draftBorderRadius,
			tilt: -2.5 + ((minds.length * 17) % 50) / 10,
			createdAt: Date.now()
		};

		try {
			await storeMind(mind);
			minds = [...minds, mind];
			composing = false;
		} catch {
			loadError = 'This document could not be saved. Try a smaller image or video.';
		} finally {
			isSaving = false;
		}
	}

	function rememberAspect(mind: Mind, mediaWidth: number, mediaHeight: number) {
		if (!mediaWidth || !mediaHeight) return;
		const aspectRatio = mediaWidth / mediaHeight;
		if (Math.abs(mind.aspectRatio - aspectRatio) < 0.01) return;

		const updated = minds.map((entry) =>
			entry.id === mind.id ? { ...entry, aspectRatio } : entry
		);
		minds = resolveCollisions(updated, mind.id);
		queueLayoutSave(minds, 'The updated layout could not be saved.');
	}

	function rememberImageAspect(mind: Mind, event: Event) {
		const image = event.currentTarget as HTMLImageElement;
		rememberAspect(mind, image.naturalWidth, image.naturalHeight);
	}

	function startResize(event: PointerEvent, mind: Mind) {
		event.preventDefault();
		event.stopPropagation();
		resizeState = { id: mind.id, startX: event.clientX, startWidth: mind.width };
	}

	function resizeMind(event: PointerEvent) {
		if (!resizeState) return;
		const nextWidth = Math.min(
			MAX_MEDIA_WIDTH,
			Math.max(MIN_MEDIA_WIDTH, resizeState.startWidth + event.clientX - resizeState.startX)
		);
		minds = minds.map((mind) =>
			mind.id === resizeState?.id ? { ...mind, width: nextWidth } : mind
		);
	}

	function finishResize() {
		if (!resizeState) return;
		const fixedId = resizeState.id;
		resizeState = null;
		minds = resolveCollisions(minds, fixedId);
		queueLayoutSave(minds, 'The resized layout could not be saved.');
	}

	async function expandCanvas(direction: 'top' | 'right' | 'bottom' | 'left') {
		const previousScrollLeft = canvasViewport.scrollLeft;
		const previousScrollTop = canvasViewport.scrollTop;

		if (direction === 'left') {
			canvasWidth += CANVAS_EXPANSION;
			originX += CANVAS_EXPANSION;
		} else if (direction === 'right') {
			canvasWidth += CANVAS_EXPANSION;
		} else if (direction === 'top') {
			canvasHeight += CANVAS_EXPANSION;
			originY += CANVAS_EXPANSION;
		} else {
			canvasHeight += CANVAS_EXPANSION;
		}

		await tick();
		await new Promise<void>((resolve) => requestAnimationFrame(() => resolve()));
		canvasViewport.scrollLeft =
			direction === 'right' ? previousScrollLeft + CANVAS_EXPANSION : previousScrollLeft;
		canvasViewport.scrollTop =
			direction === 'bottom' ? previousScrollTop + CANVAS_EXPANSION : previousScrollTop;
		saveCanvas();
	}

	function removableSpace(direction: 'top' | 'right' | 'bottom' | 'left') {
		const margin = 36;
		const mindBounds = minds.map(bounds);
		let added: number;
		let empty = Number.POSITIVE_INFINITY;

		if (direction === 'left') {
			added = originX - BASE_ORIGIN_X;
			if (mindBounds.length) {
				const nearestMind = Math.min(...mindBounds.map((item) => item.left));
				empty = nearestMind - margin + originX;
			}
		} else if (direction === 'right') {
			added = canvasWidth - originX - (BASE_CANVAS_WIDTH - BASE_ORIGIN_X);
			if (mindBounds.length) {
				const nearestMind = Math.max(...mindBounds.map((item) => item.right));
				empty = canvasWidth - originX - nearestMind - margin;
			}
		} else if (direction === 'top') {
			added = originY - baseCanvasHeight / 2;
			if (mindBounds.length) {
				const nearestMind = Math.min(...mindBounds.map((item) => item.top));
				empty = nearestMind - margin + originY;
			}
		} else {
			added = canvasHeight - originY - baseCanvasHeight / 2;
			if (mindBounds.length) {
				const nearestMind = Math.max(...mindBounds.map((item) => item.bottom));
				empty = canvasHeight - originY - nearestMind - margin;
			}
		}

		return Math.max(0, Math.min(CANVAS_EXPANSION, added, empty));
	}

	async function recedeCanvas(direction: 'top' | 'right' | 'bottom' | 'left') {
		const amount = removableSpace(direction);
		if (amount < 1) return;
		const previousScrollLeft = canvasViewport.scrollLeft;
		const previousScrollTop = canvasViewport.scrollTop;

		if (direction === 'left') {
			canvasWidth -= amount;
			originX -= amount;
		} else if (direction === 'right') {
			canvasWidth -= amount;
		} else if (direction === 'top') {
			canvasHeight -= amount;
			originY -= amount;
		} else {
			canvasHeight -= amount;
		}

		await tick();
		await new Promise<void>((resolve) => requestAnimationFrame(() => resolve()));
		canvasViewport.scrollLeft =
			direction === 'right' ? Math.max(0, previousScrollLeft - amount) : previousScrollLeft;
		canvasViewport.scrollTop =
			direction === 'bottom' ? Math.max(0, previousScrollTop - amount) : previousScrollTop;
		saveCanvas();
	}

	function openMind(mind: Mind) {
		selected = mind;
	}

	function closeViewer() {
		selected = null;
	}

	async function openExistingEditor() {
		if (!selected) return;
		const mind = selected;
		editingMindId = mind.id;
		draftImage = mind.image;
		draftMediaType = isVideo(mind) ? 'video' : 'image';
		draftAspectRatio = mind.aspectRatio;
		draftBorderRadius = mind.borderRadius;
		draftIdea = mind.idea;
		loadError = '';
		closeViewer();
		composing = true;
		await tick();
		ideaInput.focus();
		ideaInput.setSelectionRange(draftIdea.length, draftIdea.length);
	}

	async function deleteSelected() {
		if (!selected) return;
		const id = selected.id;
		closeViewer();
		await removeMind(id);
		minds = minds.filter((mind) => mind.id !== id);
	}

	function handleKeys(event: KeyboardEvent) {
		if (event.key === 'Escape') {
			if (selected) closeViewer();
			else if (composing) closeComposer();
		}
	}

	function saveCanvas() {
		if (!canvasViewport) return;
		localStorage.setItem(
			'mindscape-canvas',
			JSON.stringify({
				version: 2,
				width: canvasWidth,
				height: canvasHeight,
				originX,
				originY,
				extraTop: Math.max(0, originY - baseCanvasHeight / 2),
				extraBottom: Math.max(0, canvasHeight - originY - baseCanvasHeight / 2),
				scrollX: canvasViewport.scrollLeft,
				scrollY: canvasViewport.scrollTop
			})
		);
	}

	function fitCanvasHeightToViewport() {
		if (!canvasViewport) return;
		const nextBaseHeight = Math.max(1, canvasViewport.clientHeight);
		const extraTop = Math.max(0, originY - baseCanvasHeight / 2);
		const extraBottom = Math.max(0, canvasHeight - originY - baseCanvasHeight / 2);
		baseCanvasHeight = nextBaseHeight;
		canvasHeight = nextBaseHeight + extraTop + extraBottom;
		originY = nextBaseHeight / 2 + extraTop;
		saveCanvas();
	}

	function rememberCanvasScroll() {
		cancelAnimationFrame(scrollSaveFrame);
		scrollSaveFrame = requestAnimationFrame(saveCanvas);
	}

	onMount(() => {
		baseCanvasHeight = Math.max(1, canvasViewport.clientHeight);
		canvasHeight = baseCanvasHeight;
		originY = baseCanvasHeight / 2;

		try {
			const savedCanvas = JSON.parse(localStorage.getItem('mindscape-canvas') || 'null');
			if (Number.isFinite(savedCanvas?.width) && Number.isFinite(savedCanvas?.originX)) {
				canvasWidth = Math.max(BASE_CANVAS_WIDTH, savedCanvas.width);
				originX = Math.max(BASE_ORIGIN_X, savedCanvas.originX);
			}

			let extraTop = 0;
			let extraBottom = 0;
			if (
				savedCanvas?.version === 2 &&
				Number.isFinite(savedCanvas.extraTop) &&
				Number.isFinite(savedCanvas.extraBottom)
			) {
				extraTop = Math.max(0, savedCanvas.extraTop);
				extraBottom = Math.max(0, savedCanvas.extraBottom);
			} else if (Number.isFinite(savedCanvas?.height) && Number.isFinite(savedCanvas?.originY)) {
				extraTop = Math.max(0, savedCanvas.originY - LEGACY_BASE_ORIGIN_Y);
				extraBottom = Math.max(
					0,
					savedCanvas.height -
						savedCanvas.originY -
						(LEGACY_BASE_CANVAS_HEIGHT - LEGACY_BASE_ORIGIN_Y)
				);
			}
			canvasHeight = baseCanvasHeight + extraTop + extraBottom;
			originY = baseCanvasHeight / 2 + extraTop;
			savedScrollX = Number.isFinite(savedCanvas?.scrollX) ? savedCanvas.scrollX : null;
			savedScrollY = Number.isFinite(savedCanvas?.scrollY) ? savedCanvas.scrollY : null;
		} catch {
			// A malformed canvas value should never prevent the mindscape from loading.
		}

		void tick().then(() => {
			canvasViewport.scrollLeft =
				savedScrollX ?? Math.max(0, originX - canvasViewport.clientWidth / 2);
			canvasViewport.scrollTop =
				savedScrollY ?? Math.max(0, originY - canvasViewport.clientHeight / 2);
		});

		readMinds()
			.then((saved) => {
				minds = resolveCollisions(saved);
				queueLayoutSave(minds, 'The migrated layout could not be saved.');
			})
			.catch(() => (loadError = 'Your browser storage is unavailable.'));
		window.addEventListener('paste', onPaste);
		window.addEventListener('keydown', handleKeys);
		window.addEventListener('pointermove', resizeMind);
		window.addEventListener('pointerup', finishResize);
		window.addEventListener('resize', fitCanvasHeightToViewport);
		window.addEventListener('pagehide', saveCanvas);
		return () => {
			cancelAnimationFrame(scrollSaveFrame);
			if (radiusPreviewTimer) clearTimeout(radiusPreviewTimer);
			saveCanvas();
			window.removeEventListener('paste', onPaste);
			window.removeEventListener('keydown', handleKeys);
			window.removeEventListener('pointermove', resizeMind);
			window.removeEventListener('pointerup', finishResize);
			window.removeEventListener('resize', fitCanvasHeightToViewport);
			window.removeEventListener('pagehide', saveCanvas);
		};
	});
</script>

<svelte:head>
	<title>Mindscape</title>
	<meta
		name="description"
		content="A private, visual space for the ideas drifting through your mind."
	/>
</svelte:head>

<main
	bind:this={canvasViewport}
	class="abyss"
	aria-label="Your mindscape"
	onscroll={rememberCanvasScroll}
>
	<div class="canvas-space" style={`width:${canvasWidth}px; height:${canvasHeight}px`}>
		<div class="depth"></div>

		<div class="mind-world" style={`left:${originX}px; top:${originY}px`}>
			{#each minds as mind, index (mind.id)}
				<div
					class="mind-node"
					role="group"
					aria-label="Mindscape item"
					class:resizing={resizeState?.id === mind.id}
					style={`left:${mind.worldX}px; top:${mind.worldY}px; width:${mind.width}px; --tilt:${mind.tilt}deg; --delay:${-(index % 8) * 1.3}s`}
				>
					<button
						class="floating-mind"
						class:circular={mind.borderRadius >= MAX_BORDER_RADIUS_PERCENT}
						style={`--media-radius:${mind.borderRadius}%`}
						onclick={() => openMind(mind)}
						aria-label="Open idea"
					>
						{#if isVideo(mind)}
							<video
								src={mind.image}
								autoplay
								muted
								loop
								playsinline
								preload="metadata"
								aria-hidden="true"
								onloadedmetadata={(event) =>
									rememberAspect(
										mind,
										event.currentTarget.videoWidth,
										event.currentTarget.videoHeight
									)}
							></video>
						{:else}
							<img
								src={mind.image}
								alt=""
								draggable="false"
								onload={(event) => rememberImageAspect(mind, event)}
							/>
						{/if}
					</button>
					<button
						class="resize-handle"
						onpointerdown={(event) => startResize(event, mind)}
						aria-label="Resize idea"
					>
						<svg viewBox="0 0 16 16" aria-hidden="true">
							<path d="M6.5 13.5h7v-7M9.5 13.5l4-4" />
						</svg>
					</button>
				</div>
			{/each}
		</div>
	</div>

	<button class="add-button" onclick={openComposer} aria-label="Add an idea">
		<svg viewBox="0 0 24 24" aria-hidden="true">
			<path d="M12 5v14M5 12h14" />
		</svg>
	</button>
</main>

<nav class="edge-controls" aria-label="Resize mindscape canvas">
	<div class="edge-zone edge-top">
		<button
			class="edge-expand"
			onclick={() => expandCanvas('top')}
			aria-label="Extend canvas upward"
		>
			<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M5 12h14M14 7l5 5-5 5" /></svg>
		</button>
		<button
			class="edge-recede"
			onclick={() => recedeCanvas('top')}
			disabled={removableSpace('top') < 1}
			aria-label="Recede canvas from the top"
		>
			<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M5 12h14M14 7l5 5-5 5" /></svg>
		</button>
	</div>
	<div class="edge-zone edge-right">
		<button
			class="edge-expand"
			onclick={() => expandCanvas('right')}
			aria-label="Extend canvas right"
		>
			<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M5 12h14M14 7l5 5-5 5" /></svg>
		</button>
		<button
			class="edge-recede"
			onclick={() => recedeCanvas('right')}
			disabled={removableSpace('right') < 1}
			aria-label="Recede canvas from the right"
		>
			<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M5 12h14M14 7l5 5-5 5" /></svg>
		</button>
	</div>
	<div class="edge-zone edge-bottom">
		<button
			class="edge-expand"
			onclick={() => expandCanvas('bottom')}
			aria-label="Extend canvas downward"
		>
			<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M5 12h14M14 7l5 5-5 5" /></svg>
		</button>
		<button
			class="edge-recede"
			onclick={() => recedeCanvas('bottom')}
			disabled={removableSpace('bottom') < 1}
			aria-label="Recede canvas from the bottom"
		>
			<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M5 12h14M14 7l5 5-5 5" /></svg>
		</button>
	</div>
	<div class="edge-zone edge-left">
		<button
			class="edge-expand"
			onclick={() => expandCanvas('left')}
			aria-label="Extend canvas left"
		>
			<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M5 12h14M14 7l5 5-5 5" /></svg>
		</button>
		<button
			class="edge-recede"
			onclick={() => recedeCanvas('left')}
			disabled={removableSpace('left') < 1}
			aria-label="Recede canvas from the left"
		>
			<svg viewBox="0 0 24 24" aria-hidden="true"><path d="M5 12h14M14 7l5 5-5 5" /></svg>
		</button>
	</div>
</nav>

<!-- eslint-disable svelte/no-at-html-tags -->
{#if composing}
	<div class="modal-layer editor-layer" role="presentation">
		<div class="document-editor" role="dialog" aria-modal="true" aria-labelledby="composer-title">
			<header class="document-header">
				<div class="document-heading">
					<h1 id="composer-title">{editingMindId ? 'Edit idea document' : 'New idea document'}</h1>
					<div class="document-media-actions">
						<button class="document-media-button" onclick={() => fileInput.click()}>
							{#if draftImage}
								<span class="media-thumbnail">
									{#if draftMediaType === 'video'}
										<video src={draftImage} muted playsinline preload="metadata"></video>
									{:else}
										<img src={draftImage} alt="" />
									{/if}
								</span>
								<span>{draftMediaType === 'video' ? 'Change video' : 'Change image'}</span>
							{:else}
								<svg viewBox="0 0 24 24" aria-hidden="true">
									<path d="M4 16.5V19h16v-2.5M8 9l4-4 4 4M12 5v10" />
								</svg>
								<span>Add image or video</span>
							{/if}
						</button>
						{#if draftImage}
							<button
								class="remove-attachment"
								onclick={() => {
									draftImage = '';
									hideRadiusPreview();
								}}
								aria-label="Remove attached media"
							>
								<svg viewBox="0 0 24 24" aria-hidden="true">
									<path d="M4 7h16M9 7V4h6v3M7 7l1 13h8l1-13M10 11v5M14 11v5" />
								</svg>
							</button>
						{/if}
					</div>
				</div>
				<div class="document-actions">
					<button class="cancel-button" onclick={closeComposer}>Cancel</button>
					<button class="save-button" onclick={saveDocument} disabled={!draftImage || isSaving}>
						{isSaving ? 'Saving…' : editingMindId ? 'Save changes' : 'Add to mindscape'}
					</button>
				</div>
			</header>

			<input
				bind:this={fileInput}
				class="visually-hidden"
				type="file"
				accept="image/*,video/*"
				onchange={(event) => {
					readMedia(event.currentTarget.files?.[0]);
					event.currentTarget.value = '';
				}}
			/>

			{#if draftImage}
				<div class="media-settings">
					<label class="radius-setting" for="media-border-radius">
						<span>Corner radius</span>
						<input
							id="media-border-radius"
							type="range"
							min="0"
							max={MAX_BORDER_RADIUS_PERCENT}
							step="1"
							value={draftBorderRadius}
							oninput={showRadiusPreview}
							onpointerdown={holdRadiusPreview}
							onpointerup={releaseRadiusPreview}
							onpointercancel={releaseRadiusPreview}
							onblur={releaseRadiusPreview}
						/>
						<output for="media-border-radius">{draftBorderRadius}%</output>
					</label>
				</div>
			{/if}

			<div class="document-workspace">
				<section
					class="markdown-pane"
					aria-label="Markdown editor"
					class:dragging={isDragging}
					ondragover={(event) => {
						event.preventDefault();
						isDragging = true;
					}}
					ondragleave={() => (isDragging = false)}
					ondrop={onDrop}
				>
					<div class="pane-header">
						<span>Markdown</span>
					</div>
					<label for="markdown-source" class="visually-hidden">Markdown source</label>
					<textarea
						id="markdown-source"
						bind:this={ideaInput}
						bind:value={draftIdea}
						spellcheck="true"
						placeholder={MARKDOWN_PLACEHOLDER}
						onkeydown={(event) => {
							if (event.key === 'Enter' && (event.ctrlKey || event.metaKey)) {
								event.preventDefault();
								void saveDocument();
							}
						}}></textarea>
					{#if loadError}<p class="document-error">{loadError}</p>{/if}
				</section>

				<section class="preview-pane" aria-label="Markdown preview">
					<div class="pane-header"><span>Preview</span></div>
					<div class="preview-scroll">
						{#if draftPreview}
							<article class="markdown-body">{@html draftPreview}</article>
						{:else}
							<p class="preview-empty">Your documentation will appear here.</p>
						{/if}
					</div>
				</section>
			</div>

			{#if radiusPreviewVisible && draftImage}
				<div class="radius-preview-layer" aria-hidden="true">
					<div class="radius-preview-wrap">
						<div
							class="radius-preview-media"
							class:circular={draftBorderRadius >= MAX_BORDER_RADIUS_PERCENT}
							style={`--preview-radius:${draftBorderRadius}%; aspect-ratio:${displayedAspectRatio(draftAspectRatio, draftBorderRadius)}`}
						>
							{#if draftMediaType === 'video'}
								<video src={draftImage} autoplay muted loop playsinline></video>
							{:else}
								<img src={draftImage} alt="" />
							{/if}
						</div>
						<output>{draftBorderRadius}% radius</output>
					</div>
				</div>
			{/if}
		</div>
	</div>
{/if}

{#if selected}
	<div
		class="modal-layer viewer-layer"
		role="presentation"
		onclick={(event) => event.currentTarget === event.target && closeViewer()}
	>
		<div class="viewer" role="dialog" aria-modal="true" aria-label="Idea">
			<div class="viewer-image-wrap">
				{#if isVideo(selected)}
					<video
						src={selected.image}
						controls
						autoplay
						muted
						loop
						playsinline
						aria-label="Idea video"
					></video>
				{:else}
					<img src={selected.image} alt="" />
				{/if}
			</div>
			<div class="idea-panel">
				<button class="icon-button viewer-close" onclick={closeViewer} aria-label="Close">
					<svg viewBox="0 0 24 24" aria-hidden="true"><path d="m6 6 12 12M18 6 6 18" /></svg>
				</button>
				{#if selected.idea}
					<article class="idea-document markdown-body">
						{@html renderMarkdown(selected.idea)}
					</article>
				{:else}
					<p class="empty-idea">No documentation attached.</p>
				{/if}
				<div class="idea-panel-actions">
					<button class="edit-button" onclick={openExistingEditor} aria-label="Edit idea document">
						<svg viewBox="0 0 24 24" aria-hidden="true">
							<path d="m4 20 4.2-1 10.9-10.9a2.1 2.1 0 0 0-3-3L5.2 16 4 20ZM14.8 6.4l3 3" />
						</svg>
						Edit document
					</button>
					<button class="delete-button" onclick={deleteSelected} aria-label="Delete this idea">
						<svg viewBox="0 0 24 24" aria-hidden="true">
							<path d="M5 7h14M9 7V4h6v3M8 10v7M12 10v7M16 10v7M6.5 7l.7 13h9.6l.7-13" />
						</svg>
						Remove
					</button>
				</div>
			</div>
		</div>
	</div>
{/if}
<!-- eslint-enable svelte/no-at-html-tags -->
