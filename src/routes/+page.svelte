<script lang="ts">
	import { NostrFetcher } from 'nostr-fetch';
	import type { NostrEvent } from 'nostr-fetch';
	import * as nip19 from 'nostr-tools/nip19';
	import { insertEventIntoDescendingList } from 'nostr-tools/utils';
	import { onMount } from 'svelte';

	let publicKey = $state('');
	let events = $state<NostrEvent[]>([]);
	let profileEvents = $state<Map<string, NostrEvent>>(new Map());
	let relayListEvent = $state<NostrEvent | null>(null);
	let beforeLoad = $state(true);
	let loading = $state(false);
	let error = $state<string | null>(null);
	let searchQuery = $state('');
	let controller = new AbortController();

	function getProfile(pubkey: string): { picture?: string; name?: string } | null {
		const profileEvent = profileEvents.get(pubkey);
		if (!profileEvent || profileEvent.kind !== 0) {
			return null;
		}

		try {
			const profile = JSON.parse(profileEvent.content);
			return {
				picture: profile.picture,
				name: profile.name
			};
		} catch {
			return null;
		}
	}

	let filteredEvents = $derived.by(() => {
		const kind1Events = events.filter((event) => event.kind === 1);

		if (!searchQuery.trim()) {
			return kind1Events;
		}

		const query = searchQuery.trim().toLowerCase();
		return kind1Events.filter((event) => event.content.toLowerCase().includes(query));
	});

	const defaultRelays = ['wss://relay.damus.io', 'wss://nos.lol', 'wss://yabu.me/'];
	const metadataRelays = [
		'wss://purplepag.es/',
		'wss://user.kindpag.es/',
		'wss://directory.yabu.me/'
	];

	function normalizePubkey(input: string): string {
		const trimmed = input.trim();

		if (/^[0-9a-fA-F]{64}$/.test(trimmed)) {
			return trimmed;
		}

		if (trimmed.startsWith('npub')) {
			try {
				const decoded = nip19.decode(trimmed);
				if (decoded.type === 'npub') {
					return decoded.data as string;
				}
			} catch {
				throw new Error('無効なnpub形式です');
			}
		}

		throw new Error('公開鍵はnpub形式またはhex形式（64文字）で入力してください');
	}

	function extractWriteRelays(event: NostrEvent | null): string[] {
		if (!event || event.kind !== 10002) {
			return defaultRelays;
		}

		const writeRelays = event.tags
			.filter(
				(tag) =>
					tag[0] === 'r' && tag[1].startsWith('wss://') && URL.canParse(tag[1]) && tag[2] !== 'read'
			)
			.map((tag) => tag[1]);

		return writeRelays.length > 0 ? writeRelays : defaultRelays;
	}

	async function fetchEvents() {
		if (!publicKey.trim()) {
			error = '公開鍵を入力してください';
			return;
		}

		beforeLoad = false;
		loading = true;
		error = null;
		events = [];
		profileEvents = new Map();
		relayListEvent = null;
		searchQuery = '';

		const fetcher = NostrFetcher.init();

		try {
			const pubkey = normalizePubkey(publicKey);

			const relayListIterator = fetcher.allEventsIterator(
				metadataRelays,
				{
					authors: [pubkey],
					kinds: [0, 10002]
				},
				{},
				{ limitPerReq: 500 }
			);

			for await (const event of relayListIterator) {
				if (event.kind === 0) {
					profileEvents = new Map(profileEvents);
					const existing = profileEvents.get(event.pubkey);
					if (!existing || event.created_at > existing.created_at) {
						profileEvents.set(event.pubkey, event);
					}
				} else if (event.kind === 10002) {
					if (!relayListEvent || event.created_at > relayListEvent.created_at) {
						relayListEvent = event;
					}
				}
			}

			const writeRelays = extractWriteRelays(relayListEvent);
			console.log('[write relays]', writeRelays);

			const eventIterator = fetcher.allEventsIterator(
				writeRelays,
				{
					authors: [pubkey],
					kinds: [1]
				},
				{},
				{ limitPerReq: 500, signal: controller.signal }
			);

			for await (const event of eventIterator) {
				events = insertEventIntoDescendingList(events, event);
			}
		} catch (err) {
			error = err instanceof Error ? err.message : '投稿の取得中にエラーが発生しました';
			console.error('Error fetching events:', err);
		} finally {
			await fetcher.shutdown();
			loading = false;
		}
	}

	function abort() {
		controller.abort();
		controller = new AbortController();
	}

	function formatDate(timestamp: number): string {
		return new Date(timestamp * 1000).toLocaleString('ja-JP');
	}

	function splitTextForHighlight(
		text: string,
		query: string
	): Array<{ text: string; isMatch: boolean }> {
		if (!query.trim()) {
			return [{ text, isMatch: false }];
		}

		const queryLower = query.trim().toLowerCase();
		const textLower = text.toLowerCase();
		const parts: Array<{ text: string; isMatch: boolean }> = [];
		let lastIndex = 0;

		let index = textLower.indexOf(queryLower, lastIndex);
		while (index !== -1) {
			if (index > lastIndex) {
				parts.push({ text: text.slice(lastIndex, index), isMatch: false });
			}
			parts.push({ text: text.slice(index, index + query.length), isMatch: true });
			lastIndex = index + query.length;
			index = textLower.indexOf(queryLower, lastIndex);
		}

		if (lastIndex < text.length) {
			parts.push({ text: text.slice(lastIndex), isMatch: false });
		}

		return parts.length > 0 ? parts : [{ text, isMatch: false }];
	}

	function isValidImageUrl(url: string | undefined): boolean {
		return typeof url === 'string' && URL.canParse(url) && new URL(url).protocol === 'https:';
	}

	async function login() {
		// @ts-ignore
		const pubkey = (await window.nostr?.getPublicKey()) as string;
		if (pubkey) {
			publicKey = pubkey;
		}
	}

	onMount(async () => {
		const { init } = await import('nostr-login');
		init({ darkMode: false });
	});
