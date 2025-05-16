<script lang="ts">
	import { Select, Slider, Separator, ToggleGroup, Tooltip } from 'bits-ui';
	import CaretUpDown from 'phosphor-svelte/lib/CaretUpDown';
	import CaretDoubleUp from 'phosphor-svelte/lib/CaretDoubleUp';
	import CaretDoubleDown from 'phosphor-svelte/lib/CaretDoubleDown';
	import MapPin from 'phosphor-svelte/lib/MapPin';
	import Bird from 'phosphor-svelte/lib/Bird';
	import Check from 'phosphor-svelte/lib/Check';
	import Clock from 'phosphor-svelte/lib/Clock';
	import ArrowCounterClockwise from 'phosphor-svelte/lib/ArrowCounterClockwise';
	import DotsSixVertical from 'phosphor-svelte/lib/DotsSixVertical';
	import CaretUp from 'phosphor-svelte/lib/CaretUp';
	import CaretDown from 'phosphor-svelte/lib/CaretDown';
	import { onMount, onDestroy } from 'svelte';
	import { slide } from 'svelte/transition';

	export let selectedSpecies: string;
	export let speciesSelectItems: { value: string; label: string }[];
	export let selectedSite: string;
	export let siteSelectItems: { value: string; label: string }[];
	export let selectedHour: [number, number];
	export let hourOptions: number[];
	export let speciesCounts: Record<string, number> = {};
	export let sceneLoaded: boolean = false;

	// Tooltip data from ScatterPlot
	export let tooltip: {
		visible: boolean;
		x: number;
		y: number;
		species: string;
		site: string;
		time: string;
		confidence: number | null;
	} = { visible: false, x: 0, y: 0, species: '', site: '', time: '', confidence: null };

	// Color mode control
	export let colorMode: 'time' | 'species' | 'site' | '' = '';
	const colorModes = [
		{ value: '', label: 'None', icon: null },
		{ value: 'time', label: 'Time', icon: Clock },
		{ value: 'species', label: 'Species', icon: Bird },
		{ value: 'site', label: 'Site', icon: MapPin }
	];

	// For reset functionality
	export let onReset: () => void;

	let menuElement: HTMLDivElement;
	let dragging = false;
	let position = { x: 0, y: 0 };
	let offset = { x: 0, y: 0 };
	let hasBeenDragged = false;
	let collapsed = false;

	function initializePosition() {
		if (menuElement && !hasBeenDragged) {
			const menuWidth = menuElement.offsetWidth;
			const menuHeight = menuElement.offsetHeight;
			position.x = (window.innerWidth - menuWidth) / 2;
			position.y = window.innerHeight - menuHeight - 60;
		}
	}

	function toggleCollapsed() {
		collapsed = !collapsed;
	}

	onMount(() => {
		setTimeout(() => {
			initializePosition();
		}, 0);

		window.addEventListener('mousemove', handleMouseMove);
		window.addEventListener('mouseup', handleMouseUp);
		window.addEventListener('resize', initializePosition);
	});

	onDestroy(() => {
		window.removeEventListener('mousemove', handleMouseMove);
		window.removeEventListener('mouseup', handleMouseUp);
		window.removeEventListener('resize', initializePosition);
	});

	function handleMouseDown(event: MouseEvent) {
		const target = event.target as HTMLElement;
		if (
			target.closest(
				'button, input, select, textarea, a, [role="button"], [role="slider"], [role="listbox"], [role="option"], [role="combobox"], [role="radio"], [role="checkbox"], [data-bits-toggle-group-item], [data-bits-select-trigger], [data-bits-slider-thumb]'
			)
		) {
			return;
		}

		dragging = true;
		hasBeenDragged = true;

		const rect = menuElement.getBoundingClientRect();
		offset.x = event.clientX - rect.left;
		offset.y = event.clientY - rect.top;

		event.preventDefault();
	}

	function handleMouseMove(event: MouseEvent) {
		if (dragging) {
			position.x = event.clientX - offset.x;
			position.y = event.clientY - offset.y;
		}
	}

	function handleMouseUp() {
		if (dragging) {
			dragging = false;
		}
	}

	function getLabel(items, value, placeholder) {
		return items.find((i) => i.value === value)?.label ?? placeholder;
	}
