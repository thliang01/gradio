<script lang="ts">
	import Code from "@gradio/code";
	import Slider from "./Slider.svelte";
	import Fullscreen from "./icons/Fullscreen.svelte";
	import Close from "./icons/Close.svelte";
	import { page } from "$app/stores";
	import share from "$lib/assets/img/anchor_gray.svg";
	import { svgCheck } from "$lib/assets/copy.js";
	import { browser } from "$app/environment";
	import { onMount } from "svelte";
	import WHEEL from "$lib/json/wheel.json";

	export let demos: {
		name: string;
		dir: string;
		code: string;
		requirements: string[];
	}[];
	export let current_selection: string;
	export let show_nav = true;

	let new_demo = {
		name: "Blank",
		dir: "Blank",
		code: "# Write your own Gradio code here and see what it looks like!\nimport gradio as gr\n\n",
		requirements: []
	};

	demos.push(new_demo);

	let mounted = false;
	let controller: any;

	let dummy_elem: any = { classList: { contains: () => false } };
	let dummy_gradio: any = { dispatch: (_) => {} };

	function debounce<T extends any[]>(
		func: (...args: T) => Promise<unknown>,
		timeout: number
	): (...args: T) => void {
		let timer: any;
		return function (...args: any) {
			clearTimeout(timer);
			timer = setTimeout(() => {
				func(...args);
			}, timeout);
		};
	}

	let debounced_run_code: Function | undefined;
	let debounced_install: Function | undefined;

	function loadScript(src: string) {
		return new Promise((resolve, reject) => {
			const script = document.createElement("script");
			script.src = src;
			script.onload = () => resolve(script);
			script.onerror = () => reject(new Error(`Script load error for ${src}`));
			document.head.appendChild(script);
		});
	}

	onMount(async () => {
		try {
			await loadScript(WHEEL.gradio_lite_url + "/dist/lite.js");
			controller = createGradioApp({
				target: document.getElementById("lite-demo"),
				requirements: demos[0].requirements,
				code: demos[0].code,
				info: true,
				container: true,
				isEmbed: true,
				initialHeight: "100%",
				eager: false,
				themeMode: null,
				autoScroll: false,
				controlPageTitle: false,
				appMode: true
			});
			const debounce_timeout = 1000;
			debounced_run_code = debounce(controller.run_code, debounce_timeout);
			debounced_install = debounce(controller.install, debounce_timeout);

			mounted = true;
		} catch (error) {
			console.error("Error loading Gradio Lite:", error);
		}
	});

	let copied_link = false;
	let shared = false;
	async function copy_link(name: string) {
		let code_b64 = btoa(code);
		name = name.replaceAll(" ", "_");
		await navigator.clipboard.writeText(
			`${$page.url.href.split("?")[0]}?demo=${name}&code=${code_b64}`
		);
		copied_link = true;
		shared = true;
		setTimeout(() => (copied_link = false), 2000);
	}

	$: code = demos.find((demo) => demo.name === current_selection)?.code || "";
	$: requirements =
		demos.find((demo) => demo.name === current_selection)?.requirements || [];
	$: requirementsStr = JSON.stringify(requirements); // Use the stringified version to trigger reactivity only when the array values actually change, while the `requirements` object's identity always changes.

	$: if (mounted) {
		debounced_run_code && debounced_run_code(code);
	}
	$: if (mounted) {
		debounced_install && debounced_install(JSON.parse(requirementsStr));
	}

	let position = 0.5;

	let fullscreen = false;
	let preview_width = 100;
	let lg_breakpoint = false;

	$: lg_breakpoint = preview_width - 13 >= 688;

	if (browser) {
		let linked_demo = $page.url.searchParams.get("demo");
		let b64_code = $page.url.searchParams.get("code");

		if (linked_demo && b64_code) {
			current_selection = linked_demo.replaceAll("_", " ");
			let demo = demos.find((demo) => demo.name === current_selection);
			if (demo) {
				demo.code = atob(b64_code);
			}
		}
	}

	function show_dialog(
		current_demos: typeof demos,
		original_demos: typeof demos,
		has_shared: boolean
	) {
		let changes =
			!(JSON.stringify(current_demos) === JSON.stringify(original_demos)) &&
			!has_shared;
		if (browser) {
			if (changes) {
				window.onbeforeunload = function () {
					return true;
				};
			} else {
				window.onbeforeunload = function () {
					return null;
				};
			}
		}
	}

	let demos_copy: typeof demos = JSON.parse(JSON.stringify(demos));

	$: show_dialog(demos, demos_copy, shared);
	$: if (code) {
		shared = false;
	}
</script>

<svelte:head>
	<link rel="stylesheet" href="{WHEEL.gradio_lite_url}/dist/lite.css" />
	<link rel="stylesheet" href="https://gradio-hello-world.hf.space/theme.css" />
