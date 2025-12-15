<script lang="ts">
	import axios from 'axios';
	import { onMount, tick } from 'svelte';
	import { env } from '$env/dynamic/public';
	const TMDB_KEY = env.PUBLIC_TMDB_KEY;
	const WATCHMODE_KEY = env.PUBLIC_WATCHMODE_KEY;
	const RAPID_KEY = env.PUBLIC_RAPIDAPI_KEY;
	const RAPID_HOST = env.PUBLIC_RAPIDAPI_HOST;
	interface RawSource {
		name: string;
		type: 'sub' | 'rent' | 'buy' | 'free' | 'cinema';
		quality: string;
		price: number;
		url: string;
		logo?: string;
	}
	interface ServiceOption {
		type: 'sub' | 'rent' | 'buy' | 'free' | 'cinema';
		qualities: string[];
		priceDisplay: string;
	}
	interface GroupedSource {
		id: string;
		name: string;
		options: ServiceOption[];
		url: string;
		logo: string;
	}
	interface SearchResult {
		id: number;
		media_type: string;
		title?: string;
		name?: string;
		poster_path?: string;
		release_date?: string;
		first_air_date?: string;
		overview?: string;
	}
	interface Movie extends SearchResult {
		year?: string;
		inTheaters?: boolean;
		sources: GroupedSource[];
		overview: string;
	}
	interface WatchmodeItem {
		name: string;
		type: 'sub' | 'rent' | 'buy' | 'free';
		format: string;
		price: number | null;
		web_url: string;
		logo_100px: string;
	}
	interface RapidAPIItem {
		service: { name: string; imageSet?: { lightThemeImage?: string } };
		type: 'subscription' | 'rent' | 'buy' | 'free';
		quality: string;
		price: { amount: string } | null;
		link: string;
	}
	interface TMDBProvider {
		provider_id: number;
		provider_name: string;
		logo_path: string;
	}
	let region = 'US';
	let query = '';
	let searchBuffer: SearchResult[] = [];
	let searchResults: SearchResult[] = [];
	let isSearchFocused = false;
	let searchLoading = false;
	let searchError = '';
	let apiPage = 1;
	let searchTimer: ReturnType<typeof setTimeout>;
	let activeSearchIndex = -1;
	let selectedMovie: Movie | null = null;
	let movieLoading = false;
	let movieError = '';
	let activeTab: 'all' | 'free' | 'paid' = 'all';
	let showPosterModal = false;
	let isRegionOpen = false;
	let regionSearch = '';
	let regionContainer: HTMLDivElement;
	let searchInput: HTMLInputElement;
	let resultsContainer: HTMLDivElement;
	let regionListContainer: HTMLDivElement;
	let searchWrapper: HTMLDivElement;
	let searchCache: Record<string, SearchResult[]> = {};
	let movieCache: Record<string, Movie> = {};
	let availableProviders: TMDBProvider[] = [];
	const countryMap = [
		{ code: 'AU', name: 'Australia', currency: 'A$', flag: '🇦🇺' },
		{ code: 'BR', name: 'Brazil', currency: 'R$', flag: '🇧🇷' },
		{ code: 'CA', name: 'Canada', currency: 'C$', flag: '🇨🇦' },
		{ code: 'DK', name: 'Denmark', currency: 'kr', flag: '🇩🇰' },
		{ code: 'FI', name: 'Finland', currency: '€', flag: '🇫🇮' },
		{ code: 'FR', name: 'France', currency: '€', flag: '🇫🇷' },
		{ code: 'DE', name: 'Germany', currency: '€', flag: '🇩🇪' },
		{ code: 'IN', name: 'India', currency: '₹', flag: '🇮🇳' },
		{ code: 'IE', name: 'Ireland', currency: '€', flag: '🇮🇪' },
		{ code: 'IT', name: 'Italy', currency: '€', flag: '🇮🇹' },
		{ code: 'JP', name: 'Japan', currency: '¥', flag: '🇯🇵' },
		{ code: 'MX', name: 'Mexico', currency: 'Mex$', flag: '🇲🇽' },
		{ code: 'NL', name: 'Netherlands', currency: '€', flag: '🇳🇱' },
		{ code: 'NZ', name: 'New Zealand', currency: '$', flag: '🇳🇿' },
		{ code: 'NO', name: 'Norway', currency: 'kr', flag: '🇳🇴' },
		{ code: 'PL', name: 'Poland', currency: 'zł', flag: '🇵🇱' },
		{ code: 'PT', name: 'Portugal', currency: '€', flag: '🇵🇹' },
		{ code: 'RU', name: 'Russia', currency: '₽', flag: '🇷🇺' },
		{ code: 'SG', name: 'Singapore', currency: 'S$', flag: '🇸🇬' },
		{ code: 'ZA', name: 'South Africa', currency: 'R', flag: '🇿🇦' },
		{ code: 'KR', name: 'South Korea', currency: '₩', flag: '🇰🇷' },
		{ code: 'ES', name: 'Spain', currency: '€', flag: '🇪🇸' },
		{ code: 'SE', name: 'Sweden', currency: 'kr', flag: '🇸🇪' },
		{ code: 'TR', name: 'Turkey', currency: '₺', flag: '🇹🇷' },
		{ code: 'GB', name: 'United Kingdom', currency: '£', flag: '🇬🇧' },
		{ code: 'US', name: 'United States', currency: '$', flag: '🇺🇸' }
	];
	$: filteredCountries = countryMap.filter(
		(c) =>
			c.name.toLowerCase().includes(regionSearch.toLowerCase()) ||
			c.code.toLowerCase().includes(regionSearch.toLowerCase())
	);
	onMount(() => {
		if (searchInput) searchInput.focus();
		searchCache = {};
		movieCache = {};
		fetchAllProviders();
		document.addEventListener('click', handleClickOutside);
		document.addEventListener('keydown', handleGlobalKeydown);
		return () => {
			document.removeEventListener('click', handleClickOutside);
			document.removeEventListener('keydown', handleGlobalKeydown);
		};
	});
	async function fetchAllProviders() {
		try {
			const [movieRes, tvRes] = await Promise.all([
				axios.get(
					`https://api.themoviedb.org/3/watch/providers/movie?api_key=${TMDB_KEY}&language=en-US`
				),
				axios.get(
					`https://api.themoviedb.org/3/watch/providers/tv?api_key=${TMDB_KEY}&language=en-US`
				)
			]);
			const rawObj: Record<number, TMDBProvider> = {};
			[...(movieRes.data.results || []), ...(tvRes.data.results || [])].forEach((p) => {
				if (!rawObj[p.provider_id]) {
					rawObj[p.provider_id] = p;
				}
			});
			availableProviders = Object.values(rawObj);
		} catch {
			return;
		}
	}
	function findBestMatchLogo(serviceName: string, remoteLogo?: string): string {
		if (remoteLogo && remoteLogo.includes('http')) return remoteLogo;
		const target = serviceName.toLowerCase().replace(/[^a-z0-9]/g, '');
		let bestMatch: string | null = null;
		let bestScore = 0;
		for (const p of availableProviders) {
			const current = p.provider_name.toLowerCase().replace(/[^a-z0-9]/g, '');
			if (current === target) return `https://image.tmdb.org/t/p/w154${p.logo_path}`;
			let score = 0;
			if (current.includes(target) || target.includes(current)) score += 0.5;
			const targetWords = serviceName.toLowerCase().split(' ');
			const currentWords = p.provider_name.toLowerCase().split(' ');
			const intersection = targetWords.filter((w) => currentWords.includes(w));
			score += (intersection.length / Math.max(targetWords.length, currentWords.length)) * 0.5;
			if (score > bestScore) {
				bestScore = score;
				bestMatch = p.logo_path;
			}
		}
		if (bestScore > 0.3 && bestMatch) {
			return `https://image.tmdb.org/t/p/w154${bestMatch}`;
		}
		return '';
	}
	function handleClickOutside(event: MouseEvent) {
		if (regionContainer && !regionContainer.contains(event.target as Node)) {
			isRegionOpen = false;
		}
		if (searchWrapper && !searchWrapper.contains(event.target as Node)) {
			isSearchFocused = false;
		}
	}
	function handleGlobalKeydown(e: KeyboardEvent) {
		if (e.key === 'Escape') {
			isSearchFocused = false;
			isRegionOpen = false;
			showPosterModal = false;
			if (searchInput) searchInput.blur();
		}
	}
	function resetHome() {
		selectedMovie = null;
		query = '';
		isSearchFocused = false;
		if (searchInput) {
			setTimeout(() => searchInput.focus(), 50);
		}
	}
	async function toggleRegion() {
		isRegionOpen = !isRegionOpen;
		if (isRegionOpen) {
			await tick();
			if (regionListContainer) {
				const selected = regionListContainer.querySelector('[data-selected="true"]') as HTMLElement;
				if (selected) {
					regionListContainer.scrollTop = selected.offsetTop - regionListContainer.offsetTop;
				}
			}
		}
	}
	function selectRegion(code: string) {
		region = code;
		isRegionOpen = false;
		if (selectedMovie) {
			selectMovie(selectedMovie, true);
		}
	}
	function toTitleCase(str: string): string {
		const minorWords = new Set([
			'a',
			'an',
			'the',
			'and',
			'but',
			'or',
			'nor',
			'if',
			'as',
			'yet',
			'so',
			'at',
			'by',
			'for',
			'from',
			'in',
			'into',
			'of',
			'on',
			'to',
			'up',
			'with',
			'via',
			'off',
			'out'
		]);
		return str.replace(/\w\S*/g, (txt, index) => {
			const lower = txt.toLowerCase();
			if (index !== 0 && minorWords.has(lower) && lower.length <= 3) return lower;
			return txt.charAt(0).toUpperCase() + txt.substr(1).toLowerCase();
		});
	}
	function handleInput(e: Event) {
		const target = e.target as HTMLInputElement;
		query = toTitleCase(target.value);
		activeSearchIndex = -1;
		clearTimeout(searchTimer);
		if (!query.trim()) {
			searchBuffer = [];
			searchResults = [];
			searchError = '';
			searchLoading = false;
			apiPage = 1;
			return;
		}
		if (searchCache[query]) {
			searchBuffer = searchCache[query];
			searchResults = searchBuffer.slice(0, 10);
			return;
		}
		searchLoading = true;
		apiPage = 1;
		searchBuffer = [];
		searchResults = [];
		searchTimer = setTimeout(() => fetchSearchResults(1), 350);
	}
	function handleSearchKeydown(e: KeyboardEvent) {
		if (!searchResults.length) return;
		if (e.key === 'ArrowDown') {
			e.preventDefault();
			activeSearchIndex = (activeSearchIndex + 1) % searchResults.length;
			scrollToActiveItem();
		} else if (e.key === 'ArrowUp') {
			e.preventDefault();
			activeSearchIndex = (activeSearchIndex - 1 + searchResults.length) % searchResults.length;
			scrollToActiveItem();
		} else if (e.key === 'Enter') {
			e.preventDefault();
			if (activeSearchIndex >= 0 && searchResults[activeSearchIndex]) {
				selectMovie(searchResults[activeSearchIndex]);
				if (searchInput) searchInput.blur();
			} else {
				selectMovie(searchResults[0]);
			}
		}
	}
	function scrollToActiveItem() {
		if (!resultsContainer) return;
		const items = resultsContainer.querySelectorAll('button');
		if (items[activeSearchIndex]) {
			items[activeSearchIndex].scrollIntoView({ block: 'nearest' });
		}
	}
	async function fetchSearchResults(page: number) {
		try {
			const url = `https://api.themoviedb.org/3/search/multi?api_key=${TMDB_KEY}&query=${encodeURIComponent(query)}&include_adult=false&page=${page}`;
			const res = await axios.get(url);
			const newItems = (res.data.results as SearchResult[]).filter(
				(m) => m.media_type === 'movie' || m.media_type === 'tv'
			);
			if (newItems.length === 0 && page === 1) {
				searchError = 'No results found.';
				searchResults = [];
			} else {
				// Deduplicate before adding to buffer
				const existingIds = new Set(searchBuffer.map((i) => i.id));
				const uniqueNew = newItems.filter((i) => !existingIds.has(i.id));

				searchBuffer = [...searchBuffer, ...uniqueNew];
				if (page === 1) searchCache[query] = searchBuffer;

				const currentCount = searchResults.length;
				const nextBatchSize = currentCount + 10;
				searchResults = searchBuffer.slice(
					0,
					page === 1 && currentCount === 0 ? 10 : nextBatchSize
				);
			}
		} catch {
			if (page === 1) searchError = 'Error connecting to movie database.';
		} finally {
			searchLoading = false;
		}
	}
	function handleScroll(e: Event) {
		const target = e.target as HTMLDivElement;
		if (target.scrollHeight - target.scrollTop <= target.clientHeight + 100 && !searchLoading) {
			if (searchResults.length < searchBuffer.length) {
				searchLoading = true;
				setTimeout(() => {
					searchResults = searchBuffer.slice(0, searchResults.length + 10);
					searchLoading = false;
				}, 100);
			} else {
				searchLoading = true;
				apiPage++;
				fetchSearchResults(apiPage);
			}
		}
	}
	async function selectMovie(movie: SearchResult, forceRefresh = false) {
		isSearchFocused = false;
		movieLoading = true;
		movieError = '';
		activeSearchIndex = -1;
		const cacheKey = `${movie.id}-${region}`;
		if (!forceRefresh && movieCache[cacheKey]) {
			selectedMovie = movieCache[cacheKey];
			movieLoading = false;
			return;
		}
		selectedMovie = {
			...movie,
			overview: movie.overview || 'No description available.',
			sources: [],
			inTheaters: false,
			year: movie.release_date?.split('-')[0] || movie.first_air_date?.split('-')[0]
		} as Movie;
		try {
			if (movie.media_type === 'movie') {
				const nowPlayingUrl = `https://api.themoviedb.org/3/movie/now_playing?api_key=${TMDB_KEY}&region=${region}`;
				const res = await axios.get(nowPlayingUrl);
				if (selectedMovie) {
					selectedMovie.inTheaters = (res.data.results as SearchResult[]).some(
						(m) => m.id === movie.id
					);
				}
			}
			let rawSources = await fetchFromWatchmode(movie, 'id');
			if (rawSources.length === 0) rawSources = await fetchFromWatchmode(movie, 'name');
			if (rawSources.length === 0) rawSources = await fetchFromRapidAPI(movie);
			if (selectedMovie) {
				selectedMovie.sources = groupSources(rawSources);
				activeTab = 'all';
				movieCache[cacheKey] = selectedMovie;
			}
		} catch (e) {
			console.error(e);
			movieError = 'Could not fetch viewing options.';
		} finally {
			movieLoading = false;
		}
	}
	async function fetchFromWatchmode(
		movie: SearchResult,
		mode: 'id' | 'name'
	): Promise<RawSource[]> {
		try {
			const type = movie.media_type === 'movie' ? 'movie' : 'tv';
			let searchRes;
			if (mode === 'id') {
				const searchUrl = `https://api.watchmode.com/v1/search/?apiKey=${WATCHMODE_KEY}&search_field=tmdb_${type}_id&search_value=${movie.id}`;
				searchRes = await axios.get(searchUrl);
			} else {
				const searchUrl = `https://api.watchmode.com/v1/search/?apiKey=${WATCHMODE_KEY}&search_field=name&search_value=${encodeURIComponent(movie.title || movie.name || '')}&types=${type}`;
				searchRes = await axios.get(searchUrl);
			}
			if (!searchRes.data.title_results || searchRes.data.title_results.length === 0) return [];
			const wmId = searchRes.data.title_results[0].id;
			const sourcesUrl = `https://api.watchmode.com/v1/title/${wmId}/sources/?apiKey=${WATCHMODE_KEY}&regions=${region}`;
			const sourcesRes = await axios.get(sourcesUrl);
			return (sourcesRes.data as WatchmodeItem[]).map((s) => ({
				name: s.name,
				type: s.type,
				quality: s.format,
				price: s.price || 0,
				url: s.web_url,
				logo: s.logo_100px
			}));
		} catch {
			return [];
		}
	}
	async function fetchFromRapidAPI(movie: SearchResult): Promise<RawSource[]> {
		try {
			const options = {
				method: 'GET',
				url: 'https://streaming-availability.p.rapidapi.com/shows/search/title',
				params: {
					country: region.toLowerCase(),
					title: movie.title || movie.name,
					show_type: movie.media_type === 'movie' ? 'movie' : 'series',
					output_language: 'en'
				},
				headers: { 'x-rapidapi-key': RAPID_KEY, 'x-rapidapi-host': RAPID_HOST }
			};
			const response = await axios.request(options);
			if (!response.data || response.data.length === 0) return [];
			const match = response.data[0];
			if (!match || !match.streamingOptions || !match.streamingOptions[region.toLowerCase()])
				return [];
			const rawOptions = match.streamingOptions[region.toLowerCase()] as RapidAPIItem[];
			const normalized: RawSource[] = [];
			rawOptions.forEach((opt) => {
				normalized.push({
					name: opt.service.name,
					type: opt.type === 'subscription' ? 'sub' : opt.type,
					quality: opt.quality,
					price: opt.price ? parseFloat(opt.price.amount) : 0,
					url: opt.link,
					logo: opt.service.imageSet?.lightThemeImage || ''
				});
			});
			return normalized;
		} catch {
			return [];
		}
	}
	function groupSources(raw: RawSource[]): GroupedSource[] {
		const groups: Record<
			string,
			{
				name: string;
				url: string;
				logo: string;
				options: Record<
					string,
					{
						type: 'sub' | 'rent' | 'buy' | 'free' | 'cinema';
						qualities: Set<string>;
						prices: Set<number>;
					}
				>;
			}
		> = {};
		raw.forEach((src) => {
			const key = src.name;
			if (!groups[key]) {
				const logo = findBestMatchLogo(src.name, src.logo);
				groups[key] = {
					name: src.name,
					url: src.url,
					logo,
					options: {}
				};
			}
			const g = groups[key];
			const typeKey = src.type;
			if (!g.options[typeKey]) {
				g.options[typeKey] = { type: src.type, qualities: new Set(), prices: new Set() };
			}
			if (src.quality) g.options[typeKey].qualities.add(src.quality);
			if (src.price) g.options[typeKey].prices.add(src.price);
		});
		const currentCountry = countryMap.find((c) => c.code === region);
		const symbol = currentCountry ? currentCountry.currency : '$';
		return Object.entries(groups).map(([key, g]) => {
			const options = Object.values(g.options).map((opt) => {
				const priceArray = Array.from(opt.prices).sort((a, b) => a - b);
				let priceDisplay = '';
				if (priceArray.length > 0) {
					const min = priceArray[0];
					const max = priceArray[priceArray.length - 1];
					if (max > 0) {
						priceDisplay = min === max ? `${symbol}${min}` : `${symbol}${min}-${symbol}${max}`;
					}
				}
				const qualityOrder = ['SD', 'HD', '4K'];
				const sortedQualities = Array.from(opt.qualities).sort(
					(a, b) => qualityOrder.indexOf(a) - qualityOrder.indexOf(b)
				);
				return {
					type: opt.type,
					qualities: sortedQualities,
					priceDisplay
				};
			});
			return {
				id: key,
				name: g.name,
				url: g.url,
				logo: g.logo,
				options
			};
		});
	}
	function handleImageError(e: Event) {
		const img = e.target as HTMLImageElement;
		img.style.display = 'none';
		if (img.parentElement) {
			img.parentElement.querySelector('.fallback-text')?.classList.remove('hidden');
		}
	}