</script>

<div class="container mx-auto max-w-4xl p-6">
	<h1 class="mb-6 text-3xl font-bold">Nostr 投稿検索</h1>

	<div class="mb-6">
		<label for="pubkey" class="mb-2 block text-sm font-medium text-gray-700">
			<span>公開鍵 (pubkey)</span>
			<button onclick={login} class="rounded-md bg-blue-600 px-6 py-2 text-white hover:bg-blue-700">
				ログイン
			</button>
		</label>
		<div class="flex gap-2">
			<input
				type="text"
				id="pubkey"
				bind:value={publicKey}
				placeholder="npub1... または hex形式の公開鍵"
				class="flex-1 rounded-md border border-gray-300 px-4 py-2 focus:ring-2 focus:ring-blue-500 focus:outline-none"
				disabled={loading}
			/>
			<button
				onclick={fetchEvents}
				disabled={loading || !publicKey.trim()}
				class="rounded-md bg-blue-600 px-6 py-2 text-white hover:bg-blue-700 disabled:cursor-not-allowed disabled:bg-gray-400"
			>
				{loading ? '取得中...' : '取得'}
			</button>
		</div>
	</div>

	{#if error}
		<div class="mb-4 rounded-md border border-red-400 bg-red-100 p-4 text-red-700">
			{error}
		</div>
	{/if}

	{#if loading && events.length === 0}
		<div class="py-8 text-center">
			<div class="inline-block h-8 w-8 animate-spin rounded-full border-b-2 border-blue-600"></div>
			<p class="mt-2 text-gray-600">投稿を取得しています...</p>
		</div>
	{/if}

	{#if events.length > 0 || loading}
		<div class="mb-6">
			<label for="search" class="mb-2 block text-sm font-medium text-gray-700"> 検索 </label>
			<input
				type="text"
				id="search"
				bind:value={searchQuery}
				placeholder="投稿の内容で検索..."
				class="w-full rounded-md border border-gray-300 px-4 py-2 focus:ring-2 focus:ring-blue-500 focus:outline-none"
			/>
		</div>

		<div class="mb-4 flex items-center gap-2 text-sm text-gray-600">
			{#if loading}
				<div
					class="inline-block h-4 w-4 animate-spin rounded-full border-2 border-gray-400 border-t-transparent"
				></div>
			{/if}
			<span>
				{#if searchQuery.trim()}
					{filteredEvents.length} / {events.length} 件の投稿が一致しました
				{:else}
					{events.length} 件の投稿が見つかりました
				{/if}
				{#if loading}
					<span class="text-gray-400">（取得中...）</span>
					<button
						onclick={abort}
						class="rounded-md bg-blue-600 px-6 py-2 text-white hover:bg-blue-700"
					>
						中止
					</button>
				{/if}
			</span>
		</div>
		<div class="space-y-4">
			{#each filteredEvents as event (event.id)}
				{@const profile = getProfile(event.pubkey)}
				<div class="rounded-lg border border-gray-200 p-4 transition-shadow hover:shadow-md">
					<div class="mb-2 flex items-start gap-3">
						{#if profile?.picture && isValidImageUrl(profile.picture)}
							<img
								src={profile.picture}
								alt={profile.name || 'アイコン'}
								class="h-10 w-10 flex-shrink-0 rounded-full object-cover"
								loading="lazy"
								referrerpolicy="no-referrer"
								onerror={(e) => {
									(e.target as HTMLImageElement).style.display = 'none';
								}}
							/>
						{:else}
							<div
								class="flex h-10 w-10 flex-shrink-0 items-center justify-center rounded-full bg-gray-200 text-gray-500"
							>
								<svg
									xmlns="http://www.w3.org/2000/svg"
									class="h-6 w-6"
									fill="none"
									viewBox="0 0 24 24"
									stroke="currentColor"
								>
									<path
										stroke-linecap="round"
										stroke-linejoin="round"
										stroke-width="2"
										d="M16 7a4 4 0 11-8 0 4 4 0 018 0zM12 14a7 7 0 00-7 7h14a7 7 0 00-7-7z"
									/>
								</svg>
							</div>
						{/if}
						<div class="flex-1">
							<div class="mb-2 flex items-start justify-between">
								<div class="text-xs text-gray-500">
									{#if profile?.name}
										<div class="mb-1 font-medium text-gray-700">{profile.name}</div>
									{/if}
								</div>
								<div class="text-xs text-gray-500">
									<a
										href="https://nostter.app/{nip19.neventEncode({ id: event.id })}"
										target="_blank"
										rel="noopener noreferrer"
									>
										{formatDate(event.created_at)}
									</a>
								</div>
							</div>
							<div class="text-sm break-words whitespace-pre-wrap text-gray-700">
								{#each splitTextForHighlight(event.content, searchQuery.trim()) as part}
									{#if part.isMatch}
										<mark class="bg-yellow-200">{part.text}</mark>
									{:else}
										{part.text}
									{/if}
								{/each}
							</div>
						</div>
					</div>
				</div>
			{/each}
		</div>
		{#if searchQuery.trim() && filteredEvents.length === 0}
			<div class="py-8 text-center text-gray-500">
				「{searchQuery}」に一致する投稿が見つかりませんでした
			</div>
		{/if}
	{:else if !beforeLoad && !loading && !error && publicKey}
		<div class="py-8 text-center text-gray-500">投稿が見つかりませんでした</div>
	{/if}
</div>
