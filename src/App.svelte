<script>
	import PaczkiView from "./PaczkiView.svelte";
	export let title;
</script>

<link
	rel="stylesheet"
	href="https://cdn.jsdelivr.net/npm/svelte-material-ui@6.0.0-beta.16/bare.min.css"
/>

<main>
	<h1>{title}</h1>
</main>
<PaczkiView />

<style>
	main {
		text-align: center;
		padding: 1em;
		max-width: 240px;
		margin: 0 auto;
	}

	h1 {
		color: #ff3e00;
		text-transform: uppercase;
		font-size: 4em;
		font-weight: 100;
	}

	@media (min-width: 640px) {
		main {
			max-width: none;
		}
	}
</style>