</script>

<svelte:head>
	<title>Just Watch</title>
	<meta name="description" content="Find where to watch movies and TV shows." />
</svelte:head>
<main class="min-h-screen bg-[#0a0a0a] font-sans text-neutral-300 selection:bg-neutral-700">
	<div class="mx-auto max-w-3xl px-6 py-12">
		<header class="mb-10 text-center">
			<button
				on:click={resetHome}
				class="group relative inline-block transition-transform duration-300 hover:scale-[1.02]"
			>
				<h1
					class="bg-gradient-to-r from-white via-neutral-200 to-neutral-400 bg-clip-text text-5xl font-black tracking-tighter text-transparent drop-shadow-sm"
				>
					Just Watch
				</h1>
				<div
					class="absolute -bottom-2 left-0 h-[2px] w-full origin-left scale-x-0 bg-gradient-to-r from-neutral-200 to-transparent transition-transform duration-300 group-hover:scale-x-100"
				></div>
			</button>
		</header>
		<div class="relative z-40 mb-10" id="search-wrapper" bind:this={searchWrapper}>
			<div class="group relative flex items-center">
				<div class="pointer-events-none absolute inset-y-0 left-0 flex items-center pl-4">
					<svg
						class="h-5 w-5 text-neutral-500"
						fill="none"
						stroke="currentColor"
						viewBox="0 0 24 24"
						><path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"
						></path></svg
					>
				</div>
				<label for="search" class="sr-only">Search</label>
				<input
					id="search"
					bind:this={searchInput}
					type="text"
					name="search"
					value={query}
					on:input={handleInput}
					on:focus={() => (isSearchFocused = true)}
					on:keydown={handleSearchKeydown}
					autocomplete="off"
					placeholder="Search for a Movie or TV Show"
					class="w-full rounded-xl border border-neutral-800 bg-neutral-900 py-4 pr-40 pl-12 text-lg placeholder-neutral-600 shadow-2xl transition-all focus:border-transparent focus:ring-2 focus:ring-neutral-700 focus:outline-none"
				/>
				<div class="absolute inset-y-0 right-2 z-50 flex items-center" bind:this={regionContainer}>
					<div class="relative">
						<button
							type="button"
							on:click={toggleRegion}
							class="flex items-center gap-2 rounded-lg bg-neutral-800 py-2 pr-2 pl-3 text-sm font-medium text-neutral-400 transition-colors hover:bg-neutral-700"
							aria-label="Select Country"
							aria-haspopup="true"
							aria-expanded={isRegionOpen}
						>
							<span class="max-w-[100px] truncate"
								>{countryMap.find((c) => c.code === region)?.flag} {region}</span
							>
							<svg
								class={`h-4 w-4 transition-transform ${isRegionOpen ? 'rotate-180' : ''}`}
								fill="none"
								stroke="currentColor"
								viewBox="0 0 24 24"
								><path
									stroke-linecap="round"
									stroke-linejoin="round"
									stroke-width="2"
									d="M19 9l-7 7-7-7"
								></path></svg
							>
						</button>
						{#if isRegionOpen}
							<div
								class="absolute top-full right-0 mt-2 w-64 overflow-hidden rounded-xl border border-neutral-700 bg-neutral-800 shadow-xl"
							>
								<div class="border-b border-neutral-700 p-2">
									<input
										type="text"
										placeholder="Search..."
										bind:value={regionSearch}
										class="w-full rounded bg-neutral-900 px-3 py-2 text-sm text-white placeholder-neutral-500 focus:ring-1 focus:ring-neutral-600 focus:outline-none"
										on:click|stopPropagation
										on:input|stopPropagation
									/>
								</div>
								<div
									bind:this={regionListContainer}
									class="scrollbar-thin max-h-60 overflow-y-auto p-1"
								>
									{#each filteredCountries as c (c.code)}
										<button
											type="button"
											class="flex w-full items-center justify-between rounded px-3 py-2 text-left text-sm text-neutral-300 hover:bg-neutral-700 hover:text-white"
											on:click={() => selectRegion(c.code)}
											data-selected={region === c.code}
										>
											<span>{c.flag} {c.name}</span>
											{#if region === c.code}
												<span class="text-blue-400">✓</span>
											{/if}
										</button>
									{/each}
									{#if filteredCountries.length === 0}
										<div class="px-3 py-2 text-xs text-neutral-500">No matches</div>
									{/if}
								</div>
							</div>
						{/if}
					</div>
				</div>
			</div>
			{#if isSearchFocused && (searchResults.length > 0 || searchLoading || searchError)}
				<div
					class="scrollbar-thin scrollbar-thumb-neutral-700 absolute top-full right-0 left-0 z-40 mt-2 max-h-[380px] overflow-y-auto overscroll-contain rounded-xl border border-neutral-800 bg-[#111] shadow-2xl"
					on:scroll={handleScroll}
					bind:this={resultsContainer}
					role="listbox"
					tabindex="-1"
				>
					{#if searchResults.length > 0}
						<div class="p-2">
							{#each searchResults as movie, i (movie.id)}
								<button
									on:click={() => selectMovie(movie)}
									class={`flex w-full items-center gap-4 rounded-lg p-3 text-left transition-colors ${i === activeSearchIndex ? 'bg-neutral-800' : 'hover:bg-neutral-800/60'}`}
									role="option"
									aria-selected={i === activeSearchIndex}
									tabindex="-1"
								>
									<div
										class="relative h-12 w-8 flex-shrink-0 overflow-hidden rounded bg-neutral-800"
									>
										{#if movie.poster_path}
											<img
												src={`https://image.tmdb.org/t/p/w92${movie.poster_path}`}
												alt=""
												class="h-full w-full object-cover"
												loading="lazy"
												on:error={handleImageError}
											/>
											<span
												class="fallback-text absolute inset-0 flex hidden items-center justify-center bg-neutral-700 text-[10px] font-bold text-white"
											>
												{movie.title ? movie.title.charAt(0) : '?'}
											</span>
										{/if}
									</div>
									<div>
										<h2 class="font-bold text-neutral-200">{movie.title || movie.name}</h2>
										<p class="text-xs text-neutral-500">
											{movie.media_type.toUpperCase()} • {movie.release_date?.split('-')[0] ||
												movie.first_air_date?.split('-')[0] ||
												'N/A'}
										</p>
									</div>
								</button>
							{/each}
							{#if searchLoading}
								<div class="p-4 text-center text-sm text-neutral-500">Loading more...</div>
							{/if}
						</div>
					{:else if searchLoading}
						<div class="p-8 text-center text-neutral-500">Searching...</div>
					{:else if searchError}
						<div class="p-4 text-center text-sm text-red-400">{searchError}</div>
					{/if}
				</div>
			{/if}
		</div>
		{#if movieLoading}
			<div class="py-20 text-center">
				<div
					class="inline-block h-8 w-8 animate-spin rounded-full border-2 border-neutral-600 border-t-white"
				></div>
			</div>
		{:else if movieError}
			<div class="text-center text-red-400">{movieError}</div>
		{:else if selectedMovie}
			<div class="animate-in fade-in zoom-in-95 duration-300">
				<div class="mb-10 flex flex-col gap-8 md:flex-row">
					<button
						on:click={() => (showPosterModal = true)}
						class="group relative flex-shrink-0 self-center md:self-start"
					>
						<img
							src={selectedMovie.poster_path
								? `https://image.tmdb.org/t/p/w500${selectedMovie.poster_path}`
								: 'https://placehold.co/300x450/222/999?text=No+Cover'}
							class="w-48 rounded-xl shadow-[0_0_30px_rgba(255,255,255,0.05)] transition-transform duration-300 group-hover:scale-[1.02]"
							alt="Poster"
							loading="lazy"
						/>
						<div
							class="absolute inset-0 flex items-center justify-center rounded-xl bg-black/50 opacity-0 transition-opacity group-hover:opacity-100"
						>
							<span class="text-sm font-bold text-white">View Full</span>
						</div>
					</button>
					<div class="flex-1">
						<h1 class="mb-2 text-4xl font-bold text-white">
							{selectedMovie.name || selectedMovie.title}
						</h1>
						<div class="mb-6 flex items-center gap-3 text-sm text-neutral-400">
							<span class="rounded bg-neutral-800 px-2 py-0.5 text-neutral-300"
								>{selectedMovie.year}</span
							>
							<span>{selectedMovie.media_type === 'movie' ? 'Movie' : 'TV Series'}</span>
						</div>
						<p class="text-lg leading-relaxed text-neutral-300">{selectedMovie.overview}</p>
					</div>
				</div>
				<hr class="mb-8 border-neutral-800" />
				<div class="mb-8 flex gap-6 border-b border-neutral-800 pb-1">
					<button
						class={`pb-3 text-sm font-bold tracking-wider uppercase transition-all ${activeTab === 'all' ? 'border-b-2 border-white text-white' : 'text-neutral-500 hover:text-neutral-300'}`}
						on:click={() => (activeTab = 'all')}
					>
						All
					</button>
					<button
						class={`pb-3 text-sm font-bold tracking-wider uppercase transition-all ${activeTab === 'free' ? 'border-b-2 border-white text-white' : 'text-neutral-500 hover:text-neutral-300'}`}
						on:click={() => (activeTab = 'free')}
						disabled={!selectedMovie.sources.some((s) => s.options.some((o) => o.type === 'free'))}
						class:opacity-30={!selectedMovie.sources.some((s) =>
							s.options.some((o) => o.type === 'free')
						)}
					>
						Free
					</button>
					<button
						class={`pb-3 text-sm font-bold tracking-wider uppercase transition-all ${activeTab === 'paid' ? 'border-b-2 border-white text-white' : 'text-neutral-500 hover:text-neutral-300'}`}
						on:click={() => (activeTab = 'paid')}
						disabled={!selectedMovie.sources.some((s) => s.options.some((o) => o.type !== 'free'))}
						class:opacity-30={!selectedMovie.sources.some((s) =>
							s.options.some((o) => o.type !== 'free')
						)}
					>
						Paid
					</button>
				</div>
				<div class="space-y-3">
					{#if selectedMovie.inTheaters && (activeTab === 'all' || activeTab === 'paid')}
						<div
							class="flex items-center gap-4 rounded-xl border border-yellow-500/30 bg-yellow-500/10 p-4"
						>
							<span class="text-2xl">🎟️</span>
							<div>
								<div class="font-bold text-yellow-200">Available in Theaters</div>
								<div class="text-sm text-yellow-500/70">Check your local cinema listings.</div>
							</div>
						</div>
					{/if}
					{#each selectedMovie.sources as source (source.id)}
						{#if (activeTab === 'all' && source.options.length > 0) || (activeTab === 'free' && source.options.some((o) => o.type === 'free')) || (activeTab === 'paid' && source.options.some((o) => o.type !== 'free'))}
							{#if activeTab === 'all' || (activeTab === 'free' ? source.options.some((o) => o.type === 'free') : source.options.some((o) => o.type !== 'free'))}
								<button
									on:click={() => window.open(source.url, '_blank')}
									class="group flex w-full items-center gap-4 rounded-xl border border-neutral-800/50 bg-neutral-900/30 p-4 text-left transition-all hover:border-neutral-700 hover:bg-[#151515]"
								>
									<div
										class="relative flex h-12 w-12 flex-shrink-0 items-center justify-center overflow-hidden rounded-lg bg-white p-1"
									>
										{#if source.logo}
											<img
												src={source.logo}
												alt={source.name}
												class="relative z-10 h-full w-full object-contain"
												loading="lazy"
												on:error={handleImageError}
											/>
											<span
												class="fallback-text absolute inset-0 z-0 flex hidden items-center justify-center bg-neutral-200 text-xs font-bold text-neutral-800"
											>
												{source.name.substring(0, 1)}
											</span>
										{:else}
											<span
												class="absolute inset-0 flex items-center justify-center bg-neutral-200 text-xs font-bold text-neutral-800"
											>
												{source.name.substring(0, 1)}
											</span>
										{/if}
									</div>
									<div class="flex-1">
										<div class="mb-2 flex items-center gap-2">
											<span class="font-bold text-white">{source.name}</span>
										</div>
										<div class="flex flex-wrap gap-2">
											{#each source.options as opt (opt.type)}
												{#if activeTab === 'all' || (activeTab === 'free' && opt.type === 'free') || (activeTab === 'paid' && opt.type !== 'free')}
													<div
														class="flex flex-col items-start gap-1 rounded border border-neutral-800 bg-neutral-950/50 px-2 py-1.5"
													>
														<div class="flex items-center gap-2">
															<span
																class={`rounded px-1.5 py-0.5 text-[10px] font-bold tracking-wide uppercase 
															${opt.type === 'free' ? 'bg-purple-500/20 text-purple-300' : 'bg-amber-500/20 text-amber-200'}`}
															>
																{opt.type === 'sub' ? 'Stream' : opt.type}
															</span>
															{#if opt.priceDisplay}
																<span class="font-mono text-xs font-bold text-white"
																	>{opt.priceDisplay}</span
																>
															{/if}
														</div>
														<div class="flex flex-wrap gap-1">
															{#each opt.qualities as quality (quality)}
																<span class="text-[9px] text-neutral-500 uppercase"
																	>{quality || 'HD'}</span
																>
															{/each}
														</div>
													</div>
												{/if}
											{/each}
										</div>
									</div>
								</button>
							{/if}
						{/if}
					{/each}
					{#if !selectedMovie.sources.some( (s) => (activeTab === 'all' ? true : activeTab === 'free' ? s.options.some((o) => o.type === 'free') : s.options.some((o) => o.type !== 'free')) ) && !(selectedMovie.inTheaters && activeTab !== 'free')}
						<div class="rounded-xl border border-dashed border-neutral-800 py-12 text-center">
							<p class="text-neutral-500">
								No {activeTab} options found in {countryMap.find((c) => c.code === region)?.name}.
							</p>
						</div>
					{/if}
				</div>
			</div>
		{/if}
		{#if showPosterModal && selectedMovie}
			<div
				class="fixed inset-0 z-[100] flex items-center justify-center bg-black/90 p-4 backdrop-blur-sm"
				on:click={() => (showPosterModal = false)}
				role="dialog"
				tabindex="-1"
				on:keydown={(e) => e.key === 'Escape' && (showPosterModal = false)}
			>
				<img
					src={`https://image.tmdb.org/t/p/original${selectedMovie.poster_path}`}
					class="max-h-full max-w-full rounded-lg shadow-2xl"
					alt="Poster Fullscreen"
					loading="lazy"
				/>
			</div>
		{/if}
	</div>
</main>

<style>
	::-webkit-scrollbar {
		width: 6px;
	}
	::-webkit-scrollbar-track {
		background: #111;
	}
	::-webkit-scrollbar-thumb {
		background: #333;
		border-radius: 3px;
	}
	::-webkit-scrollbar-thumb:hover {
		background: #444;
	}
</style>
