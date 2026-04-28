<script lang="ts">
	import './layout.css';
	import '@fontsource/fira-sans/latin-400.css';
	import '@fontsource/fira-sans/latin-500.css';
	import '@fontsource/fira-sans/latin-600.css';
	import '@fontsource/inter/latin-400.css';
	import '@fontsource/inter/latin-500.css';
	import '@fontsource/inter/latin-600.css';
	import '@fontsource/nunito/latin-400.css';
	import '@fontsource/nunito/latin-500.css';
	import '@fontsource/nunito/latin-600.css';
	import '@fontsource/open-sans/latin-400.css';
	import '@fontsource/open-sans/latin-500.css';
	import '@fontsource/open-sans/latin-600.css';
	import '@fontsource/poppins/latin-400.css';
	import '@fontsource/poppins/latin-500.css';
	import '@fontsource/poppins/latin-600.css';
	import '@fontsource/montserrat/latin-400.css';
	import '@fontsource/montserrat/latin-500.css';
	import '@fontsource/montserrat/latin-600.css';
	import favicon from '$lib/assets/favicon.svg';

	let { children } = $props();
</script>

<svelte:head><link rel="icon" href={favicon} /></svelte:head>
{@render children()}