</script>

{#if tooltip.visible}
	<div
		class="bg-[var(--color-menu-background)]/70 shadow-popover pointer-events-none fixed z-50 rounded-lg border border-zinc-600 px-3 py-2 text-xs text-zinc-200 backdrop-blur-md"
		style="top: {tooltip.y + 15}px; left: {tooltip.x + 15}px; min-width: 120px; max-width: 240px;"
	>
		<div><span class="text-zinc-300">Species:</span> {tooltip.species || 'N/A'}</div>
		<div><span class="text-zinc-300">Site:</span> {tooltip.site || 'N/A'}</div>
		<div><span class="text-zinc-300">Time:</span> {tooltip.time || 'N/A'}</div>
		<div>
			<span class="text-zinc-300">Confidence:</span>
			{tooltip.confidence !== null ? tooltip.confidence.toFixed(2) : 'N/A'}
		</div>
	</div>
{/if}

<!-- Mobile Caret button, center bottom, absolutely positioned -->
{#if window.innerWidth < 1024}
	<div class="fixed left-1/2 bottom-4 z-50 -translate-x-1/2">
		{#if collapsed}
			<button
				type="button"
				class="flex items-center justify-center w-12 h-12"
				on:click|stopPropagation={toggleCollapsed}
				aria-label="Expand controls"
			>
				<CaretUp class="size-5 text-zinc-300" />
			</button>
		{:else}
			<button
				type="button"
				class="flex items-center justify-center w-12 h-12"
				on:click|stopPropagation={toggleCollapsed}
				aria-label="Collapse controls"
			>
				<CaretDown class="size-5 text-zinc-300" />
			</button>
		{/if}
	</div>
{/if}

<!-- Controls Menu -->
{#if !collapsed || window.innerWidth >= 1024}
	<div
		bind:this={menuElement}
		on:mousedown={handleMouseDown}
		role="toolbar"
		aria-label="Controls Panel"
		tabindex="0"
		class="fixed z-40 bg-[var(--color-menu-background)]/70 shadow-popover pointer-events-auto
		flex flex-col lg:flex-row lg:items-center gap-2 rounded-xl p-2 lg:px-3
		lg:py-3 text-zinc-100 backdrop-blur-md transition-all duration-200
		data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95
		w-[90vw] min-w-[90vw] max-w-[95vw] md:max-w-[450px] md:min-w-[450px] lg:w-auto lg:min-w-0 lg:max-w-none"
		style="left: {position.x}px; top: {position.y}px; 
			cursor: {dragging ? 'grabbing' : 'grab'};
			transition: opacity 2s cubic-bezier(.4,0,.2,1); opacity: {sceneLoaded ? 1 : 0};
			pointer-events: {sceneLoaded ? 'auto' : 'none'};
			--tw-animate-duration: 250ms;"
		data-state={sceneLoaded && (!collapsed || window.innerWidth >= 1024) ? 'open' : 'closed'}
	>
		<!-- Drag Handle Icon (desktop only) -->
		<div class="hidden lg:flex items-center justify-center h-10 w-auto lg:-ml-1.5 lg:-mr-1 text-zinc-400 hover:text-zinc-200 transition-colors">
			<DotsSixVertical class="size-5 text-zinc-300" weight="bold" />
		</div>

		<!-- Color Mode ToggleGroup -->
		<Tooltip.Provider>
			<Tooltip.Root delayDuration={200}>
				<Tooltip.Trigger asChild>
					<div class="w-full lg:min-w-[120px] flex flex-col">
						<div
							class="bg-background/80 flex w-full h-10 items-center justify-between rounded-[9px] border border-zinc-500 px-2"
						>
							<span class="mr-1 ml-1 text-sm text-zinc-300">Color
							by</span>
							<ToggleGroup.Root bind:value={colorMode} type="single" class="flex items-center">
								{#each colorModes as mode}
									<ToggleGroup.Item
										aria-label={mode.label === 'None' ? 'No color' : 'Color by ' + mode.label}
										value={mode.value}
										class="inline-flex size-8 items-center justify-center rounded-[5px] transition-all hover:bg-zinc-700/30 data-[state=off]:text-zinc-400 data-[state=on]:bg-zinc-700/60 data-[state=on]:text-zinc-100"
									>
										{#if mode.icon}
											<svelte:component this={mode.icon} class="size-5 text-zinc-300 flex-shrink-0" />
										{:else}
											<span class="flex size-5 items-center justify-center text-zinc-400 flex-shrink-0">–</span>
										{/if}
									</ToggleGroup.Item>
								{/each}
							</ToggleGroup.Root>
						</div>
					</div>
				</Tooltip.Trigger>
				<Tooltip.Content sideOffset={14}>
					<div class="bg-[var(--color-menu-background)]/70 bg-background shadow-popover backdrop-blur-md border border-zinc-500 rounded-xl px-3 py-2 text-xs text-zinc-300">
						Choose how to color the points
					</div>
				</Tooltip.Content>
			</Tooltip.Root>
		</Tooltip.Provider>

		<!-- Species Select -->
		<Tooltip.Provider>
			<Tooltip.Root delayDuration={200}>
				<Tooltip.Trigger asChild>
					<div class="w-full lg:min-w-[120px] flex flex-col">
						<Select.Root type="single" bind:value={selectedSpecies} items={speciesSelectItems}>
							<Select.Trigger
								class="bg-background/80 flex w-full lg:w-48 lg:max-w-[12rem] lg:min-w-[12rem] h-10 items-center rounded-[9px] border border-zinc-500 px-3 text-sm font-medium transition-colors select-none hover:bg-zinc-700/30 focus:ring-0 focus:outline-none data-placeholder:text-zinc-300 data-[state=open]:border-zinc-500 data-[state=open]:bg-zinc-700/50"
								aria-label="Select species"
							>
								<Bird class="mr-2 size-5 text-zinc-300 flex-shrink-0" />
								<span class="truncate">{getLabel(speciesSelectItems, selectedSpecies, 'Select Species')}</span>
								<CaretUpDown class="ml-auto size-5 text-zinc-300 flex-shrink-0" />
							</Select.Trigger>
							<Select.Portal>
								<Select.Content
									class="bg-[var(--color-menu-background)]/70 bg-background shadow-popover focus-override data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 z-50 max-h-80 w-[var(--bits-select-anchor-width)] min-w-[var(--bits-select-anchor-width)] rounded-xl
										border border-zinc-500 px-1 py-2 outline-hidden backdrop-blur-md select-none"
									sideOffset={14}
								>
									<Select.ScrollUpButton
										class="flex w-full items-center justify-center py-1 text-zinc-300"
									>
										<CaretDoubleUp class="size-4 text-zinc-300 flex-shrink-0" />
									</Select.ScrollUpButton>
									<Select.Viewport class="p-1">
										{#each speciesSelectItems as item (item.value)}
											<Select.Item
												value={item.value}
												label={item.label}
												class="flex h-9 w-full items-center rounded-[5px] px-3 text-sm text-zinc-100 capitalize outline-hidden transition-colors select-none data-[disabled]:opacity-50 data-[highlighted]:bg-zinc-700/60 data-[highlighted]:text-zinc-100"
												disabled={item.disabled}
											>
												{#snippet children({ selected })}
													<span
														class="block max-w-[10.5rem] overflow-hidden text-ellipsis whitespace-nowrap"
														>{item.label}</span
													>
													{#if !selected}
														<span class="ml-auto text-xs font-normal text-zinc-400 tabular-nums"
															>{speciesCounts[item.value] ?? ''}</span
														>
													{/if}
													{#if selected}
														<Check class="ml-auto size-4 text-zinc-300 flex-shrink-0" />
													{/if}
												{/snippet}
											</Select.Item>
										{/each}
									</Select.Viewport>
									<Select.ScrollDownButton
										class="flex w-full items-center justify-center py-1 text-zinc-300"
									>
										<CaretDoubleDown class="size-4 text-zinc-300 flex-shrink-0" />
									</Select.ScrollDownButton>
								</Select.Content>
							</Select.Portal>
						</Select.Root>
					</div>
				</Tooltip.Trigger>
				<Tooltip.Content sideOffset={14}>
					<div class="bg-[var(--color-menu-background)]/70 bg-background shadow-popover backdrop-blur-md border border-zinc-500 rounded-xl px-3 py-2 text-xs text-zinc-300">
						Filter by species
					</div>
				</Tooltip.Content>
			</Tooltip.Root>
		</Tooltip.Provider>

		<!-- Site Select -->
		<Tooltip.Provider>
			<Tooltip.Root delayDuration={200}>
				<Tooltip.Trigger asChild>
					<div class="w-full lg:min-w-[120px] flex flex-col">
						<Select.Root type="single" bind:value={selectedSite} items={siteSelectItems}>
							<Select.Trigger
								class="bg-background/80 flex w-full lg:w-32 lg:max-w-[8rem] lg:min-w-[8rem] h-10 items-center rounded-[9px] border border-zinc-500 px-3 text-sm font-medium transition-colors select-none hover:bg-zinc-700/30 focus:ring-0 focus:outline-none data-placeholder:text-zinc-300 data-[state=open]:border-zinc-500 data-[state=open]:bg-zinc-700/50"
								aria-label="Select site"
							>
								<MapPin class="mr-2 size-5 text-zinc-300 flex-shrink-0" />
								<span class="truncate">{getLabel(siteSelectItems, selectedSite, 'Select Site')}</span>
								<CaretUpDown class="ml-auto size-5 text-zinc-300 flex-shrink-0" />
							</Select.Trigger>
							<Select.Portal>
								<Select.Content
									class="bg-[var(--color-menu-background)]/70 bg-background shadow-popover focus-override data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 z-50 max-h-80 w-[var(--bits-select-anchor-width)] min-w-[var(--bits-select-anchor-width)] rounded-xl
										border border-zinc-500 px-1 py-2 outline-hidden backdrop-blur-md select-none"
									sideOffset={14}
								>
									<Select.ScrollUpButton
										class="flex w-full items-center justify-center py-1 text-zinc-300"
									>
										<CaretDoubleUp class="size-4 text-zinc-300 flex-shrink-0" />
									</Select.ScrollUpButton>
									<Select.Viewport class="p-1">
										{#each siteSelectItems as item (item.value)}
											<Select.Item
												value={item.value}
												label={item.label}
												class="flex h-9 w-full items-center rounded-[5px] px-3 text-sm text-zinc-100 capitalize outline-hidden transition-colors select-none data-[disabled]:opacity-50 data-[highlighted]:bg-zinc-700/60 data-[highlighted]:text-zinc-100"
												disabled={item.disabled}
											>
												{#snippet children({ selected })}
													{item.label}
													{#if selected}
														<Check class="ml-auto size-4 text-zinc-300 flex-shrink-0" />
													{/if}
												{/snippet}
											</Select.Item>
										{/each}
									</Select.Viewport>
									<Select.ScrollDownButton
										class="flex w-full items-center justify-center py-1 text-zinc-300"
									>
										<CaretDoubleDown class="size-4 text-zinc-300 flex-shrink-0" />
									</Select.ScrollDownButton>
								</Select.Content>
							</Select.Portal>
						</Select.Root>
					</div>
				</Tooltip.Trigger>
				<Tooltip.Content sideOffset={14}>
					<div class="bg-[var(--color-menu-background)]/70 bg-background shadow-popover backdrop-blur-md border border-zinc-500 rounded-xl px-3 py-2 text-xs text-zinc-300">
						Filter by site
					</div>
				</Tooltip.Content>
			</Tooltip.Root>
		</Tooltip.Provider>

		<!-- Time/Slider + Reset Button in a row, no overlap -->
		<Tooltip.Provider>
			<Tooltip.Root delayDuration={200}>
				<Tooltip.Trigger asChild>
					<div class="flex w-full lg:min-w-[320px] flex-row items-center gap-0">
						<div class="flex w-full flex-col items-stretch">
							<div class="w-full">
								<div
									class="bg-background/80 flex w-full h-10 items-center rounded-[9px] border border-zinc-500 px-3 py-1 lg:py-0"
								>
									<Clock class="mr-2 size-5 text-zinc-300 flex-shrink-0" />
									<span
										class="rounded border-none bg-transparent py-0.5 font-mono text-xs font-medium lg:whitespace-nowrap text-zinc-100 mr-1 lg:mr-4 flex-shrink-0"
									>
										{#if selectedHour && selectedHour.length === 2}
											{Math.floor(selectedHour[0] / 2)
												.toString()
												.padStart(2, '0')}:{selectedHour[0] % 2 === 0 ? '00' : '30'}h -
											{Math.floor(selectedHour[1] / 2)
												.toString()
												.padStart(2, '0')}:{selectedHour[1] % 2 === 0 ? '00' : '30'}h
										{:else}
											--:--h - --:--h
										{/if}
									</span>
									<Slider.Root
										type="multiple"
										min={hourOptions[0] ?? 0}
										max={hourOptions[hourOptions.length - 1] ?? 47}
										step={1}
										bind:value={selectedHour}
										class="relative flex flex-grow min-w-0 touch-none items-center select-none"
									>
										{#snippet children({ thumbs })}
											<span
												class="relative h-0.5 w-full grow cursor-pointer overflow-hidden rounded bg-zinc-600"
											>
												<Slider.Range class="absolute h-full bg-zinc-300" />
											</span>
											{#each thumbs as index}
												<Slider.Thumb
													{index}
													class="block h-2.5 w-2.5 cursor-pointer rounded-full border border-zinc-300 bg-zinc-300 shadow-md transition-none hover:border-zinc-100 focus:border-zinc-200 focus:outline-none disabled:pointer-events-none disabled:opacity-50"
												/>
											{/each}
										{/snippet}
									</Slider.Root>
								</div>
							</div>
						</div>
						<!-- Reset Button, outside the time/slider box, with left margin -->
					</div>
				</Tooltip.Trigger>
				<Tooltip.Content sideOffset={14}>
					<div class="bg-[var(--color-menu-background)]/70 bg-background shadow-popover backdrop-blur-md border border-zinc-500 rounded-xl px-3 py-2 text-xs text-zinc-300">
						Filter by time range
					</div>
				</Tooltip.Content>
			</Tooltip.Root>
		</Tooltip.Provider>

		<!-- Reset Button, outside the time/slider box, with left margin -->
		<Tooltip.Provider>
			<Tooltip.Root delayDuration={200}>
				<Tooltip.Trigger asChild>
					<button
						on:click={onReset}
						class="bg-background/80 flex w-full lg:w-10 h-10 items-center justify-center rounded-[9px] border border-zinc-500 text-sm font-medium transition-colors hover:bg-zinc-700/30 focus:outline-none active:scale-95"
						aria-label="Reset all filters"
						type="button"
					>
						<ArrowCounterClockwise class="size-5 text-zinc-300 flex-shrink-0" />
					</button>
				</Tooltip.Trigger>
				<Tooltip.Content sideOffset={14}>
					<div class="bg-[var(--color-menu-background)]/70 bg-background shadow-popover backdrop-blur-md border border-zinc-500 rounded-xl px-3 py-2 text-xs text-zinc-300">
						Reset all filters
					</div>
				</Tooltip.Content>
			</Tooltip.Root>
		</Tooltip.Provider>
	</div>
{/if}