</svelte:head>

<div class="flex flex-row" style="position: absolute; top: -6%; right: 0.4%">
	<button
		class="border border-gray-300 rounded-md mx-2 px-2 py-.5 my-[3px] text-md text-gray-600 hover:bg-gray-50 flex"
		on:click={() => copy_link(current_selection)}
	>
		{#if !copied_link}
			<img
				class="!w-5 align-text-top inline-block self-center mr-1"
				src={share}
			/>
			<p class="inline-block">Share Your App</p>
		{:else}
			<div class="inline-block align-text-top !w-5 self-center">
				{@html svgCheck}
			</div>
			<p class="inline-block">Copied Link!</p>
		{/if}
	</button>
</div>
<div
	class=" absolute top-0 bottom-0 right-0"
	style="left:{show_nav ? 200 : 37}px"
>
	<Slider bind:position bind:show_nav>
		<div class="flex-row min-w-0 h-full" class:flex={!fullscreen}>
			{#each demos as demo, i}
				<div
					hidden={current_selection !== demo.name}
					class="code-editor w-full border-r"
					style="width: {position * 100}%"
				>
					<div class="flex justify-between align-middle h-8 border-b pl-4 pr-2">
						<h3 class="pt-1">Code</h3>
						<div class="flex float-right"></div>
					</div>

					<Code
						bind:value={demos[i].code}
						on:input={() => console.log("input")}
						label=""
						language="python"
						target={dummy_elem}
						gradio={dummy_gradio}
						lines={10}
						interactive="true"
					/>
				</div>
			{/each}

			<div
				class="preview w-full mx-auto"
				style="width: {fullscreen ? 100 : (1 - position) * 100}%"
				class:fullscreen
				bind:clientWidth={preview_width}
			>
				<div
					class="flex justify-between align-middle h-8 border-b pl-4 pr-2 ml-0 sm:ml-2"
				>
					<div class="flex align-middle">
						<h3 class="pr-2 pt-1">Preview</h3>
						<p class="pt-1.5 text-sm text-gray-600 hidden sm:block">
							{preview_width - 13}px
						</p>
						<p
							class:text-orange-300={lg_breakpoint}
							class:text-gray-300={!lg_breakpoint}
							class="pt-2 text-sm pl-2 w-6 hidden sm:block"
						>
							<svg viewBox="0 0 110 100" xmlns="http://www.w3.org/2000/svg">
								<rect width="50" height="100" rx="15" fill="currentColor" />
								<rect
									x="60"
									width="50"
									height="100"
									rx="15"
									fill="currentColor"
								/>
							</svg>
						</p>
						<p
							class:text-orange-300={!lg_breakpoint}
							class:text-gray-300={lg_breakpoint}
							class="pt-2 text-sm pl-2 w-6 hidden sm:block"
						>
							<svg viewBox="0 0 110 110" xmlns="http://www.w3.org/2000/svg">
								<rect width="110" height="45" rx="15" fill="currentColor" />
								<rect
									y="50"
									width="110"
									height="45"
									rx="15"
									fill="currentColor"
								/>
							</svg>
						</p>
					</div>
					<div class="flex">
						{#if !fullscreen}<button
								class="ml-1 w-[20px] float-right text-gray-600"
								on:click={() => (fullscreen = true)}><Fullscreen /></button
							>{:else}
							<button
								class="ml-1 w-[15px] float-right text-gray-600"
								on:click={() => (fullscreen = false)}><Close /></button
							>
						{/if}
					</div>
				</div>

				<div class="lite-demo h-[93%] pl-3" id="lite-demo" />
			</div>
		</div>
	</Slider>
</div>

<style>
	:global(div.code-editor div.block) {
		height: calc(100% - 2rem);
		border-radius: 0;
		border: none;
	}

	:global(div.code-editor div.block .cm-gutters) {
		background-color: white;
	}

	:global(div.code-editor div.block .cm-content) {
		width: 0;
	}

	:global(div.lite-demo div.gradio-container) {
		height: 100%;
		overflow-y: scroll;
		margin: 0 !important;
	}

	.code-editor :global(label) {
		display: none;
	}

	.code-editor :global(.codemirror-wrappper) {
		border-radius: var(--block-radius);
	}

	.code-editor :global(> .block) {
		border: none !important;
	}

	.code-editor :global(.cm-scroller) {
		height: 100% !important;
	}

	.lite-demo :global(.embed-container) {
		border: none !important;
	}

	.fullscreen {
		position: fixed !important;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		z-index: 1000;
		background-color: white;
	}
	/* .preview {
			width: 100% !important;
		} */
	@media (max-width: 640px) {
		.preview {
			width: 100% !important;
		}
	}
</style>
