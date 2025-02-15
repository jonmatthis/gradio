<script lang="ts">
	import { createEventDispatcher, tick, getContext } from "svelte";
	import type { FileData } from "@gradio/client";
	import { upload_files, upload, prepare_files } from "@gradio/client";
	import { _ } from "svelte-i18n";
	import UploadProgress from "./UploadProgress.svelte";

	export let filetype: string | string[] | null = null;
	export let dragging = false;
	export let boundedheight = true;
	export let center = true;
	export let flex = true;
	export let file_count = "single";
	export let disable_click = false;
	export let root: string;
	export let hidden = false;
	export let format: "blob" | "file" = "file";
	export let include_sources = false;
	export let uploading = false;

	let upload_id: string;
	let file_data: FileData[];
	let accept_file_types: string | null;

	// Needed for wasm support
	const upload_fn = getContext<typeof upload_files>("upload_files");

	let hidden_upload: HTMLInputElement;

	const dispatch = createEventDispatcher();

	$: if (filetype == null || typeof filetype === "string") {
		accept_file_types = filetype;
	} else {
		filetype = filetype.map((x) => {
			if (x.startsWith(".")) {
				return x;
			}
			return x + "/*";
		});
		accept_file_types = filetype.join(", ");
	}
	function updateDragging(): void {
		dragging = !dragging;
	}

	export function open_file_upload(): void {
		if (disable_click) return;
		hidden_upload.value = "";
		hidden_upload.click();
	}

	async function handle_upload(
		file_data: FileData[]
	): Promise<(FileData | null)[]> {
		await tick();
		upload_id = Math.random().toString(36).substring(2, 15);
		uploading = true;
		const _file_data = await upload(file_data, root, upload_id, upload_fn);
		dispatch("load", file_count === "single" ? _file_data?.[0] : _file_data);
		uploading = false;
		return _file_data || [];
	}

	export async function load_files(
		files: File[] | Blob[]
	): Promise<(FileData | null)[] | void> {
		if (!files.length) {
			return;
		}
		let _files: File[] = files.map((f) => new File([f], f.name));
		file_data = await prepare_files(_files);
		return await handle_upload(file_data);
	}

	async function load_files_from_upload(e: Event): Promise<void> {
		const target = e.target as HTMLInputElement;
		if (!target.files) return;
		if (format != "blob") {
			await load_files(Array.from(target.files));
		} else {
			if (file_count === "single") {
				dispatch("load", target.files[0]);
				return;
			}
			dispatch("load", target.files);
		}
	}

	function is_valid_mimetype(
		file_accept: string | string[] | null,
		mime_type: string
	): boolean {
		if (!file_accept || file_accept === "*" || file_accept === "file/*") {
			return true;
		}
		if (typeof file_accept === "string" && file_accept.endsWith("/*")) {
			file_accept = file_accept.split(",");
		}
		if (Array.isArray(file_accept)) {
			return (
				file_accept.includes(mime_type) ||
				file_accept.some((type) => {
					const [category] = type.split("/");
					return type.endsWith("/*") && mime_type.startsWith(category + "/");
				})
			);
		}
		return file_accept === mime_type;
	}

	async function loadFilesFromDrop(e: DragEvent): Promise<void> {
		dragging = false;
		if (!e.dataTransfer?.files) return;

		const files_to_load = Array.from(e.dataTransfer.files).filter((f) => {
			const file_extension =
				f.type !== "" ? f.type : "." + f.name.split(".").pop();
			if (file_extension && is_valid_mimetype(filetype, file_extension)) {
				return true;
			}
			dispatch("error", `Invalid file type only ${filetype} allowed.`);
			return false;
		});
		await load_files(files_to_load);
	}
</script>

{#if uploading}
	{#if !hidden}
		<UploadProgress {root} {upload_id} files={file_data} />
	{/if}
{:else}
	<button
		class:hidden
		class:center
		class:boundedheight
		class:flex
		style:height={include_sources ? "calc(100% - 40px" : "100%"}
		tabindex={hidden ? -1 : 0}
		on:drag|preventDefault|stopPropagation
		on:dragstart|preventDefault|stopPropagation
		on:dragend|preventDefault|stopPropagation
		on:dragover|preventDefault|stopPropagation
		on:dragenter|preventDefault|stopPropagation
		on:dragleave|preventDefault|stopPropagation
		on:drop|preventDefault|stopPropagation
		on:click={open_file_upload}
		on:drop={loadFilesFromDrop}
		on:dragenter={updateDragging}
		on:dragleave={updateDragging}
	>
		<slot />
		<input
			aria-label="file upload"
			type="file"
			bind:this={hidden_upload}
			on:change={load_files_from_upload}
			accept={accept_file_types}
			multiple={file_count === "multiple" || undefined}
			webkitdirectory={file_count === "directory" || undefined}
			mozdirectory={file_count === "directory" || undefined}
		/>
	</button>
{/if}

<style>
	button {
		cursor: pointer;
		width: var(--size-full);
	}

	.hidden {
		display: none;
		height: 0 !important;
		position: absolute;
		width: 0;
		flex-grow: 0;
	}

	.center {
		display: flex;
		justify-content: center;
	}
	.flex {
		display: flex;
		justify-content: center;
		align-items: center;
	}

	input {
		display: none;
	}
</style>
