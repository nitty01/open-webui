<script lang="ts">
	import { getContext, onDestroy, onMount } from 'svelte';
	import { toast } from 'svelte-sonner';
	import {
		controlOllamaPull,
		deleteModel,
		getOllamaCatalogue,
		getOllamaPullStatus,
		pullModel
	} from '$lib/apis/ollama';

	const i18n = getContext('i18n');

	let loading = false;
	let search = '';
	let searchInput = '';
	let view = 'all';
	let models: any[] = [];
	let page = 1;
	let pageSize = 25;
	let total: number | null = null;
	let totalPages: number | null = null;
	let localCount = 0;
	let hasPrev = false;
	let hasNext = false;
	let sortBy = 'alias';
	let sortDir: 'asc' | 'desc' = 'asc';
	let capabilityFilter = '';
	let publisherFilter = '';
	let sourceFilter = '';
	let fitFilter = '';
	let pollTimer: ReturnType<typeof setInterval> | null = null;
	let searchDebounce: ReturnType<typeof setTimeout> | null = null;
	let pulling: Record<string, { status: string; progress?: number }> = {};
	let pullStatusesRaw: any[] = [];
	let liveDiscovery = true;
	let trustedOnly = false;
	let minDownloads = 0;
	let minLikes = 0;
	let qualityMode = 'off';
	let systemProfile: { ram_gb?: number | null; gpu_vram_gb?: number | null } | null = null;
	let liveIndexMeta: {
		count?: number;
		error?: string | null;
		stale?: boolean;
		source?: string;
	} | null = null;
	const ACTIVE_PROBE_WINDOW_MS = 30000;
	const PAGE_SIZE_OPTIONS = [10, 25, 50, 100];
	let controlSelection: Record<string, string> = {};

	const catalogueQueryOptions = (forceRefresh = false) => {
		const options: Record<string, unknown> = {
			live: liveDiscovery,
			refresh: forceRefresh,
			trusted_only: trustedOnly,
			min_downloads: minDownloads,
			min_likes: minLikes,
			quality: qualityMode,
			page,
			page_size: pageSize,
			sort_by: sortBy,
			sort_dir: sortDir,
			raw: true
		};
		const q = search.trim();
		if (q) options.q = q;
		if (view === 'installed') options.installed = true;
		if (view === 'downloadable') options.downloadable = true;
		if (capabilityFilter.trim()) options.capability = capabilityFilter.trim();
		if (publisherFilter.trim()) options.publisher = publisherFilter.trim();
		if (sourceFilter.trim()) options.source = sourceFilter.trim();
		if (fitFilter.trim()) options.fit = fitFilter.trim();
		return options;
	};

	const loadCatalogue = async (forceRefresh = false) => {
		loading = true;
		try {
			const [catalogue, pullStatuses] = await Promise.all([
				getOllamaCatalogue(localStorage.token, 0, catalogueQueryOptions(forceRefresh)),
				getOllamaPullStatus(localStorage.token, null, 0)
			]);
			pullStatusesRaw = pullStatuses ?? [];

			const statusMap = Object.fromEntries((pullStatuses ?? []).map((s) => [s.model, s]));
			systemProfile = catalogue?.system_profile ?? null;
			liveIndexMeta = catalogue?.live_index ?? null;
			total = catalogue?.total ?? null;
			localCount = catalogue?.local_count ?? 0;
			page = catalogue?.page ?? page;
			pageSize = catalogue?.page_size ?? pageSize;
			totalPages = catalogue?.total_pages ?? null;
			hasPrev = catalogue?.has_prev ?? false;
			hasNext = catalogue?.has_next ?? false;
			models = (catalogue?.models ?? []).map((m) => {
				const pullStatus = m.pull_status ?? statusMap[m.alias] ?? statusMap[m.installed_as];
				return { ...m, pull_status: pullStatus };
			});
		} catch (e) {
			console.error(e);
			toast.error($i18n.t('Failed to load model catalogue'));
		} finally {
			loading = false;
		}
	};

	const scheduleSearch = () => {
		if (searchDebounce) clearTimeout(searchDebounce);
		searchDebounce = setTimeout(async () => {
			search = searchInput;
			page = 1;
			await loadCatalogue();
		}, 350);
	};

	const goToPage = async (nextPage: number) => {
		if (nextPage < 1 || (totalPages > 0 && nextPage > totalPages)) return;
		page = nextPage;
		await loadCatalogue();
	};

	const toggleSort = async (field: string) => {
		if (sortBy === field) {
			sortDir = sortDir === 'asc' ? 'desc' : 'asc';
		} else {
			sortBy = field;
			sortDir = 'asc';
		}
		page = 1;
		await loadCatalogue();
	};

	const sortIndicator = (field: string) => {
		if (sortBy !== field) return '';
		return sortDir === 'asc' ? ' ↑' : ' ↓';
	};

	const applyFilters = async () => {
		page = 1;
		await loadCatalogue(true);
	};

	const deriveCapabilities = (m: any) => {
		if (Array.isArray(m.capabilities) && m.capabilities.length > 0) {
			return m.capabilities;
		}
		const tags = (m.tags ?? []).map((t) => `${t}`.toLowerCase());
		const out = [];
		if (tags.includes('code') || tags.includes('coder')) out.push('coding');
		if (tags.includes('reasoning') || tags.includes('deepseek')) out.push('reasoning');
		if (tags.includes('general')) out.push('chat');
		if ((m.context_length ?? 0) >= 16384) out.push('long-context');
		if (tags.includes('latest')) out.push('newer');
		return out;
	};

	const formatDate = (value: string | null | undefined) => {
		if (!value) return 'unknown';
		const dt = new Date(value);
		if (Number.isNaN(dt.getTime())) return value;
		return dt.toISOString().slice(0, 10);
	};

	const getProgress = (m: any) => {
		if (m.pull_status?.total && m.pull_status?.completed) {
			return Math.max(0, Math.min(100, Math.round((m.pull_status.completed / m.pull_status.total) * 100)));
		}
		const key = m.pull_name ?? m.alias;
		if (pulling[key]?.progress !== undefined) return pulling[key].progress;
		return null;
	};

	const getRawProgress = (s: any) => {
		if (s?.total && typeof s?.completed === 'number' && s.total > 0) {
			return Math.max(0, Math.min(100, Math.round((s.completed / s.total) * 100)));
		}
		return null;
	};

	const normalizeStatus = (status: string | null | undefined) => (status ?? '').toLowerCase();

	const isFailedStatus = (status: string | null | undefined) => {
		const s = normalizeStatus(status);
		return s.includes('error') || s.includes('fail');
	};

	const isSuccessStatus = (status: string | null | undefined) => {
		const s = normalizeStatus(status);
		return s === 'success' || s === 'done' || s.includes('complete');
	};

	const getProbeState = (m: any) => {
		const status = normalizeStatus(m?.pull_status?.status);
		if (!status) return null;
		if (status.includes('pause')) return 'paused';
		if (status.includes('suspend')) return 'suspended';
		if (isFailedStatus(status)) return 'failed';
		if (isSuccessStatus(status)) return 'completed';

		// If backend marks it as downloading/pulling, probe recency via updated_at heartbeat.
		if (status.includes('download') || status.includes('pull')) {
			const updatedAtRaw = m?.pull_status?.updated_at;
			if (!updatedAtRaw) return 'suspended';
			const updatedAt = new Date(updatedAtRaw).getTime();
			if (Number.isNaN(updatedAt)) return 'suspended';
			return Date.now() - updatedAt <= ACTIVE_PROBE_WINDOW_MS ? 'active' : 'suspended';
		}

		return status;
	};

	const getProbeClass = (probe: string | null) => {
		if (!probe) return 'bg-gray-100 text-gray-700 dark:bg-gray-800 dark:text-gray-300';
		if (probe === 'active')
			return 'bg-emerald-100 text-emerald-700 dark:bg-emerald-950/40 dark:text-emerald-300';
		if (probe === 'paused')
			return 'bg-amber-100 text-amber-700 dark:bg-amber-950/40 dark:text-amber-300';
		if (probe === 'suspended')
			return 'bg-orange-100 text-orange-700 dark:bg-orange-950/40 dark:text-orange-300';
		if (probe === 'failed')
			return 'bg-red-100 text-red-700 dark:bg-red-950/40 dark:text-red-300';
		if (probe === 'completed')
			return 'bg-blue-100 text-blue-700 dark:bg-blue-950/40 dark:text-blue-300';
		return 'bg-gray-100 text-gray-700 dark:bg-gray-800 dark:text-gray-300';
	};

	const probeModel = async (m: any) => {
		try {
			const modelKey = m.pull_name ?? m.alias ?? m.installed_as;
			const statuses = await getOllamaPullStatus(localStorage.token, modelKey, 0);
			const status = (statuses ?? [])[0] ?? null;
			models = models.map((row) =>
				row.alias === m.alias ? { ...row, pull_status: status ?? row.pull_status } : row
			);
			if (!status) {
				toast.info($i18n.t('No active pull status found for {{model}}', { model: modelKey }));
			}
		} catch (e: any) {
			toast.error($i18n.t('Probe failed: {{error}}', { error: e?.message ?? e }));
		}
	};

	const getStatusBadgeClass = (status: string | null | undefined) => {
		if (isFailedStatus(status)) {
			return 'bg-red-100 text-red-700 dark:bg-red-950/40 dark:text-red-300';
		}
		if (isSuccessStatus(status)) {
			return 'bg-emerald-100 text-emerald-700 dark:bg-emerald-950/40 dark:text-emerald-300';
		}
		if (status) {
			return 'bg-amber-100 text-amber-700 dark:bg-amber-950/40 dark:text-amber-300';
		}
		return 'bg-gray-100 text-gray-700 dark:bg-gray-800 dark:text-gray-300';
	};

	const getStatusTextClass = (status: string | null | undefined) => {
		if (isFailedStatus(status)) return 'text-red-600 dark:text-red-300';
		if (isSuccessStatus(status)) return 'text-emerald-600 dark:text-emerald-300';
		if (status) return 'text-amber-600 dark:text-amber-300';
		return 'text-gray-500 dark:text-gray-400';
	};

	const getProgressBarClass = (status: string | null | undefined) => {
		if (isFailedStatus(status)) {
			return 'bg-red-500';
		}
		if (isSuccessStatus(status)) {
			return 'bg-emerald-500';
		}
		return 'bg-amber-500';
	};

	const startPull = async (pullName: string) => {
		if (pulling[pullName]) return;
		pulling = { ...pulling, [pullName]: { status: 'starting' } };

		try {
			toast.success($i18n.t('Download started for {{alias}}', { alias: pullName }));
			const [res] = await pullModel(localStorage.token, pullName, 0);
			const reader = res?.body?.pipeThrough(new TextDecoderStream()).getReader();
			let buffer = '';

			if (reader) {
				while (true) {
					const { value, done } = await reader.read();
					if (done) break;
					buffer += value ?? '';
					const lines = buffer.split('\n');
					buffer = lines.pop() ?? '';

					for (const line of lines) {
						if (!line.trim()) continue;
						try {
							const evt = JSON.parse(line);
							if (evt.error || evt.detail) {
								throw new Error(evt.error ?? evt.detail);
							}
							const progress =
								evt.total && evt.completed
									? Math.max(0, Math.min(100, Math.round((evt.completed / evt.total) * 100)))
									: undefined;
							pulling = {
								...pulling,
								[pullName]: {
									status: evt.status ?? 'pulling',
									...(progress !== undefined ? { progress } : {})
								}
							};
						} catch {
							// Skip malformed lines from streamed output.
						}
					}
				}
			}

			toast.success($i18n.t('Download stream completed for {{alias}}', { alias: pullName }));
		} catch (e: any) {
			toast.error($i18n.t('Download failed: {{error}}', { error: e?.message ?? e }));
		} finally {
			delete pulling[pullName];
			pulling = { ...pulling };
			await loadCatalogue();
		}
	};

	const removeModel = async (m: any) => {
		const modelId = m.installed_as;
		if (!modelId) return;

		if (!window.confirm($i18n.t('Delete {{name}} from disk?', { name: m.display_name ?? modelId }))) {
			return;
		}

		try {
			await deleteModel(localStorage.token, modelId);
			toast.success($i18n.t('Deleted {{name}}', { name: m.display_name ?? modelId }));
			await loadCatalogue();
		} catch (e: any) {
			toast.error($i18n.t('Delete failed: {{error}}', { error: e?.message ?? e }));
		}
	};

	const controlPull = async (
		m: any,
		action: 'stop' | 'pause' | 'resume' | 'restart' | 'purge' | 'delete' | 'clear'
	) => {
		const modelKey = m.pull_name ?? m.alias ?? m.installed_as;
		if (!modelKey) return;
		try {
			await controlOllamaPull(localStorage.token, modelKey, action, 0);
			if (action === 'stop' || action === 'pause') {
				toast.success($i18n.t('Stopped {{model}}', { model: modelKey }));
			} else if (action === 'resume') {
				toast.success($i18n.t('Resumed {{model}}', { model: modelKey }));
			} else if (action === 'restart') {
				toast.success($i18n.t('Restarted {{model}}', { model: modelKey }));
			} else if (action === 'clear') {
				toast.success($i18n.t('Cleared status for {{model}}', { model: modelKey }));
			} else {
				toast.success($i18n.t('Purged {{model}} from system', { model: modelKey }));
			}
			await loadCatalogue();
		} catch (e: any) {
			toast.error($i18n.t('Control action failed: {{error}}', { error: e?.message ?? e }));
		}
	};

	const getControlKey = (m: any) => m.pull_name ?? m.alias ?? m.installed_as ?? '';

	const getAvailableActions = (m: any) =>
		(m.pull_status?.actions ?? []).filter((a: string) =>
			['stop', 'pause', 'resume', 'restart', 'purge', 'clear'].includes(a)
		);

	const getQuickAction = (m: any): 'stop' | 'pause' | 'resume' | 'restart' | null => {
		const actions = getAvailableActions(m);
		if (actions.includes('stop')) return 'stop';
		if (actions.includes('pause')) return 'pause';
		if (actions.includes('resume')) return 'resume';
		if (actions.includes('restart')) return 'restart';
		return null;
	};

	const actionLabel = (action: string) => {
		if (action === 'stop' || action === 'pause') return $i18n.t('Stop');
		if (action === 'resume') return $i18n.t('Resume');
		if (action === 'restart') return $i18n.t('Restart');
		if (action === 'purge') return $i18n.t('Purge');
		if (action === 'clear') return $i18n.t('Clear');
		return action;
	};

	const setSelectedControl = (m: any, value: string) => {
		const key = getControlKey(m);
		controlSelection = {
			...controlSelection,
			[key]: value
		};
	};

	const getSelectedControl = (m: any) => {
		const key = getControlKey(m);
		return controlSelection[key] ?? '';
	};

	const runSelectedControl = async (m: any) => {
		const selected = getSelectedControl(m);
		if (!selected) return;
		await controlPull(m, selected as 'stop' | 'pause' | 'resume' | 'restart' | 'purge' | 'clear');
		setSelectedControl(m, '');
	};

	const getRawAvailableActions = (s: any) =>
		(s?.actions ?? []).filter((a: string) =>
			['stop', 'pause', 'resume', 'restart', 'purge', 'clear'].includes(a)
		);

	const getRawQuickAction = (s: any): 'stop' | 'pause' | 'resume' | 'restart' | null => {
		const actions = getRawAvailableActions(s);
		if (actions.includes('stop')) return 'stop';
		if (actions.includes('pause')) return 'pause';
		if (actions.includes('resume')) return 'resume';
		if (actions.includes('restart')) return 'restart';
		return null;
	};

	const setRawSelectedControl = (model: string, value: string) => {
		controlSelection = {
			...controlSelection,
			[model]: value
		};
	};

	const getRawSelectedControl = (model: string) => controlSelection[model] ?? '';

	const runRawSelectedControl = async (s: any) => {
		const modelKey = s?.model;
		const selected = getRawSelectedControl(modelKey);
		if (!modelKey || !selected) return;
		try {
			await controlOllamaPull(
				localStorage.token,
				modelKey,
				selected as 'stop' | 'pause' | 'resume' | 'restart' | 'purge' | 'clear',
				0
			);
			toast.success($i18n.t('{{action}} applied to {{model}}', { action: actionLabel(selected), model: modelKey }));
			setRawSelectedControl(modelKey, '');
			await loadCatalogue();
		} catch (e: any) {
			toast.error($i18n.t('Control action failed: {{error}}', { error: e?.message ?? e }));
		}
	};

	const runRawQuickAction = async (s: any) => {
		const modelKey = s?.model;
		const action = getRawQuickAction(s);
		if (!modelKey || !action) return;
		try {
			await controlOllamaPull(localStorage.token, modelKey, action, 0);
			toast.success($i18n.t('{{action}} applied to {{model}}', { action: actionLabel(action), model: modelKey }));
			await loadCatalogue();
		} catch (e: any) {
			toast.error($i18n.t('Control action failed: {{error}}', { error: e?.message ?? e }));
		}
	};

	onMount(async () => {
		searchInput = search;
		await loadCatalogue();
		pollTimer = setInterval(() => {
			loadCatalogue(false);
		}, 10000);
	});

	onDestroy(() => {
		if (pollTimer) clearInterval(pollTimer);
		if (searchDebounce) clearTimeout(searchDebounce);
	});
