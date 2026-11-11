<script lang="ts">
	import favicon from '#lib/assets/favicon.svg';
	import type { LayoutProps } from './$types';

	let { children }: LayoutProps = $props();
</script>

<svelte:head>
	<title>Auro Consultorio | Demo</title>
	<meta name="description" content="Prototipo de agenda para consultorio con datos ficticios." />
	<link rel="icon" href={favicon} />
</svelte:head>

{@render children()}
