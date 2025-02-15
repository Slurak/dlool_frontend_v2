<script lang="ts" context="module">
	function handleMarkdownInsertion(ele: HTMLTextAreaElement, value: string, func: InsertionFunc) {
		const { selectionStart, selectionEnd } = ele;

		const [newValue, offset1, offset2] = func(value, selectionStart, selectionEnd);
		ele.focus();

		asyncRequestAnimationFrame().then(() =>
			ele.setSelectionRange(selectionStart + offset1, selectionEnd + offset2)
		);
		return newValue;
	}
</script>

<script lang="ts">
	import { type IconSource } from 'svelte-hero-icons';
	import type { Readable } from 'svelte/store';
	import Frame from './Frame.svelte';
	import Line from './Area/Line.svelte';
	import Inner from './Area/Inner.svelte';
	import Markdown from '../markdown/Markdown.svelte';
	import {
		insertBold,
		insertHeading,
		insertItalic,
		insertLink,
		type InsertionFunc
	} from '$lib/utils/markdown/generate';
	import { i } from '$lib/i18n/store';
	import Store from '../utils/Store.svelte';
	import { createEventDispatcher } from 'svelte';
	import { diceCoefficient } from 'dice-coefficient';
	import type { Gemoji } from 'gemoji';
	import { getCursorPixelPosition } from '$lib/utils/dom/cursor';
	import EmojiSelector from '../emoji/EmojiSelector.svelte';
	import { asyncRequestAnimationFrame } from '$lib/utils/dom';
	import { prettify } from '$lib/utils/strings/prettify';

	export let placeholder: Readable<string>;
	export let icon: IconSource | null = null;

	export let markdown = false;
	export let prettifiying = true;

	export let value: string = '';

	export let isValid: boolean | null = null;
	export let rows = 2;

	export let emoji = false;

	let ele: HTMLTextAreaElement | undefined;

	let viewMode: 'edit' | 'preview' = 'edit';

	const dispatch = createEventDispatcher<{
		input: string;
	}>();

	let emojiRecommendations: Gemoji[] = [];
	let emojiPopupX = 0;
	let emojiPopupY = 0;
</script>

<Frame>
	<div class="flex w-full flex-col items-stretch gap-2">
		{#if markdown}
			<Line
				bind:viewMode
				on:heading={() => {
					if (!ele) return;
					value = handleMarkdownInsertion(ele, value, insertHeading);
				}}
				on:bold={() => {
					if (!ele) return;
					value = handleMarkdownInsertion(ele, value, insertBold);
				}}
				on:italic={() => {
					if (!ele) return;
					value = handleMarkdownInsertion(ele, value, insertItalic);
				}}
				on:link={() => {
					if (!ele) return;
					value = handleMarkdownInsertion(ele, value, insertLink);
				}}
			/>

			<hr class="h-1 w-full border-zinc-300 dark:border-zinc-700" />
		{/if}

		{#if viewMode === 'edit'}
			<Inner
				bind:ele
				{placeholder}
				{icon}
				{isValid}
				{rows}
				bind:value
				on:input={async ({ detail }) => {
					dispatch('input', detail);
					if (!emoji) return;
					if (!ele) return;

					const selection = ele?.selectionStart;
					if (selection === undefined) return;

					const textUntilSelection = detail.slice(0, selection);
					const isEmoji = /(\s|^):\w+:?$/.test(textUntilSelection);
					if (!isEmoji) {
						emojiRecommendations = [];
						return;
					}

					const emojiSearch = textUntilSelection
						.replaceAll(':', ' ')
						.split(/\s+/)
						.filter((x) => x)
						.at(-1);
					if (!emojiSearch) return;

					const gemoji = await import('gemoji').then(({ gemoji }) => gemoji);

					const sortedGemoji = gemoji
						.map((ge) => ({
							...ge,
							distance: Math.max(...ge.names.map((name) => diceCoefficient(name, emojiSearch)))
						}))
						.sort(({ distance: a }, { distance: b }) => b - a)
						.slice(0, 5)
						.filter(({ distance }) => distance > 0.375);

					emojiRecommendations = sortedGemoji;

					const { x, y } = getCursorPixelPosition(ele);
					emojiPopupY = y;
					emojiPopupX = x;
				}}
				on:keydown={(e) => {
					const { key, ctrlKey } = e;

					if (!ele) return;
					if (!ctrlKey) return;

					switch (key) {
						case 'b':
							e.preventDefault();
							value = handleMarkdownInsertion(ele, value, insertBold);
							return;
						case 'i':
							e.preventDefault();
							value = handleMarkdownInsertion(ele, value, insertItalic);
							return;
						case 'l':
							e.preventDefault();
							value = handleMarkdownInsertion(ele, value, insertLink);
							return;
						case 'h':
							e.preventDefault();
							value = handleMarkdownInsertion(ele, value, insertHeading);
							return;
						default:
							console.log(key);
					}
				}}
			>
				<slot slot="pre-validator" name="pre-validator" />
				<slot slot="post-validator" name="post-validator" />
				<div
					class=" absolute left-[--left] top-[--top]"
					style:--top={`${emojiPopupY}px`}
					style:--left={`${emojiPopupX}px`}
				>
					{#if emoji && emojiRecommendations.length > 0}
						<EmojiSelector
							emoji={emojiRecommendations}
							on:select={async ({ detail: emoji }) => {
								if (!ele) return;
								const start = value.slice(0, ele.selectionStart).replace(/:\w+:?$/, emoji.emoji);
								value = start + value.slice(ele.selectionStart);
								await asyncRequestAnimationFrame();
								ele.setSelectionRange(start.length, start.length);
								ele.focus();
							}}
						/>
					{/if}
				</div>
			</Inner>
		{:else if viewMode === 'preview'}
			{#if value.trim() === ''}
				<span class="w-full text-center text-gray-600 dark:text-gray-400">
					<Store store={i('markdownpreview.noPreview')} />
				</span>
			{:else}
				<Markdown markdown={prettifiying ? prettify(value) : value} />
			{/if}
		{/if}
	</div>
</Frame>