</script>

<div class="text-sm">
	<div class="mb-4 flex flex-col gap-2 lg:flex-row lg:items-center lg:justify-between">
		<div>
			<div class="text-lg font-medium">{$i18n.t('Model Explorer')}</div>
			<div class="text-xs text-gray-500 dark:text-gray-400">
				{$i18n.t('Browse compatible models, metadata, capabilities, and storage actions')}
			</div>
		</div>
		<button
			class="px-3 py-1.5 rounded-lg bg-gray-900 text-white dark:bg-white dark:text-gray-900 text-xs font-medium"
			on:click={() => loadCatalogue(true)}
		>
			{$i18n.t('Refresh Live Catalogue')}
		</button>
	</div>
	{#if systemProfile}
		<div class="mb-3 text-xs text-gray-500 dark:text-gray-400">
			{$i18n.t('System profile')}: RAM {systemProfile?.ram_gb ?? 'unknown'} GB, GPU VRAM {systemProfile?.gpu_vram_gb ?? 'unknown'} GB
		</div>
	{/if}
	{#if liveDiscovery && liveIndexMeta}
		<div class="mb-3 text-xs text-gray-500 dark:text-gray-400">
			{$i18n.t('HF live index')}: {liveIndexMeta?.count ?? 0}
			{#if liveIndexMeta?.stale}
				<span class="text-amber-600 dark:text-amber-300"> · {$i18n.t('stale cache')}</span>
			{/if}
			{#if liveIndexMeta?.error}
				<span class="text-red-600 dark:text-red-300"> · {liveIndexMeta.error}</span>
			{/if}
		</div>
	{/if}

	<div class="mb-3 flex flex-col gap-2 lg:flex-row lg:flex-wrap">
		<input
			class="flex-1 px-3 py-2 rounded-xl bg-gray-100 dark:bg-gray-850 outline-hidden"
			placeholder={$i18n.t('Search (repo:, publisher:, tag:, capability:, fit:)')}
			bind:value={searchInput}
			on:input={scheduleSearch}
		/>
		<select
			class="px-3 py-2 rounded-xl bg-gray-100 dark:bg-gray-850 outline-hidden"
			bind:value={view}
			on:change={applyFilters}
		>
			<option value="all">{$i18n.t('All')}</option>
			<option value="installed">{$i18n.t('Installed')}</option>
			<option value="downloadable">{$i18n.t('Downloadable')}</option>
		</select>
		<label class="px-3 py-2 rounded-xl bg-gray-100 dark:bg-gray-850 inline-flex items-center gap-2">
			<input type="checkbox" bind:checked={liveDiscovery} on:change={applyFilters} />
			{$i18n.t('Include live discovery')}
		</label>
		<label class="px-3 py-2 rounded-xl bg-gray-100 dark:bg-gray-850 inline-flex items-center gap-2">
			<input type="checkbox" bind:checked={trustedOnly} on:change={applyFilters} />
			{$i18n.t('Trusted publishers only')}
		</label>
		<label class="px-3 py-2 rounded-xl bg-gray-100 dark:bg-gray-850 inline-flex items-center gap-2">
			{$i18n.t('Min downloads')}
			<input
				type="number"
				min="0"
				class="w-24 bg-transparent outline-hidden"
				bind:value={minDownloads}
				on:change={applyFilters}
			/>
		</label>
		<label class="px-3 py-2 rounded-xl bg-gray-100 dark:bg-gray-850 inline-flex items-center gap-2">
			{$i18n.t('Min likes')}
			<input
				type="number"
				min="0"
				class="w-20 bg-transparent outline-hidden"
				bind:value={minLikes}
				on:change={applyFilters}
			/>
		</label>
		<select
			class="px-3 py-2 rounded-xl bg-gray-100 dark:bg-gray-850 outline-hidden"
			bind:value={qualityMode}
			on:change={applyFilters}
		>
			<option value="off">{$i18n.t('Quality filter off')}</option>
			<option value="strict">{$i18n.t('Strict quality')}</option>
		</select>
		<input
			class="px-3 py-2 rounded-xl bg-gray-100 dark:bg-gray-850 outline-hidden"
			placeholder={$i18n.t('Capability filter')}
			bind:value={capabilityFilter}
			on:change={applyFilters}
		/>
		<input
			class="px-3 py-2 rounded-xl bg-gray-100 dark:bg-gray-850 outline-hidden"
			placeholder={$i18n.t('Publisher filter')}
			bind:value={publisherFilter}
			on:change={applyFilters}
		/>
		<input
			class="px-3 py-2 rounded-xl bg-gray-100 dark:bg-gray-850 outline-hidden"
			placeholder={$i18n.t('Source filter')}
			bind:value={sourceFilter}
			on:change={applyFilters}
		/>
		<select
			class="px-3 py-2 rounded-xl bg-gray-100 dark:bg-gray-850 outline-hidden"
			bind:value={fitFilter}
			on:change={applyFilters}
		>
			<option value="">{$i18n.t('Any fit')}</option>
			<option value="recommended">{$i18n.t('Recommended')}</option>
			<option value="possible">{$i18n.t('Possible')}</option>
			<option value="not_recommended">{$i18n.t('Not recommended')}</option>
		</select>
	</div>

	<div class="mb-3 flex flex-wrap items-center justify-between gap-2 text-xs text-gray-500 dark:text-gray-400">
		<div>
			{#if total != null}
				{$i18n.t('Showing {{count}} of {{total}} models', { count: models.length, total })}
				{#if totalPages != null && totalPages > 0}
					<span> · {$i18n.t('Page {{page}} of {{pages}}', { page, pages: totalPages })}</span>
				{/if}
			{:else}
				{$i18n.t('{{count}} models on this page', { count: models.length })}
				<span> · {$i18n.t('Page {{page}}', { page })}</span>
				{#if hasNext}
					<span> · {$i18n.t('more available from Hugging Face')}</span>
				{/if}
				{#if localCount > 0}
					<span> · {localCount} {$i18n.t('local')}</span>
				{/if}
			{/if}
		</div>
		<div class="flex items-center gap-2">
			<label class="inline-flex items-center gap-1.5">
				{$i18n.t('Page size')}
				<select
					class="px-2 py-1 rounded-lg bg-gray-100 dark:bg-gray-850 outline-hidden"
					bind:value={pageSize}
					on:change={applyFilters}
				>
					{#each PAGE_SIZE_OPTIONS as size}
						<option value={size}>{size}</option>
					{/each}
				</select>
			</label>
			<button
				class="px-2 py-1 rounded-lg border border-gray-300 dark:border-gray-700 disabled:opacity-40"
				disabled={!hasPrev || loading}
				on:click={() => goToPage(1)}
			>
				{$i18n.t('First')}
			</button>
			<button
				class="px-2 py-1 rounded-lg border border-gray-300 dark:border-gray-700 disabled:opacity-40"
				disabled={!hasPrev || loading}
				on:click={() => goToPage(page - 1)}
			>
				{$i18n.t('Prev')}
			</button>
			<button
				class="px-2 py-1 rounded-lg border border-gray-300 dark:border-gray-700 disabled:opacity-40"
				disabled={!hasNext || loading}
				on:click={() => goToPage(page + 1)}
			>
				{$i18n.t('Next')}
			</button>
			<button
				class="px-2 py-1 rounded-lg border border-gray-300 dark:border-gray-700 disabled:opacity-40"
				disabled={!hasNext || loading}
				on:click={() => goToPage(totalPages)}
			>
				{$i18n.t('Last')}
			</button>
		</div>
	</div>

	<div class="rounded-2xl border border-gray-200 dark:border-gray-800 overflow-hidden">
		{#if pullStatusesRaw.length > 0}
			<div class="px-3 py-2 border-b border-gray-200 dark:border-gray-800 bg-gray-50 dark:bg-gray-900/70">
				<div class="text-[0.7rem] uppercase tracking-wide text-gray-500 dark:text-gray-400">
					{$i18n.t('Transfer Status')}
				</div>
				<div class="mt-2 grid gap-1.5">
					{#each pullStatusesRaw as s (s.model)}
						<div class="rounded-lg border border-gray-200 dark:border-gray-800 bg-white dark:bg-gray-950/40 px-2.5 py-1.5">
							<div class="flex items-center justify-between gap-2">
								<div class="min-w-0">
									<div class="truncate font-medium">{s.model}</div>
									<div class={`text-[0.7rem] ${getStatusTextClass(s.status)}`}>
										{s.status}
									</div>
								</div>
								{#if getRawProgress(s) !== null}
									<div class="text-xs text-gray-500 dark:text-gray-400">{getRawProgress(s)}%</div>
								{/if}
							</div>
							{#if getRawAvailableActions(s).length > 0}
								<div class="mt-1.5 flex flex-wrap items-center gap-1.5">
									{#if getRawQuickAction(s)}
										<button
											class="px-2 py-1 rounded-lg border border-blue-500 text-blue-700 dark:text-blue-300 text-[0.7rem]"
											on:click={() => runRawQuickAction(s)}
										>
											{actionLabel(getRawQuickAction(s))}
										</button>
									{/if}
									<select
										class="min-w-[7rem] px-2 py-1 rounded-lg bg-gray-100 dark:bg-gray-850 border border-gray-300 dark:border-gray-700 text-[0.7rem] outline-hidden"
										value={getRawSelectedControl(s.model)}
										on:change={(e) => setRawSelectedControl(s.model, e.currentTarget.value)}
									>
										<option value="">{`${$i18n.t('More controls')}...`}</option>
										{#each getRawAvailableActions(s) as action}
											{#if action !== getRawQuickAction(s)}
												<option value={action}>{actionLabel(action)}</option>
											{/if}
										{/each}
									</select>
									<button
										class="px-2 py-1 rounded-lg border border-gray-500 text-gray-700 dark:text-gray-200 text-[0.7rem] disabled:opacity-40"
										disabled={!getRawSelectedControl(s.model)}
										on:click={() => runRawSelectedControl(s)}
									>
										{$i18n.t('Run')}
									</button>
								</div>
							{/if}
							{#if getRawProgress(s) !== null}
								<div class="mt-1 h-1.5 w-full rounded-full bg-gray-200 dark:bg-gray-800 overflow-hidden">
									<div
										class={`h-full rounded-full transition-all duration-300 ${getProgressBarClass(s.status)}`}
										style={`width: ${getRawProgress(s)}%`}
									></div>
								</div>
							{/if}
							{#if s.error}
								<div class="mt-1 text-[0.7rem] text-red-600 dark:text-red-300 break-words">{s.error}</div>
							{/if}
						</div>
					{/each}
				</div>
			</div>
		{/if}
		<div class="max-h-[65vh] overflow-auto">
			<table class="w-full text-left text-xs">
				<thead class="sticky top-0 bg-gray-50 dark:bg-gray-900/95">
					<tr class="border-b border-gray-200 dark:border-gray-800">
						<th class="px-3 py-2">
							<button class="font-medium" on:click={() => toggleSort('display_name')}>
								{$i18n.t('Model')}{sortIndicator('display_name')}
							</button>
						</th>
						<th class="px-3 py-2">{$i18n.t('Capabilities')}</th>
						<th class="px-3 py-2">
							<button class="font-medium" on:click={() => toggleSort('downloads')}>
								{$i18n.t('Metadata')}{sortIndicator('downloads')}
							</button>
						</th>
						<th class="px-3 py-2">
							<button class="font-medium" on:click={() => toggleSort('fit')}>
								{$i18n.t('Status')}{sortIndicator('fit')}
							</button>
						</th>
						<th class="px-3 py-2">{$i18n.t('Actions')}</th>
					</tr>
				</thead>
				<tbody>
					{#if loading && models.length === 0}
						<tr><td class="px-3 py-3 text-gray-500" colspan="5">{$i18n.t('Loading...')}</td></tr>
					{:else if models.length === 0}
						<tr><td class="px-3 py-3 text-gray-500" colspan="5">{$i18n.t('No models found')}</td></tr>
					{:else}
						{#each models as m (m.alias)}
							<tr class="border-b border-gray-100 dark:border-gray-850 align-top">
								<td class="px-3 py-2 min-w-[220px]">
									<div class="font-medium">{m.display_name ?? m.alias}</div>
									<div class="text-gray-500 dark:text-gray-400">{m.alias}</div>
								</td>
								<td class="px-3 py-2 min-w-[180px]">
									<div class="flex flex-wrap gap-1">
										{#each deriveCapabilities(m) as cap}
											<span class="px-2 py-0.5 rounded-full bg-gray-200 dark:bg-gray-800">{cap}</span>
										{/each}
										{#if deriveCapabilities(m).length === 0}
											<span class="text-gray-400">-</span>
										{/if}
									</div>
								</td>
								<td class="px-3 py-2 min-w-[320px]">
									<div><span class="text-gray-500">repo:</span> {m.repo}</div>
									<div><span class="text-gray-500">file:</span> {m.filename}</div>
									<div><span class="text-gray-500">context:</span> {m.context_length ?? 'unknown'}</div>
									<div><span class="text-gray-500">updated:</span> {formatDate(m?.metadata?.knowledge_last_update)}</div>
									<div><span class="text-gray-500">downloads:</span> {m?.metadata?.downloads ?? 'unknown'}</div>
									<div><span class="text-gray-500">likes:</span> {m?.metadata?.likes ?? 'unknown'}</div>
									{#if m?.metadata?.pipeline_tag}
										<div><span class="text-gray-500">pipeline:</span> {m.metadata.pipeline_tag}</div>
									{/if}
									<div class="mt-1 flex flex-wrap gap-1">
										{#each m.tags ?? [] as tag}
											<span class="px-1.5 py-0.5 rounded bg-gray-100 dark:bg-gray-850">{tag}</span>
										{/each}
									</div>
								</td>
								<td class="px-3 py-2 min-w-[150px]">
									<div>{m.installed ? $i18n.t('Installed') : $i18n.t('Not installed')}</div>
									{#if m.runtime_fit}
										<div class="mt-1 text-gray-500 dark:text-gray-400">
											{$i18n.t('Fit')}: {m.runtime_fit.recommendation}
										</div>
										<div class="text-gray-500 dark:text-gray-400">{m.runtime_fit.reason}</div>
										{#if m.runtime_fit.estimated_model_gb}
											<div class="text-gray-500 dark:text-gray-400">
												{$i18n.t('Estimated size')}: {m.runtime_fit.estimated_model_gb} GB
											</div>
										{/if}
									{/if}
									{#if m.pull_status?.status}
										<div class="mt-1">
											<span
												class={`inline-flex items-center rounded-full px-2 py-0.5 text-[0.65rem] font-medium ${getStatusBadgeClass(
													m.pull_status.status
												)}`}
											>
												{m.pull_status.status}
											</span>
										</div>
									{/if}
									{#if getProgress(m) !== null}
										<div class="mt-1 text-gray-500 dark:text-gray-400">{getProgress(m)}%</div>
										<div class="mt-1 h-1.5 w-full rounded-full bg-gray-200 dark:bg-gray-800 overflow-hidden">
											<div
												class={`h-full rounded-full transition-all duration-300 ${getProgressBarClass(
													m.pull_status?.status
												)}`}
												style={`width: ${getProgress(m)}%`}
											></div>
										</div>
									{/if}
									{#if getProbeState(m)}
										<div class="mt-1">
											<span
												class={`inline-flex items-center rounded-full px-2 py-0.5 text-[0.65rem] font-medium ${getProbeClass(
													getProbeState(m)
												)}`}
											>
												{$i18n.t('Probe')}: {getProbeState(m)}
											</span>
										</div>
									{/if}
									{#if m.pull_status?.error}
										<div class="mt-1 text-red-600 dark:text-red-300 break-words">
											{m.pull_status.error}
										</div>
									{/if}
								</td>
								<td class="px-3 py-2 min-w-[160px]">
									<div class="space-y-1.5">
										<div class="flex flex-wrap gap-1.5">
										{#if m.downloadable}
											<button
												class="px-2 py-1 rounded-lg bg-gray-900 text-white dark:bg-white dark:text-gray-900 text-[0.7rem]"
												on:click={() => startPull(m.pull_name ?? m.alias)}
											>
												{$i18n.t('Download')}
											</button>
										{/if}
										{#if m.pull_status?.status}
											{#if getQuickAction(m)}
												<button
													class="px-2 py-1 rounded-lg border border-blue-500 text-blue-700 dark:text-blue-300 text-[0.7rem]"
													on:click={() => controlPull(m, getQuickAction(m))}
												>
													{actionLabel(getQuickAction(m))}
												</button>
											{/if}
											<button
												class="px-2 py-1 rounded-lg border border-gray-400 text-gray-700 dark:text-gray-200 text-[0.7rem]"
												on:click={() => probeModel(m)}
											>
												{$i18n.t('Probe')}
											</button>
										{/if}
										{#if m.installed && m.deletable !== false}
											<button
												class="px-2 py-1 rounded-lg border border-red-500 bg-red-50 text-red-700 dark:bg-red-950/30 dark:text-red-300 text-[0.7rem]"
												on:click={() => removeModel(m)}
											>
												{$i18n.t('Delete from system')}
											</button>
										{/if}
										</div>
										{#if getAvailableActions(m).length > 0}
											<div class="flex items-center gap-1.5">
												<select
													class="min-w-[7rem] px-2 py-1 rounded-lg bg-gray-100 dark:bg-gray-850 border border-gray-300 dark:border-gray-700 text-[0.7rem] outline-hidden"
													value={getSelectedControl(m)}
													on:change={(e) => setSelectedControl(m, e.currentTarget.value)}
												>
													<option value="">{`${$i18n.t('More controls')}...`}</option>
													{#each getAvailableActions(m) as action}
														{#if action !== getQuickAction(m)}
															<option value={action}>{actionLabel(action)}</option>
														{/if}
													{/each}
												</select>
												<button
													class="px-2 py-1 rounded-lg border border-gray-500 text-gray-700 dark:text-gray-200 text-[0.7rem] disabled:opacity-40"
													disabled={!getSelectedControl(m)}
													on:click={() => runSelectedControl(m)}
												>
													{$i18n.t('Run')}
												</button>
											</div>
										{/if}
									</div>
								</td>
							</tr>
						{/each}
					{/if}
				</tbody>
			</table>
		</div>
	</div>
</div>
