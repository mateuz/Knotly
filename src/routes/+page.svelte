<script lang="ts">
	import { SvelteMap } from 'svelte/reactivity';
	import { Type, ChevronDown, Table2, MousePointer2, Hand, X, Download, Plus, Minus } from '@lucide/svelte';
	import faviconSvg from '$lib/assets/favicon.svg?raw';

	type Tool = 'select' | 'hand';

	type Row = {
		id: string;
		from: string;
		to: string;
		amount: number;
		comparison: number | null;
	};

	type ComputedNode = {
		name: string;
		col: number;
		x: number;
		y: number;
		height: number;
		color: string;
		total: number;
		comparison: number | null;
	};

	type ComputedLink = {
		id: string;
		from: string;
		to: string;
		amount: number;
		comparison: number | null;
		d: string;
		color: string;
		midX: number;
		midY: number;
		linkHeight: number;
	};

	const FONTS = [
		{ name: 'Fira Sans', family: 'Fira Sans' },
		{ name: 'Inter', family: 'Inter' },
		{ name: 'Nunito', family: 'Nunito' },
		{ name: 'Open Sans', family: 'Open Sans' },
		{ name: 'Poppins', family: 'Poppins' },
		{ name: 'Montserrat', family: 'Montserrat' },
	] as const;

	const COLORS = ['#7c3aed', '#2563eb', '#059669', '#0891b2', '#d97706', '#dc2626', '#ec4899', '#f59e0b'];
	const NODE_W = 20;
	const COL_X0 = 155;
	const COL_GAP = 240;
	const NODE_GAP = 24;
	const MAX_H = 350;

	// ── State ───────────────────────────────────────────────────────────────────
	let activeTool = $state<Tool>('select');
	let zoom = $state(1);
	let panX = $state(0);
	let panY = $state(0);
	let isPanning = $state(false);
	let spaceHeld = $state(false);
	let selectedFont = $state('Fira Sans');
	let fontPickerOpen = $state(false);
	let editorOpen = $state(true);
	let nextId = $state(6);
	let downloadOpen = $state(false);
	let svgEl = $state<SVGSVGElement | null>(null);
	let linkOpacity = $state(0.22);
	let labelBoxOpacity = $state(1);
	let labelFontSize = $state(12);
	let editingNode = $state<string | null>(null);
	let editNodeName = $state('');
	let nodeColors = new SvelteMap<string, string>();

	// Node drag
	let manualPositions = new SvelteMap<string, { x: number; y: number }>();
	let draggingNode = $state<string | null>(null);
	let dragNodeOffsetX = 0;
	let dragNodeOffsetY = 0;
	let hoveredNode = $state<string | null>(null);
	let hoveredHandle = $state<string | null>(null);
	let hoveredRemoveHandle = $state<string | null>(null);
	let connectingFrom = $state<string | null>(null);
	let connectGhostX = $state(0);
	let connectGhostY = $state(0);
	let connectTarget = $state<string | null>(null);

	let rows = $state<Row[]>([
		{ id: '1', from: 'Revenue', to: 'Product', amount: 6000, comparison: 12.5 },
		{ id: '2', from: 'Revenue', to: 'Services', amount: 4000, comparison: -3.2 },
		{ id: '3', from: 'Product', to: 'APAC', amount: 3000, comparison: 8.1 },
		{ id: '4', from: 'Product', to: 'EMEA', amount: 3000, comparison: 15.3 },
		{ id: '5', from: 'Services', to: 'Online', amount: 4000, comparison: -1.8 },
	]);

	// Pan drag (non-reactive, mutation-tracked manually)
	let panDragStartX = 0;
	let panDragStartY = 0;
	let panStartX = 0;
	let panStartY = 0;

	const MIN_ZOOM = 0.05;
	const MAX_ZOOM = 20;
	const GRID_SIZE = 40;

	// ── Derived ─────────────────────────────────────────────────────────────────
	let fontFamily = $derived(`'${selectedFont}', sans-serif`);
	let cursorClass = $derived(
		connectingFrom !== null || hoveredHandle !== null
			? 'cursor-crosshair'
			: hoveredRemoveHandle !== null
				? 'cursor-pointer'
				: activeTool === 'hand' || spaceHeld
				? isPanning ? 'cursor-grabbing' : 'cursor-grab'
				: draggingNode !== null
					? 'cursor-grabbing'
					: hoveredNode !== null
						? 'cursor-move'
						: 'cursor-default'
	);
	let gridPx = $derived(GRID_SIZE * zoom);
	let gridOffsetX = $derived(((panX % gridPx) + gridPx) % gridPx - gridPx / 2);
	let gridOffsetY = $derived(((panY % gridPx) + gridPx) % gridPx - gridPx / 2);
	let showGrid = $derived(zoom > 0.15);
	let layout = $derived(computeSankey(rows, manualPositions, nodeColors));
	let editNodeLayout = $derived(editingNode ? (layout.nodes.find(n => n.name === editingNode) ?? null) : null);
	let editNodeRows = $derived(editingNode ? rows.filter(r => r.from === editingNode || r.to === editingNode) : []);
	let editOverlayPos = $derived((() => {
		const node = editNodeLayout;
		if (!node) return null;
		const lineH = Math.round(labelFontSize * 1.25);
		const firstY = Math.round(labelFontSize * 1.33);
		const bh = node.comparison !== null ? firstY + lineH * 2 + 12 : firstY + lineH + 13;
		const bw = Math.max(Math.round(labelFontSize * 8.33), node.name.length * Math.round(labelFontSize * 0.583) + 20);
		const bx = node.col === 0 ? node.x - 5 - bw : node.x + NODE_W + 5;
		const by = node.y + node.height / 2 - bh / 2;
		return { x: bx * zoom + panX, y: by * zoom + panY };
	})());

	// ── Sankey layout ───────────────────────────────────────────────────────────
	function computeSankey(
		rows: Row[],
		overrides: SvelteMap<string, { x: number; y: number }>,
		colorOverrides: SvelteMap<string, string>
	): { nodes: ComputedNode[]; links: ComputedLink[] } {
		const valid = rows.filter(
			(r) => r.from.trim() && r.to.trim() && !isNaN(r.amount) && r.amount > 0
		);
		if (valid.length === 0) return { nodes: [], links: [] };

		const nodeData = new SvelteMap<string, { totalIn: number; totalOut: number; col: number }>();
		for (const r of valid) {
			if (!nodeData.has(r.from)) nodeData.set(r.from, { totalIn: 0, totalOut: 0, col: -1 });
			if (!nodeData.has(r.to)) nodeData.set(r.to, { totalIn: 0, totalOut: 0, col: -1 });
			nodeData.get(r.from)!.totalOut += r.amount;
			nodeData.get(r.to)!.totalIn += r.amount;
		}

		// BFS column assignment
		const hasIncoming = new Set(valid.map((r) => r.to));
		const sources = [...nodeData.keys()].filter((n) => !hasIncoming.has(n));
		const queue: Array<{ name: string; col: number }> = sources.map((n) => ({ name: n, col: 0 }));
		for (const s of sources) nodeData.get(s)!.col = 0;

		while (queue.length > 0) {
			const { name, col } = queue.shift()!;
			for (const r of valid) {
				if (r.from !== name) continue;
				const nd = nodeData.get(r.to)!;
				if (nd.col <= col) {
					nd.col = col + 1;
					queue.push({ name: r.to, col: col + 1 });
				}
			}
		}
		for (const [, nd] of nodeData) if (nd.col === -1) nd.col = 0;

		const maxFlow = Math.max(...[...nodeData.values()].map((nd) => Math.max(nd.totalIn, nd.totalOut)));
		const scale = MAX_H / maxFlow;

		const colGroups = new SvelteMap<number, string[]>();
		for (const [name, nd] of nodeData) {
			if (!colGroups.has(nd.col)) colGroups.set(nd.col, []);
			colGroups.get(nd.col)!.push(name);
		}

		const allNames = [...nodeData.keys()];

		// Weighted-average comparison per node (outgoing flows first, fallback to incoming)
		const nodeComparisons = new Map<string, number | null>();
		for (const [name] of nodeData) {
			const out = valid.filter((r) => r.from === name && r.comparison != null);
			const inp = valid.filter((r) => r.to === name && r.comparison != null);
			const src = out.length > 0 ? out : inp;
			if (src.length === 0) { nodeComparisons.set(name, null); continue; }
			const sumAmt = src.reduce((s, r) => s + r.amount, 0);
			nodeComparisons.set(name, src.reduce((s, r) => s + r.comparison! * r.amount, 0) / sumAmt);
		}

		const positions = new SvelteMap<string, ComputedNode>();

		for (const [col, names] of colGroups) {
			names.sort();
			const defaultX = COL_X0 + col * COL_GAP;
			const totalH =
				names.reduce(
					(s, n) => s + Math.max(nodeData.get(n)!.totalIn, nodeData.get(n)!.totalOut) * scale,
					0
				) +
				(names.length - 1) * NODE_GAP;
			let computedY = Math.max(40, 200 - totalH / 2);

			for (const name of names) {
				const nd = nodeData.get(name)!;
				const height = Math.max(nd.totalIn, nd.totalOut) * scale;
				const color = colorOverrides.get(name) ?? COLORS[allNames.indexOf(name) % COLORS.length];
				const override = overrides.get(name);
				positions.set(name, {
					name,
					col,
					x: override ? override.x : defaultX,
					y: override ? override.y : computedY,
					height,
					color,
					total: Math.max(nd.totalIn, nd.totalOut),
					comparison: nodeComparisons.get(name) ?? null,
				});
				computedY += height + NODE_GAP;
			}
		}

		// Link paths with vertical stacking at each node
		const usedOut = new SvelteMap<string, number>();
		const usedIn = new SvelteMap<string, number>();
		for (const name of nodeData.keys()) { usedOut.set(name, 0); usedIn.set(name, 0); }

		const links: ComputedLink[] = [];
		for (const r of valid) {
			const fp = positions.get(r.from);
			const tp = positions.get(r.to);
			if (!fp || !tp) continue;

			const lh = r.amount * scale;
			const fo = usedOut.get(r.from)!;
			const ti = usedIn.get(r.to)!;

			const x1 = fp.x + NODE_W;
			const y1t = fp.y + fo;
			const y1b = y1t + lh;
			const x2 = tp.x;
			const y2t = tp.y + ti;
			const y2b = y2t + lh;
			const mx = (x1 + x2) / 2;

			const d = `M${x1} ${y1t} C${mx} ${y1t} ${mx} ${y2t} ${x2} ${y2t} L${x2} ${y2b} C${mx} ${y2b} ${mx} ${y1b} ${x1} ${y1b} Z`;
			const cmp = r.comparison == null || isNaN(r.comparison) ? null : r.comparison;

			links.push({
				id: r.id,
				from: r.from,
				to: r.to,
				amount: r.amount,
				comparison: cmp,
				d,
				color: fp.color,
				midX: mx,
				midY: (y1t + y1b + y2t + y2b) / 4,
				linkHeight: lh,
			});

			usedOut.set(r.from, fo + lh);
			usedIn.set(r.to, ti + lh);
		}

		return { nodes: [...positions.values()], links };
	}

	// ── Hit testing ─────────────────────────────────────────────────────────────
	// Hit area includes the node bar + approximate label region for easy grabbing
	function hitTestNode(cx: number, cy: number): ComputedNode | null {
		const LABEL_W = 130;
		const PAD = 5;
		for (const node of layout.nodes) {
			const xMin = node.col === 0 ? node.x - LABEL_W : node.x - PAD;
			const xMax = node.col === 0 ? node.x + NODE_W + PAD : node.x + NODE_W + LABEL_W;
			if (cx >= xMin && cx <= xMax && cy >= node.y - PAD && cy <= node.y + node.height + PAD) {
				return node;
			}
		}
		return null;
	}

	function hitTestHandle(cx: number, cy: number): string | null {
		for (const node of layout.nodes) {
			const hx = node.x + NODE_W / 2;
			const hy = node.y - 10;
			if (Math.hypot(cx - hx, cy - hy) <= 9) return node.name;
		}
		return null;
	}

	function hitTestRemoveHandle(cx: number, cy: number): string | null {
		for (const node of layout.nodes) {
			const hx = node.x + NODE_W / 2;
			const hy = node.y + node.height + 10;
			if (Math.hypot(cx - hx, cy - hy) <= 9) return node.name;
		}
		return null;
	}

	function removeNode(name: string) {
		for (let i = rows.length - 1; i >= 0; i--) {
			if (rows[i].from === name || rows[i].to === name) rows.splice(i, 1);
		}
		manualPositions.delete(name);
		nodeColors.delete(name);
		if (editingNode === name) editingNode = null;
	}

	// Convert screen coords → canvas (world) coords
	function toCanvas(clientX: number, clientY: number) {
		return { x: (clientX - panX) / zoom, y: (clientY - panY) / zoom };
	}

	// ── Canvas interaction ───────────────────────────────────────────────────────
	function handleWheel(e: WheelEvent) {
		e.preventDefault();
		if (e.ctrlKey || e.metaKey) {
			const rect = (e.currentTarget as SVGSVGElement).getBoundingClientRect();
			const mouseX = e.clientX - rect.left;
			const mouseY = e.clientY - rect.top;
			const factor = e.deltaY > 0 ? 0.9 : 1 / 0.9;
			const newZoom = Math.max(MIN_ZOOM, Math.min(MAX_ZOOM, zoom * factor));
			panX = mouseX - (mouseX - panX) * (newZoom / zoom);
			panY = mouseY - (mouseY - panY) * (newZoom / zoom);
			zoom = newZoom;
		} else {
			panX -= e.deltaX;
			panY -= e.deltaY;
		}
	}

	function onDblClick(e: MouseEvent) {
		const { x: cx, y: cy } = toCanvas(e.clientX, e.clientY);
		const hit = hitTestNode(cx, cy);
		if (hit) startEditNode(hit.name);
	}

	function onPointerDown(e: PointerEvent) {
		if (editingNode) return;
		const svg = e.currentTarget as SVGSVGElement;

		if (activeTool === 'select' && e.button === 0) {
			const { x: cx, y: cy } = toCanvas(e.clientX, e.clientY);

			// Remove handle — delete node
			const removeHandle = hitTestRemoveHandle(cx, cy);
			if (removeHandle) {
				removeNode(removeHandle);
				e.preventDefault();
				return;
			}

			// Connection handle — start drawing a connection
			const handle = hitTestHandle(cx, cy);
			if (handle) {
				connectingFrom = handle;
				connectGhostX = cx;
				connectGhostY = cy;
				connectTarget = null;
				svg.setPointerCapture(e.pointerId);
				e.preventDefault();
				return;
			}

			// Node drag
			const hit = hitTestNode(cx, cy);
			if (hit) {
				draggingNode = hit.name;
				dragNodeOffsetX = cx - hit.x;
				dragNodeOffsetY = cy - hit.y;
				svg.setPointerCapture(e.pointerId);
				e.preventDefault();
				return;
			}
		}

		// Canvas pan
		if (e.button === 1 || activeTool === 'hand' || spaceHeld) {
			isPanning = true;
			panDragStartX = e.clientX;
			panDragStartY = e.clientY;
			panStartX = panX;
			panStartY = panY;
			svg.setPointerCapture(e.pointerId);
			e.preventDefault();
		}
	}

	function onPointerMove(e: PointerEvent) {
		const { x: cx, y: cy } = toCanvas(e.clientX, e.clientY);

		if (connectingFrom !== null) {
			connectGhostX = cx;
			connectGhostY = cy;
			const hit = hitTestNode(cx, cy);
			connectTarget = hit && hit.name !== connectingFrom ? hit.name : null;
			return;
		}

		if (draggingNode !== null) {
			manualPositions.set(draggingNode, {
				x: cx - dragNodeOffsetX,
				y: cy - dragNodeOffsetY,
			});
			return;
		}

		if (isPanning) {
			panX = panStartX + (e.clientX - panDragStartX);
			panY = panStartY + (e.clientY - panDragStartY);
			return;
		}

		if (activeTool === 'select') {
			hoveredNode = hitTestNode(cx, cy)?.name ?? null;
			hoveredHandle = hitTestHandle(cx, cy);
			hoveredRemoveHandle = hitTestRemoveHandle(cx, cy);
		}
	}

	function onPointerUp() {
		if (connectingFrom !== null) {
			const from = connectingFrom;
			const target = connectTarget;
			const gx = connectGhostX;
			const gy = connectGhostY;
			connectingFrom = null;
			connectTarget = null;

			if (target) {
				rows.push({ id: String(nextId++), from, to: target, amount: 1000, comparison: null });
				startEditNode(target);
			} else {
				const newName = generateNodeName();
				manualPositions.set(newName, { x: gx - NODE_W / 2, y: gy - 20 });
				rows.push({ id: String(nextId++), from, to: newName, amount: 1000, comparison: null });
				startEditNode(newName);
			}
			return;
		}
		draggingNode = null;
		isPanning = false;
	}

	function onPointerLeave() {
		if (!draggingNode) { hoveredNode = null; hoveredHandle = null; hoveredRemoveHandle = null; }
	}

	function onKeyDown(e: KeyboardEvent) {
		if ((e.target as HTMLElement).tagName === 'INPUT') return;
		if (e.code === 'Space' && !e.repeat) { spaceHeld = true; e.preventDefault(); }
		if (e.key === 'h') activeTool = 'hand';
		if (e.key === 'v' || e.key === 'Escape') { activeTool = 'select'; connectingFrom = null; }
		if ((e.ctrlKey || e.metaKey) && e.key === '0') centerDiagram();
	}

	function onKeyUp(e: KeyboardEvent) {
		if (e.code === 'Space') spaceHeld = false;
	}

	function zoomBy(factor: number) {
		const cx = window.innerWidth / 2;
		const cy = window.innerHeight / 2;
		const newZoom = Math.max(MIN_ZOOM, Math.min(MAX_ZOOM, zoom * factor));
		panX = cx - (cx - panX) * (newZoom / zoom);
		panY = cy - (cy - panY) * (newZoom / zoom);
		zoom = newZoom;
	}

	function centerDiagram() {
		if (layout.nodes.length === 0) return;
		const PADDING = 80;
		const HEADER_H = 48;
		const EDITOR_W = editorOpen ? 320 : 0;
		const BOX_GAP = 5;

		let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
		for (const node of layout.nodes) {
			const lblLineH = Math.round(labelFontSize * 1.25);
			const lblFirstY = Math.round(labelFontSize * 1.33);
			const bw = Math.max(Math.round(labelFontSize * 8.33), node.name.length * Math.round(labelFontSize * 0.583) + 20);
			const bh = node.comparison !== null ? lblFirstY + lblLineH * 2 + 12 : lblFirstY + lblLineH + 13;
			const lx = node.col === 0 ? node.x - BOX_GAP - bw : node.x;
			const rx = node.col === 0 ? node.x + NODE_W : node.x + NODE_W + BOX_GAP + bw;
			const ty = node.y + node.height / 2 - bh / 2;
			const by = node.y + node.height / 2 + bh / 2;
			minX = Math.min(minX, lx);
			maxX = Math.max(maxX, rx);
			minY = Math.min(minY, Math.min(node.y, ty));
			maxY = Math.max(maxY, Math.max(node.y + node.height, by));
		}

		const dw = maxX - minX;
		const dh = maxY - minY;
		const cx = (minX + maxX) / 2;
		const cy = (minY + maxY) / 2;

		const availW = window.innerWidth - EDITOR_W - PADDING * 2;
		const availH = window.innerHeight - HEADER_H - PADDING * 2;
		zoom = Math.min(availW / dw, availH / dh, 2.5);

		const screenCX = (window.innerWidth - EDITOR_W) / 2;
		const screenCY = HEADER_H + (window.innerHeight - HEADER_H) / 2;
		panX = screenCX - cx * zoom;
		panY = screenCY - cy * zoom;
	}

	function resetView() { centerDiagram(); }

	let _centered = false;
	$effect(() => {
		if (!_centered && layout.nodes.length > 0) {
			_centered = true;
			centerDiagram();
		}
	});

	// ── Row operations ───────────────────────────────────────────────────────────
	function addRow() {
		rows.push({ id: String(nextId++), from: '', to: '', amount: 0, comparison: null });
	}

	function deleteRow(id: string) {
		const idx = rows.findIndex((r) => r.id === id);
		if (idx !== -1) rows.splice(idx, 1);
	}

	function generateNodeName(): string {
		const existing = new Set(layout.nodes.map(n => n.name));
		let i = 1;
		while (existing.has(`Node ${i}`)) i++;
		return `Node ${i}`;
	}

	function startEditNode(name: string) {
		confirmEditNode();
		editingNode = name;
		editNodeName = name;
	}

	function confirmEditNode() {
		if (!editingNode) return;
		const original = editingNode;
		editingNode = null;
		const trimmed = editNodeName.trim();
		if (!trimmed || trimmed === original) return;
		for (const row of rows) {
			if (row.from === original) row.from = trimmed;
			if (row.to === original) row.to = trimmed;
		}
		if (nodeColors.has(original)) {
			nodeColors.set(trimmed, nodeColors.get(original)!);
			nodeColors.delete(original);
		}
		if (manualPositions.has(original)) {
			manualPositions.set(trimmed, manualPositions.get(original)!);
			manualPositions.delete(original);
		}
	}

	function fmtAmount(v: number): string {
		if (v >= 1_000_000) return `$${(v / 1_000_000).toFixed(1)}M`;
		if (v >= 1_000) return `$ ${(v / 1_000).toFixed(1)}K`;
		return `$${v.toFixed(0)}`;
	}

	function fmtNodeCmp(v: number): string {
		const sign = v > 0 ? '+' : '';
		return `${sign}${v.toFixed(1)}% Y/Y`;
	}

	// ── Export ───────────────────────────────────────────────────────────────────
	function getExportBBox(padding = 40): { x: number; y: number; w: number; h: number } | null {
		if (layout.nodes.length === 0) return null;
		const LABEL_PAD = 100;
		let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
		for (const node of layout.nodes) {
			minX = Math.min(minX, node.x - (node.col === 0 ? LABEL_PAD : 0));
			maxX = Math.max(maxX, node.x + NODE_W + (node.col !== 0 ? LABEL_PAD : 0));
			minY = Math.min(minY, node.y);
			maxY = Math.max(maxY, node.y + node.height);
		}
		return { x: minX - padding, y: minY - padding, w: maxX - minX + padding * 2, h: maxY - minY + padding * 2 };
	}

	function buildExportSvg(): string | null {
		const bbox = getExportBBox();
		if (!bbox || !svgEl) return null;
		const { x, y, w, h } = bbox;

		const mainG = svgEl.querySelector(':scope > g') as SVGGElement | null;
		if (!mainG) return null;
		const clonedG = mainG.cloneNode(true) as SVGGElement;
		clonedG.removeAttribute('transform');

		const filterEl = svgEl.querySelector('filter');
		const defsStr = filterEl
			? `<defs>${new XMLSerializer().serializeToString(filterEl)}</defs>`
			: '';

		return [
			`<svg xmlns="http://www.w3.org/2000/svg" width="${w}" height="${h}" viewBox="${x} ${y} ${w} ${h}">`,
			defsStr,
			`<rect x="${x}" y="${y}" width="${w}" height="${h}" fill="white"/>`,
			new XMLSerializer().serializeToString(clonedG),
			`</svg>`,
		].join('\n');
	}

	function downloadSVG() {
		const svgStr = buildExportSvg();
		if (!svgStr) return;
		const blob = new Blob([svgStr], { type: 'image/svg+xml' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = 'sankey-diagram.svg';
		a.click();
		URL.revokeObjectURL(url);
		downloadOpen = false;
	}

	async function downloadPNG() {
		const svgStr = buildExportSvg();
		const bbox = getExportBBox();
		if (!svgStr || !bbox) return;
		downloadOpen = false;

		const scale = 2;
		const canvas = document.createElement('canvas');
		canvas.width = Math.round(bbox.w * scale);
		canvas.height = Math.round(bbox.h * scale);
		const ctx = canvas.getContext('2d')!;
		ctx.fillStyle = 'white';
		ctx.fillRect(0, 0, canvas.width, canvas.height);
		ctx.scale(scale, scale);

		const blob = new Blob([svgStr], { type: 'image/svg+xml' });
		const url = URL.createObjectURL(blob);
		await new Promise<void>((resolve) => {
			const img = new Image();
			img.onload = () => { ctx.drawImage(img, 0, 0); resolve(); };
			img.src = url;
		});
		URL.revokeObjectURL(url);

		const a = document.createElement('a');
		a.href = canvas.toDataURL('image/png');
		a.download = 'sankey-diagram.png';
		a.click();
	}
</script>

<svelte:window onkeydown={onKeyDown} onkeyup={onKeyUp} />

<div class="w-screen h-screen overflow-hidden bg-slate-50 select-none relative">

	<!-- ── Header ── -->
	<header class="absolute inset-x-0 top-0 h-12 bg-white border-b border-slate-200 flex items-center px-4 z-20 gap-3">
		<div class="flex items-center gap-2 shrink-0">
			<div style="width:22px;height:22px;flex-shrink:0;" class="[&>svg]:size-full">{@html faviconSvg}</div>
			<span class="font-semibold text-slate-800 text-sm tracking-tight">Sankey Editor</span>
		</div>

		<div class="w-px h-5 bg-slate-200 shrink-0"></div>

		<!-- Font picker -->
		<div class="relative shrink-0">
			<button
				onclick={() => (fontPickerOpen = !fontPickerOpen)}
				class="flex items-center gap-1.5 px-3 py-1.5 rounded-lg border border-slate-200 text-sm text-slate-700 hover:bg-slate-50 transition-colors bg-white"
				style="font-family: {fontFamily};"
			>
				<Type size={13} class="text-slate-400 shrink-0" />
				<span>{selectedFont}</span>
				<ChevronDown size={10} class="text-slate-400 shrink-0" />
			</button>

			{#if fontPickerOpen}
				<button class="fixed inset-0 z-20 cursor-default" onclick={() => (fontPickerOpen = false)} aria-label="Close font picker" tabindex="-1"></button>
				<div class="absolute top-full left-0 mt-1 bg-white rounded-xl shadow-lg border border-slate-200 py-1 z-30 w-44 overflow-hidden">
					{#each FONTS as font (font.family)}
						<button
							class="w-full px-4 py-2.5 text-left text-sm transition-colors {selectedFont === font.family ? 'text-violet-700 bg-violet-50' : 'text-slate-700 hover:bg-slate-50'}"
							style="font-family: '{font.family}', sans-serif;"
							onclick={() => { selectedFont = font.family; fontPickerOpen = false; }}
						>{font.name}</button>
					{/each}
				</div>
			{/if}
		</div>

		<div class="flex-1"></div>

		<!-- Download -->
		<div class="relative shrink-0">
			<button
				onclick={() => (downloadOpen = !downloadOpen)}
				class="flex items-center gap-1.5 px-3 py-1.5 rounded-lg border border-slate-200 bg-white text-sm text-slate-600 hover:bg-slate-50 transition-colors"
				title="Download diagram"
			>
				<Download size={14} />
				<span>Export</span>
				<ChevronDown size={10} class="text-slate-400" />
			</button>

			{#if downloadOpen}
				<button class="fixed inset-0 z-20 cursor-default" onclick={() => (downloadOpen = false)} aria-label="Close download menu" tabindex="-1"></button>
				<div class="absolute top-full right-0 mt-1 bg-white rounded-xl shadow-lg border border-slate-200 py-1 z-30 w-36 overflow-hidden">
					<button
						onclick={downloadSVG}
						class="w-full px-4 py-2.5 text-left text-sm text-slate-700 hover:bg-slate-50 transition-colors"
					>Download SVG</button>
					<button
						onclick={downloadPNG}
						class="w-full px-4 py-2.5 text-left text-sm text-slate-700 hover:bg-slate-50 transition-colors"
					>Download PNG</button>
				</div>
			{/if}
		</div>

		<!-- Editor toggle -->
		<button
			onclick={() => (editorOpen = !editorOpen)}
			class="flex items-center gap-1.5 px-3 py-1.5 rounded-lg border text-sm transition-colors {editorOpen ? 'border-violet-300 bg-violet-50 text-violet-700' : 'border-slate-200 bg-white text-slate-600 hover:bg-slate-50'}"
			title="Toggle data editor"
		>
			<Table2 size={14} />
			<span>Data</span>
		</button>
	</header>

	<!-- ── Left toolbar ── -->
	<aside class="absolute left-3 top-1/2 -translate-y-1/2 z-10 flex flex-col gap-1 bg-white rounded-xl shadow-md border border-slate-200 p-1.5">
		<button
			class="w-9 h-9 flex items-center justify-center rounded-lg transition-colors {activeTool === 'select' ? 'bg-violet-100 text-violet-700' : 'text-slate-500 hover:bg-slate-100'}"
			onclick={() => (activeTool = 'select')}
			title="Select / Move (V)"
		>
			<MousePointer2 size={15} />
		</button>
		<button
			class="w-9 h-9 flex items-center justify-center rounded-lg transition-colors {activeTool === 'hand' ? 'bg-violet-100 text-violet-700' : 'text-slate-500 hover:bg-slate-100'}"
			onclick={() => (activeTool = 'hand')}
			title="Hand / Pan (H)"
		>
			<Hand size={17} />
		</button>
	</aside>

	<!-- ── Canvas SVG ── -->
	<svg
		bind:this={svgEl}
		{@attach (el: SVGSVGElement) => {
			el.addEventListener('wheel', handleWheel, { passive: false });
			return () => el.removeEventListener('wheel', handleWheel);
		}}
		role="application"
		aria-label="Sankey diagram canvas"
		class="absolute inset-0 w-full h-full {cursorClass}"
		onpointerdown={onPointerDown}
		onpointermove={onPointerMove}
		onpointerup={onPointerUp}
		onpointerleave={onPointerLeave}
		ondblclick={onDblClick}
		oncontextmenu={(e) => e.preventDefault()}
	>
		<defs>
			{#if showGrid}
				<pattern id="dot-grid" x={gridOffsetX} y={gridOffsetY} width={gridPx} height={gridPx} patternUnits="userSpaceOnUse">
					<circle cx={gridPx / 2} cy={gridPx / 2} r="1.5" fill="#94a3b8" opacity="0.5" />
				</pattern>
			{/if}
			<filter id="node-lift" x="-40%" y="-40%" width="180%" height="180%">
				<feDropShadow dx="0" dy="5" stdDeviation="7" flood-color="#000" flood-opacity="0.22" />
			</filter>
			<filter id="label-shadow" x="-20%" y="-30%" width="140%" height="160%">
				<feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#000" flood-opacity="0.08" />
			</filter>
		</defs>

		<rect width="100%" height="100%" fill="#f8fafc" />
		{#if showGrid}
			<rect width="100%" height="100%" fill="url(#dot-grid)" />
		{/if}

		<g transform="translate({panX},{panY}) scale({zoom})" style="font-family: {fontFamily};">

			<!-- ── Links ── -->
			{#each layout.links as link (link.id)}
				<path d={link.d} fill={link.color} opacity={linkOpacity} />
			{/each}

			<!-- ── Nodes ── -->
			{#each layout.nodes as node (node.name)}
				{@const isDragging = draggingNode === node.name}
				{@const isHovered = hoveredNode === node.name && !isDragging}
				{@const LBL_LINE_H = Math.round(labelFontSize * 1.25)}
				{@const LBL_FIRST_Y = Math.round(labelFontSize * 1.33)}
				{@const BOX_H = node.comparison !== null ? LBL_FIRST_Y + LBL_LINE_H * 2 + 12 : LBL_FIRST_Y + LBL_LINE_H + 13}
				{@const BOX_W = Math.max(Math.round(labelFontSize * 8.33), node.name.length * Math.round(labelFontSize * 0.583) + 20)}
				{@const BOX_GAP = 5}
				{@const bx = node.col === 0 ? node.x - BOX_GAP - BOX_W : node.x + NODE_W + BOX_GAP}
				{@const by = node.y + node.height / 2 - BOX_H / 2}

				<!-- Node bar -->
				<rect
					x={node.x}
					y={node.y}
					width={NODE_W}
					height={node.height}
					rx="3"
					fill={node.color}
					filter={isDragging ? 'url(#node-lift)' : ''}
					stroke={connectTarget === node.name ? 'white' : isHovered ? 'white' : 'none'}
					stroke-width={connectTarget === node.name ? 3 : 2.5}
					stroke-opacity="0.7"
				/>

				<!-- Connection handle (top) -->
				{#if (hoveredNode === node.name || hoveredHandle === node.name) && !connectingFrom && activeTool === 'select'}
					<g transform="translate({node.x + NODE_W / 2 - 7}, {node.y - 18})" style="pointer-events:none;" opacity="0.75">
						<Plus size={14} color={node.color} />
					</g>
				{/if}

				<!-- Remove handle (bottom) -->
				{#if (hoveredNode === node.name || hoveredRemoveHandle === node.name) && !connectingFrom && activeTool === 'select'}
					<g transform="translate({node.x + NODE_W / 2 - 7}, {node.y + node.height + 4})" style="pointer-events:none;" opacity="0.75">
						<Minus size={14} color={node.color} />
					</g>
				{/if}

				<!-- Label box -->
				<g>
					<rect
						x={bx} y={by}
						width={BOX_W} height={BOX_H}
						rx="6"
						fill="white"
						opacity={isDragging ? labelBoxOpacity * 0.5 : labelBoxOpacity}
						stroke={node.color}
						stroke-width="0.75"
						stroke-opacity="0.35"
						filter="url(#label-shadow)"
					/>
					<text x={bx + 10} y={by + LBL_FIRST_Y} font-size={labelFontSize} font-weight="700" fill="#1e293b">{node.name}</text>
					<text x={bx + 10} y={by + LBL_FIRST_Y + LBL_LINE_H} font-size={labelFontSize - 1} fill="#64748b">{fmtAmount(node.total)}</text>
					{#if node.comparison !== null}
						<text
							x={bx + 10} y={by + LBL_FIRST_Y + LBL_LINE_H * 2}
							font-size={labelFontSize - 1}
							fill={node.comparison >= 0 ? '#16a34a' : '#dc2626'}
						>{fmtNodeCmp(node.comparison)}</text>
					{/if}
				</g>
			{/each}

			<!-- ── Connection ghost ── -->
			{#if connectingFrom}
				{@const src = layout.nodes.find(n => n.name === connectingFrom)}
				{#if src}
					{@const x1 = src.x + NODE_W / 2}
					{@const y1 = src.y - 10}
					{@const mx = (x1 + connectGhostX) / 2}
					<path
						d="M{x1} {y1} C{mx} {y1} {mx} {connectGhostY} {connectGhostX} {connectGhostY}"
						fill="none"
						stroke={src.color}
						stroke-width="2"
						stroke-dasharray="6 4"
						opacity="0.75"
					/>
					{#if !connectTarget}
						<g transform="translate({connectGhostX - 7}, {connectGhostY - 7})" style="pointer-events:none;" opacity="0.85">
							<Plus size={14} color={src.color} />
						</g>
					{/if}
				{/if}
			{/if}

		</g>
	</svg>

	<!-- ── Node edit overlay ── -->
	{#if editingNode && editOverlayPos}
		<button class="fixed inset-0 z-20 cursor-default" onclick={confirmEditNode} tabindex="-1" aria-label="Close node editor"></button>
		<div
			role="dialog"
			aria-label="Edit node"
			tabindex="-1"
			class="absolute z-30 bg-white rounded-xl shadow-xl border border-violet-200 p-3 flex flex-col gap-2"
			style="left:{editOverlayPos.x}px; top:{editOverlayPos.y}px; min-width:220px;"
			onpointerdown={(e) => e.stopPropagation()}
		>
			<input
				type="text"
				bind:value={editNodeName}
				class="text-sm font-bold text-slate-800 px-2 py-1.5 rounded-lg border border-violet-300 outline-none focus:ring-2 focus:ring-violet-200 w-full"
				placeholder="Node name"
				onkeydown={(e) => { e.stopPropagation(); if (e.key === 'Enter') confirmEditNode(); if (e.key === 'Escape') editingNode = null; }}
				{@attach (el: HTMLInputElement) => { el.focus(); el.select(); }}
			/>
			{#if editNodeRows.length > 0}
				<div class="flex flex-col gap-1.5 pt-1 border-t border-slate-100">
					<div class="grid gap-1 text-xs text-slate-400 font-medium pt-1" style="grid-template-columns: 1fr 72px 52px;">
						<span>Flow</span><span>Amount</span><span>vs %</span>
					</div>
					{#each editNodeRows as row (row.id)}
						<div class="grid gap-1 items-center" style="grid-template-columns: 1fr 72px 52px;">
							<span class="text-xs text-slate-500 truncate">
								{row.from === editingNode ? `→ ${row.to}` : `← ${row.from}`}
							</span>
							<input
								type="number"
								bind:value={row.amount}
								min="0"
								class="text-xs px-1.5 py-1 rounded border border-slate-200 outline-none focus:border-violet-300 w-full"
								onkeydown={(e) => { e.stopPropagation(); if (e.key === 'Enter') confirmEditNode(); if (e.key === 'Escape') editingNode = null; }}
							/>
							<input
								type="number"
								value={row.comparison ?? ''}
								placeholder="—"
								class="text-xs px-1.5 py-1 rounded border border-slate-200 outline-none focus:border-violet-300 w-full"
								oninput={(e) => { const v = (e.currentTarget as HTMLInputElement).value; row.comparison = v === '' ? null : parseFloat(v); }}
								onkeydown={(e) => { e.stopPropagation(); if (e.key === 'Enter') confirmEditNode(); if (e.key === 'Escape') editingNode = null; }}
							/>
						</div>
					{/each}
				</div>
			{/if}
			<button
				onclick={confirmEditNode}
				class="mt-0.5 py-1.5 rounded-lg bg-violet-600 text-white text-xs font-medium hover:bg-violet-700 transition-colors"
			>Done</button>
		</div>
	{/if}

	<!-- ── Data editor panel ── -->
	{#if editorOpen}
		<div class="absolute right-0 top-12 bottom-0 w-80 bg-white border-l border-slate-200 flex flex-col z-10 shadow-xl">
			<div class="flex items-center px-4 py-3 border-b border-slate-100 shrink-0">
				<span class="text-sm font-semibold text-slate-700">Data</span>
				<span class="ml-2 text-xs text-slate-400">{rows.length} rows</span>
			</div>

			<div class="grid gap-px shrink-0 px-2 py-2 bg-slate-50 border-b border-slate-100 text-xs font-medium text-slate-500" style="grid-template-columns: 1fr 1fr 68px 58px 24px;">
				<span class="px-2">From</span>
				<span class="px-2">To</span>
				<span class="px-2">Amount</span>
				<span class="px-2">vs %</span>
				<span></span>
			</div>

			<div class="flex-1 overflow-y-auto">
				{#each rows as row (row.id)}
					<div class="grid gap-px items-center border-b border-slate-50 hover:bg-slate-50/60 group transition-colors" style="grid-template-columns: 1fr 1fr 68px 58px 24px;">
						<input
							class="w-full text-xs px-2 py-2 border border-transparent rounded focus:border-violet-300 focus:outline-none focus:bg-white bg-transparent transition-colors placeholder:text-slate-300 mx-1 my-1"
							bind:value={row.from}
							placeholder="Source"
						/>
						<input
							class="w-full text-xs px-2 py-2 border border-transparent rounded focus:border-violet-300 focus:outline-none focus:bg-white bg-transparent transition-colors placeholder:text-slate-300 mx-1 my-1"
							bind:value={row.to}
							placeholder="Target"
						/>
						<input
							type="number"
							class="w-full text-xs px-2 py-2 border border-transparent rounded focus:border-violet-300 focus:outline-none focus:bg-white bg-transparent transition-colors placeholder:text-slate-300 mx-1 my-1"
							bind:value={row.amount}
							placeholder="0"
							min="0"
						/>
						<input
							type="number"
							class="w-full text-xs px-2 py-2 border border-transparent rounded focus:border-violet-300 focus:outline-none focus:bg-white bg-transparent transition-colors placeholder:text-slate-300 mx-1 my-1"
							value={row.comparison ?? ''}
							oninput={(e) => {
								const v = (e.currentTarget as HTMLInputElement).value;
								row.comparison = v === '' ? null : parseFloat(v);
							}}
							placeholder="12.5"
						/>
						<button
							onclick={() => deleteRow(row.id)}
							class="w-6 h-6 flex items-center justify-center rounded text-slate-300 hover:text-red-500 hover:bg-red-50 transition-colors opacity-0 group-hover:opacity-100 mr-1"
							title="Delete row"
						>
							<X size={10} />
						</button>
					</div>
				{/each}
			</div>

			<div class="p-3 border-t border-slate-100 shrink-0">
				<button
					onclick={addRow}
					class="w-full py-2.5 rounded-lg border border-dashed border-slate-300 text-xs text-slate-500 hover:border-violet-400 hover:text-violet-600 hover:bg-violet-50 transition-colors"
				>+ Add row</button>
			</div>

			<!-- ── Style section ── -->
			<div class="border-t border-slate-200 shrink-0">
				<div class="px-4 py-2.5 flex items-center gap-2">
					<span class="text-xs font-semibold text-slate-500 uppercase tracking-wide">Style</span>
				</div>

				<!-- Link opacity -->
				<div class="px-4 pb-3 flex items-center gap-3">
					<span class="text-xs text-slate-500 shrink-0 w-16">Label opacity</span>
					<input
						type="range"
						min="0"
						max="1"
						step="0.01"
						bind:value={labelBoxOpacity}
						class="flex-1 accent-violet-600 h-1.5 cursor-pointer"
					/>
					<span class="text-xs text-slate-400 tabular-nums w-8 text-right">{Math.round(labelBoxOpacity * 100)}%</span>
				</div>
				<div class="px-4 pb-3 flex items-center gap-3">
					<span class="text-xs text-slate-500 shrink-0 w-16">Link opacity</span>
					<input
						type="range"
						min="0"
						max="1"
						step="0.01"
						bind:value={linkOpacity}
						class="flex-1 accent-violet-600 h-1.5 cursor-pointer"
					/>
					<span class="text-xs text-slate-400 tabular-nums w-8 text-right">{Math.round(linkOpacity * 100)}%</span>
				</div>

				<!-- Label font size -->
				<div class="px-4 pb-3 flex items-center gap-3">
					<span class="text-xs text-slate-500 shrink-0 w-16">Label size</span>
					<input
						type="range"
						min="8"
						max="24"
						step="1"
						bind:value={labelFontSize}
						class="flex-1 accent-violet-600 h-1.5 cursor-pointer"
					/>
					<span class="text-xs text-slate-400 tabular-nums w-8 text-right">{labelFontSize}px</span>
				</div>

				<!-- Node colors -->
				{#if layout.nodes.length > 0}
					<div class="px-4 pb-1 text-xs text-slate-500 font-medium">Node colors</div>
					<div class="px-3 pb-3 flex flex-col gap-0.5 max-h-40 overflow-y-auto">
						{#each layout.nodes as node (node.name)}
							<label class="flex items-center gap-2.5 py-1 px-1 rounded-lg hover:bg-slate-50 cursor-pointer group">
								<span
									class="w-5 h-5 rounded shrink-0 border border-black/10 overflow-hidden relative"
									style="background: {node.color};"
								>
									<input
										type="color"
										value={node.color}
										oninput={(e) => nodeColors.set(node.name, e.currentTarget.value)}
										class="absolute inset-0 opacity-0 cursor-pointer w-full h-full"
									/>
								</span>
								<span class="text-xs text-slate-600 truncate">{node.name}</span>
							</label>
						{/each}
					</div>
				{/if}
			</div>
		</div>
	{/if}

	<!-- ── Zoom controls ── -->
	<div
		class="absolute bottom-4 z-20 flex items-center gap-1 bg-white rounded-lg shadow-md border border-slate-200 px-2 py-1.5 transition-all duration-200"
		style:right={editorOpen ? '336px' : '16px'}
	>
		<button class="w-7 h-7 flex items-center justify-center rounded text-slate-500 hover:bg-slate-100 text-base leading-none" onclick={() => zoomBy(0.9)} title="Zoom out">−</button>
		<button class="text-xs font-medium text-slate-600 w-14 text-center hover:bg-slate-100 rounded px-1 py-0.5 tabular-nums" onclick={resetView} title="Reset view (Ctrl+0)">{Math.round(zoom * 100)}%</button>
		<button class="w-7 h-7 flex items-center justify-center rounded text-slate-500 hover:bg-slate-100 text-base leading-none" onclick={() => zoomBy(1 / 0.9)} title="Zoom in">+</button>
	</div>

	<!-- ── Keyboard hints ── -->
	<div class="absolute bottom-4 left-1/2 -translate-x-1/2 z-10 flex items-center gap-3 text-xs text-slate-400 pointer-events-none">
		<span><kbd class="bg-slate-100 border border-slate-200 px-1 rounded text-slate-500">V</kbd> Select</span>
		<span><kbd class="bg-slate-100 border border-slate-200 px-1 rounded text-slate-500">H</kbd> Hand</span>
		<span><kbd class="bg-slate-100 border border-slate-200 px-1 rounded text-slate-500">Space</kbd> Pan</span>
		<span><kbd class="bg-slate-100 border border-slate-200 px-1 rounded text-slate-500">Ctrl</kbd>+Scroll Zoom</span>
	</div>

</div>
