<script lang="ts">
	import Store from '$lib/components/utils/Store.svelte';
	import type { Readable } from 'svelte/store';
	import { Icon, type IconSource } from 'svelte-hero-icons';

	export let href: string;
	export let name: Readable<string>;
	export let icon: IconSource | null = null;
</script>

<a
	{href}
	class="
		relative flex min-w-[min(8rem,100%)] flex-1 items-center justify-center gap-2 px-2 py-2 text-black
		after:absolute after:bottom-0 after:left-0 after:h-0.5 after:w-0 after:bg-black after:transition-[width]
		hover:after:w-full dark:text-white after:dark:bg-white
	"
>
	{#if icon}
		<Icon src={icon} class="h-6 w-6 min-w-6" mini />
	{/if}
	<Store store={name} />
</a>
