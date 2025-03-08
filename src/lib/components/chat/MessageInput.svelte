<script lang="ts">
	import { toast } from 'svelte-sonner';
	import { v4 as uuidv4 } from 'uuid';
	import { createPicker, getAuthToken } from '$lib/utils/google-drive-picker';
	import { pickAndDownloadFile } from '$lib/utils/onedrive-file-picker';

	import { onMount, tick, getContext, createEventDispatcher, onDestroy } from 'svelte';
	const dispatch = createEventDispatcher();

	import {
		type Model,
		mobile,
		settings,
		showSidebar,
		models,
		config,
		showCallOverlay,
		tools,
		user as _user,
		showControls,
		TTSWorker
	} from '$lib/stores';

	import { blobToFile, compressImage, createMessagesList, findWordIndices } from '$lib/utils';
	import { transcribeAudio } from '$lib/apis/audio';
	import { uploadFile } from '$lib/apis/files';
	import { generateAutoCompletion } from '$lib/apis';
	import { deleteFileById } from '$lib/apis/files';

	import { WEBUI_BASE_URL, WEBUI_API_BASE_URL, PASTED_TEXT_CHARACTER_LIMIT } from '$lib/constants';

	import InputMenu from './MessageInput/InputMenu.svelte';
	import VoiceRecording from './MessageInput/VoiceRecording.svelte';
	import FilesOverlay from './MessageInput/FilesOverlay.svelte';
	import Commands from './MessageInput/Commands.svelte';

	import RichTextInput from '../common/RichTextInput.svelte';
	import Tooltip from '../common/Tooltip.svelte';
	import FileItem from '../common/FileItem.svelte';
	import Image from '../common/Image.svelte';

	import XMark from '../icons/XMark.svelte';
	import Headphone from '../icons/Headphone.svelte';
	import GlobeAlt from '../icons/GlobeAlt.svelte';
	import PhotoSolid from '../icons/PhotoSolid.svelte';
	import Photo from '../icons/Photo.svelte';
	import CommandLine from '../icons/CommandLine.svelte';
	import { KokoroWorker } from '$lib/workers/KokoroWorker';

	const i18n = getContext('i18n');

	export let transparentBackground = false;

	export let onChange: Function = () => {};
	export let createMessagePair: Function;
	export let stopResponse: Function;

	export let autoScroll = false;

	export let atSelectedModel: Model | undefined = undefined;
	export let selectedModels: [''];
        export let attestationValid = false;
        export let attestationInfo = null;
  $: {
    try {
      console.log('[MessageInput] attestation State Update:', {
        valid: attestationValid,
        info: attestationInfo ? JSON.parse(JSON.stringify(attestationInfo)) : 'undefined'
      });
    } catch (e) {
      console.error('attestationInfo failed:', e);
    }
  }
	let selectedModelIds = [];
	$: selectedModelIds = atSelectedModel !== undefined ? [atSelectedModel.id] : selectedModels;

	export let history;

	export let prompt = '';
	export let files = [];

	export let selectedToolIds = [];

	export let imageGenerationEnabled = false;
	export let webSearchEnabled = false;
	export let codeInterpreterEnabled = false;

	$: onChange({
		prompt,
		files,
		selectedToolIds,
		imageGenerationEnabled,
		webSearchEnabled
	});

	let loaded = false;
	let recording = false;

	let chatInputContainerElement;
	let chatInputElement;

	let filesInputElement;
	let commandsElement;

	let inputFiles;
	let dragged = false;

	let user = null;
	export let placeholder = '';

	let visionCapableModels = [];
	$: visionCapableModels = [...(atSelectedModel ? [atSelectedModel] : selectedModels)].filter(
		(model) => $models.find((m) => m.id === model)?.info?.meta?.capabilities?.vision ?? true
	);

	const scrollToBottom = () => {
		const element = document.getElementById('messages-container');
		element.scrollTo({
			top: element.scrollHeight,
			behavior: 'smooth'
		});
	};

	const screenCaptureHandler = async () => {
		try {
			// Request screen media
			const mediaStream = await navigator.mediaDevices.getDisplayMedia({
				video: { cursor: 'never' },
				audio: false
			});
			// Once the user selects a screen, temporarily create a video element
			const video = document.createElement('video');
			video.srcObject = mediaStream;
			// Ensure the video loads without affecting user experience or tab switching
			await video.play();
			// Set up the canvas to match the video dimensions
			const canvas = document.createElement('canvas');
			canvas.width = video.videoWidth;
			canvas.height = video.videoHeight;
			// Grab a single frame from the video stream using the canvas
			const context = canvas.getContext('2d');
			context.drawImage(video, 0, 0, canvas.width, canvas.height);
			// Stop all video tracks (stop screen sharing) after capturing the image
			mediaStream.getTracks().forEach((track) => track.stop());

			// bring back focus to this current tab, so that the user can see the screen capture
			window.focus();

			// Convert the canvas to a Base64 image URL
			const imageUrl = canvas.toDataURL('image/png');
			// Add the captured image to the files array to render it
			files = [...files, { type: 'image', url: imageUrl }];
			// Clean memory: Clear video srcObject
			video.srcObject = null;
		} catch (error) {
			// Handle any errors (e.g., user cancels screen sharing)
			console.error('Error capturing screen:', error);
		}
	};

	const uploadFileHandler = async (file, fullContext: boolean = false) => {
		if ($_user?.role !== 'admin' && !($_user?.permissions?.chat?.file_upload ?? true)) {
			toast.error($i18n.t('You do not have permission to upload files.'));
			return null;
		}

		const tempItemId = uuidv4();
		const fileItem = {
			type: 'file',
			file: '',
			id: null,
			url: '',
			name: file.name,
			collection_name: '',
			status: 'uploading',
			size: file.size,
			error: '',
			itemId: tempItemId,
			...(fullContext ? { context: 'full' } : {})
		};

		if (fileItem.size == 0) {
			toast.error($i18n.t('You cannot upload an empty file.'));
			return null;
		}

		files = [...files, fileItem];

		try {
			// During the file upload, file content is automatically extracted.
			const uploadedFile = await uploadFile(localStorage.token, file);

			if (uploadedFile) {
				console.log('File upload completed:', {
					id: uploadedFile.id,
					name: fileItem.name,
					collection: uploadedFile?.meta?.collection_name
				});

				if (uploadedFile.error) {
					console.warn('File upload warning:', uploadedFile.error);
					toast.warning(uploadedFile.error);
				}

				fileItem.status = 'uploaded';
				fileItem.file = uploadedFile;
				fileItem.id = uploadedFile.id;
				fileItem.collection_name =
					uploadedFile?.meta?.collection_name || uploadedFile?.collection_name;
				fileItem.url = `${WEBUI_API_BASE_URL}/files/${uploadedFile.id}`;

				files = files;
			} else {
				files = files.filter((item) => item?.itemId !== tempItemId);
			}
		} catch (e) {
			toast.error(`${e}`);
			files = files.filter((item) => item?.itemId !== tempItemId);
		}
	};

	const inputFilesHandler = async (inputFiles) => {
		console.log('Input files handler called with:', inputFiles);
		inputFiles.forEach((file) => {
			console.log('Processing file:', {
				name: file.name,
				type: file.type,
				size: file.size,
				extension: file.name.split('.').at(-1)
			});

			if (
				($config?.file?.max_size ?? null) !== null &&
				file.size > ($config?.file?.max_size ?? 0) * 1024 * 1024
			) {
				console.log('File exceeds max size limit:', {
					fileSize: file.size,
					maxSize: ($config?.file?.max_size ?? 0) * 1024 * 1024
				});
				toast.error(
					$i18n.t(`File size should not exceed {{maxSize}} MB.`, {
						maxSize: $config?.file?.max_size
					})
				);
				return;
			}

			if (
				['image/gif', 'image/webp', 'image/jpeg', 'image/png', 'image/avif'].includes(file['type'])
			) {
				if (visionCapableModels.length === 0) {
					toast.error($i18n.t('Selected model(s) do not support image inputs'));
					return;
				}
				let reader = new FileReader();
				reader.onload = async (event) => {
					let imageUrl = event.target.result;

					if ($settings?.imageCompression ?? false) {
						const width = $settings?.imageCompressionSize?.width ?? null;
						const height = $settings?.imageCompressionSize?.height ?? null;

						if (width || height) {
							imageUrl = await compressImage(imageUrl, width, height);
						}
					}

					files = [
						...files,
						{
							type: 'image',
							url: `${imageUrl}`
						}
					];
				};
				reader.readAsDataURL(file);
			} else {
				uploadFileHandler(file);
			}
		});
	};

	const handleKeyDown = (event: KeyboardEvent) => {
		if (event.key === 'Escape') {
			console.log('Escape');
			dragged = false;
		}
	};

	const onDragOver = (e) => {
		e.preventDefault();

		// Check if a file is being dragged.
		if (e.dataTransfer?.types?.includes('Files')) {
			dragged = true;
		} else {
			dragged = false;
		}
	};

	const onDragLeave = () => {
		dragged = false;
	};

	const onDrop = async (e) => {
		e.preventDefault();
		console.log(e);

		if (e.dataTransfer?.files) {
			const inputFiles = Array.from(e.dataTransfer?.files);
			if (inputFiles && inputFiles.length > 0) {
				console.log(inputFiles);
				inputFilesHandler(inputFiles);
			}
		}

		dragged = false;
	};

	onMount(async () => {
		loaded = true;

		window.setTimeout(() => {
			const chatInput = document.getElementById('chat-input');
			chatInput?.focus();
		}, 0);

		window.addEventListener('keydown', handleKeyDown);

		await tick();

		const dropzoneElement = document.getElementById('chat-container');

		dropzoneElement?.addEventListener('dragover', onDragOver);
		dropzoneElement?.addEventListener('drop', onDrop);
		dropzoneElement?.addEventListener('dragleave', onDragLeave);
	});

	onDestroy(() => {
		console.log('destroy');
		window.removeEventListener('keydown', handleKeyDown);

		const dropzoneElement = document.getElementById('chat-container');

		if (dropzoneElement) {
			dropzoneElement?.removeEventListener('dragover', onDragOver);
			dropzoneElement?.removeEventListener('drop', onDrop);
			dropzoneElement?.removeEventListener('dragleave', onDragLeave);
		}
	});
</script>

<FilesOverlay show={dragged} />

{#if loaded}
	<div class="w-full font-primary">
		<div class=" mx-auto inset-x-0 bg-transparent flex justify-center">
			<div
				class="flex flex-col px-3 {($settings?.widescreenMode ?? null)
					? 'max-w-full'
					: 'max-w-6xl'} w-full"
			>
				<div class="relative">
					{#if autoScroll === false && history?.currentId}
						<div
							class=" absolute -top-12 left-0 right-0 flex justify-center z-30 pointer-events-none"
						>
							<button
								class=" bg-white border border-gray-100 dark:border-none dark:bg-white/20 p-1.5 rounded-full pointer-events-auto"
								on:click={() => {
									autoScroll = true;
									scrollToBottom();
								}}
							>
								<svg
									xmlns="http://www.w3.org/2000/svg"
									viewBox="0 0 20 20"
									fill="currentColor"
									class="w-5 h-5"
								>
									<path
										fill-rule="evenodd"
										d="M10 3a.75.75 0 01.75.75v10.638l3.96-4.158a.75.75 0 111.08 1.04l-5.25 5.5a.75.75 0 01-1.08 0l-5.25-5.5a.75.75 0 111.08-1.04l3.96 4.158V3.75A.75.75 0 0110 3z"
										clip-rule="evenodd"
									/>
								</svg>
							</button>
						</div>
					{/if}
				</div>

				<div class="w-full relative">
					{#if atSelectedModel !== undefined || selectedToolIds.length > 0 || webSearchEnabled || ($settings?.webSearch ?? false) === 'always' || imageGenerationEnabled || codeInterpreterEnabled}
						<div
							class="px-3 pb-0.5 pt-1.5 text-left w-full flex flex-col absolute bottom-0 left-0 right-0 bg-linear-to-t from-white dark:from-gray-900 z-10"
						>
							{#if selectedToolIds.length > 0}
								<div class="flex items-center justify-between w-full">
									<div class="flex items-center gap-2.5 text-sm dark:text-gray-500">
										<div class="pl-1">
											<span class="relative flex size-2">
												<span
													class="animate-ping absolute inline-flex h-full w-full rounded-full bg-yellow-400 opacity-75"
												/>
												<span class="relative inline-flex rounded-full size-2 bg-yellow-500" />
											</span>
										</div>
										<div class="  text-ellipsis line-clamp-1 flex">
											{#each selectedToolIds.map((id) => {
												return $tools ? $tools.find((t) => t.id === id) : { id: id, name: id };
											}) as tool, toolIdx (toolIdx)}
												<Tooltip
													content={tool?.meta?.description ?? ''}
													className=" {toolIdx !== 0 ? 'pl-0.5' : ''} shrink-0"
													placement="top"
												>
													{tool.name}
												</Tooltip>

												{#if toolIdx !== selectedToolIds.length - 1}
													<span>, </span>
												{/if}
											{/each}
										</div>
									</div>
								</div>
							{/if}

							{#if webSearchEnabled || ($config?.features?.enable_web_search && ($settings?.webSearch ?? false)) === 'always'}
								<div class="flex items-center justify-between w-full">
									<div class="flex items-center gap-2.5 text-sm dark:text-gray-500">
										<div class="pl-1">
											<span class="relative flex size-2">
												<span
													class="animate-ping absolute inline-flex h-full w-full rounded-full bg-blue-400 opacity-75"
												/>
												<span class="relative inline-flex rounded-full size-2 bg-blue-500" />
											</span>
										</div>
										<div class=" translate-y-[0.5px]">{$i18n.t('Search the internet')}</div>
									</div>
								</div>
							{/if}

							{#if imageGenerationEnabled}
								<div class="flex items-center justify-between w-full">
									<div class="flex items-center gap-2.5 text-sm dark:text-gray-500">
										<div class="pl-1">
											<span class="relative flex size-2">
												<span
													class="animate-ping absolute inline-flex h-full w-full rounded-full bg-teal-400 opacity-75"
												/>
												<span class="relative inline-flex rounded-full size-2 bg-teal-500" />
											</span>
										</div>
										<div class=" translate-y-[0.5px]">{$i18n.t('Generate an image')}</div>
									</div>
								</div>
							{/if}

							{#if codeInterpreterEnabled}
								<div class="flex items-center justify-between w-full">
									<div class="flex items-center gap-2.5 text-sm dark:text-gray-500">
										<div class="pl-1">
											<span class="relative flex size-2">
												<span
													class="animate-ping absolute inline-flex h-full w-full rounded-full bg-green-400 opacity-75"
												/>
												<span class="relative inline-flex rounded-full size-2 bg-green-500" />
											</span>
										</div>
										<div class=" translate-y-[0.5px]">{$i18n.t('Execute code for analysis')}</div>
									</div>
								</div>
							{/if}

							{#if atSelectedModel !== undefined}
								<div class="flex items-center justify-between w-full">
									<div class="pl-[1px] flex items-center gap-2 text-sm dark:text-gray-500">
										<img
											crossorigin="anonymous"
											alt="model profile"
											class="size-3.5 max-w-[28px] object-cover rounded-full"
											src={$models.find((model) => model.id === atSelectedModel.id)?.info?.meta
												?.profile_image_url ??
												($i18n.language === 'dg-DG'
													? `/doge.png`
													: `${WEBUI_BASE_URL}/static/favicon.png`)}
										/>
										<div class="translate-y-[0.5px]">
											Talking to <span class=" font-medium">{atSelectedModel.name}</span>
										</div>
									</div>
									<div>
										<button
											class="flex items-center dark:text-gray-500"
											on:click={() => {
												atSelectedModel = undefined;
											}}
										>
											<XMark />
										</button>
									</div>
								</div>
							{/if}
						</div>
					{/if}

					<Commands
						bind:this={commandsElement}
						bind:prompt
						bind:files
						on:upload={(e) => {
							dispatch('upload', e.detail);
						}}
						on:select={(e) => {
							const data = e.detail;

							if (data?.type === 'model') {
								atSelectedModel = data.data;
							}

							const chatInputElement = document.getElementById('chat-input');
							chatInputElement?.focus();
						}}
					/>
				</div>
			</div>
		</div>

		<div class="{transparentBackground ? 'bg-transparent' : 'bg-white dark:bg-gray-900'} ">
			<div
				class="{($settings?.widescreenMode ?? null)
					? 'max-w-full'
					: 'max-w-6xl'} px-2.5 mx-auto inset-x-0"
			>
				<div class="">
					<input
						bind:this={filesInputElement}
						bind:files={inputFiles}
						type="file"
						hidden
						multiple
						on:change={async () => {
							if (inputFiles && inputFiles.length > 0) {
								const _inputFiles = Array.from(inputFiles);
								inputFilesHandler(_inputFiles);
							} else {
								toast.error($i18n.t(`File not found.`));
							}

							filesInputElement.value = '';
						}}
					/>

					{#if recording}
						<VoiceRecording
							bind:recording
							on:cancel={async () => {
								recording = false;

								await tick();
								document.getElementById('chat-input')?.focus();
							}}
							on:confirm={async (e) => {
								const { text, filename } = e.detail;
								prompt = `${prompt}${text} `;

								recording = false;

								await tick();
								document.getElementById('chat-input')?.focus();

								if ($settings?.speechAutoSend ?? false) {
									dispatch('submit', prompt);
								}
							}}
						/>
					{:else}
						<form
							class="w-full flex gap-1.5"
							on:submit|preventDefault={() => {
								// check if selectedModels support image input
								dispatch('submit', prompt);
							}}
						>
							<div
								class="flex-1 flex flex-col relative w-full rounded-3xl px-1 bg-gray-600/5 dark:bg-gray-400/5 dark:text-gray-100"
								dir={$settings?.chatDirection ?? 'LTR'}
							>
								{#if files.length > 0}
									<div class="mx-2 mt-2.5 -mb-1 flex items-center flex-wrap gap-2">
										{#each files as file, fileIdx}
											{#if file.type === 'image'}
												<div class=" relative group">
													<div class="relative flex items-center">
														<Image
															src={file.url}
															alt="input"
															imageClassName=" size-14 rounded-xl object-cover"
														/>
														{#if atSelectedModel ? visionCapableModels.length === 0 : selectedModels.length !== visionCapableModels.length}
															<Tooltip
																className=" absolute top-1 left-1"
																content={$i18n.t('{{ models }}', {
																	models: [
																		...(atSelectedModel ? [atSelectedModel] : selectedModels)
																	]
																		.filter((id) => !visionCapableModels.includes(id))
																		.join(', ')
																})}
															>
																<svg
																	xmlns="http://www.w3.org/2000/svg"
																	viewBox="0 0 24 24"
																	fill="currentColor"
																	class="size-4 fill-yellow-300"
																>
																	<path
																		fill-rule="evenodd"
																		d="M9.401 3.003c1.155-2 4.043-2 5.197 0l7.355 12.748c1.154 2-.29 4.5-2.599 4.5H4.645c-2.309 0-3.752-2.5-2.598-4.5L9.4 3.003ZM12 8.25a.75.75 0 0 1 .75.75v3.75a.75.75 0 0 1-1.5 0V9a.75.75 0 0 1 .75-.75Zm0 8.25a.75.75 0 1 0 0-1.5.75.75 0 0 0 0 1.5Z"
																		clip-rule="evenodd"
																	/>
																</svg>
															</Tooltip>
														{/if}
													</div>
													<div class=" absolute -top-1 -right-1">
														<button
															class=" bg-white text-black border border-white rounded-full group-hover:visible invisible transition"
															type="button"
															on:click={() => {
																files.splice(fileIdx, 1);
																files = files;
															}}
														>
															<svg
																xmlns="http://www.w3.org/2000/svg"
																viewBox="0 0 20 20"
																fill="currentColor"
																class="size-4"
															>
																<path
																	d="M6.28 5.22a.75.75 0 00-1.06 1.06L8.94 10l-3.72 3.72a.75.75 0 101.06 1.06L10 11.06l3.72 3.72a.75.75 0 101.06-1.06L11.06 10l3.72-3.72a.75.75 0 00-1.06-1.06L10 8.94 6.28 5.22z"
																/>
															</svg>
														</button>
													</div>
												</div>
											{:else}
												<FileItem
													item={file}
													name={file.name}
													type={file.type}
													size={file?.size}
													loading={file.status === 'uploading'}
													dismissible={true}
													edit={true}
													on:dismiss={async () => {
														if (file.type !== 'collection' && !file?.collection) {
															if (file.id) {
																// This will handle both file deletion and Chroma cleanup
																await deleteFileById(localStorage.token, file.id);
															}
														}

														// Remove from UI state
														files.splice(fileIdx, 1);
														files = files;
													}}
													on:click={() => {
														console.log(file);
													}}
												/>
											{/if}
										{/each}
									</div>
								{/if}

								<div class="px-2.5">
									{#if $settings?.richTextInput ?? true}
										<div
											class="scrollbar-hidden text-left bg-transparent dark:text-gray-100 outline-hidden w-full pt-3 px-1 resize-none h-fit max-h-80 overflow-auto"
										>
											<RichTextInput
												bind:this={chatInputElement}
												bind:value={prompt}
												id="chat-input"
												messageInput={true}
												shiftEnter={!$mobile ||
													!(
														'ontouchstart' in window ||
														navigator.maxTouchPoints > 0 ||
														navigator.msMaxTouchPoints > 0
													)}
												placeholder={placeholder ? placeholder : $i18n.t('Send a Message')}
												largeTextAsFile={$settings?.largeTextAsFile ?? false}
												autocomplete={$config?.features.enable_autocomplete_generation}
												generateAutoCompletion={async (text) => {
													if (selectedModelIds.length === 0 || !selectedModelIds.at(0)) {
														toast.error($i18n.t('Please select a model first.'));
													}

													const res = await generateAutoCompletion(
														localStorage.token,
														selectedModelIds.at(0),
														text,
														history?.currentId
															? createMessagesList(history, history.currentId)
															: null
													).catch((error) => {
														console.log(error);

														return null;
													});

													console.log(res);
													return res;
												}}
												on:keydown={async (e) => {
													e = e.detail.event;

													const isCtrlPressed = e.ctrlKey || e.metaKey; // metaKey is for Cmd key on Mac
													const commandsContainerElement =
														document.getElementById('commands-container');

													if (e.key === 'Escape') {
														stopResponse();
													}

													// Command/Ctrl + Shift + Enter to submit a message pair
													if (isCtrlPressed && e.key === 'Enter' && e.shiftKey) {
														e.preventDefault();
														createMessagePair(prompt);
													}

													// Check if Ctrl + R is pressed
													if (prompt === '' && isCtrlPressed && e.key.toLowerCase() === 'r') {
														e.preventDefault();
														console.log('regenerate');

														const regenerateButton = [
															...document.getElementsByClassName('regenerate-response-button')
														]?.at(-1);

														regenerateButton?.click();
													}

													if (prompt === '' && e.key == 'ArrowUp') {
														e.preventDefault();

														const userMessageElement = [
															...document.getElementsByClassName('user-message')
														]?.at(-1);

														if (userMessageElement) {
															userMessageElement.scrollIntoView({ block: 'center' });
															const editButton = [
																...document.getElementsByClassName('edit-user-message-button')
															]?.at(-1);

															editButton?.click();
														}
													}

													if (commandsContainerElement) {
														if (commandsContainerElement && e.key === 'ArrowUp') {
															e.preventDefault();
															commandsElement.selectUp();

															const commandOptionButton = [
																...document.getElementsByClassName('selected-command-option-button')
															]?.at(-1);
															commandOptionButton.scrollIntoView({ block: 'center' });
														}

														if (commandsContainerElement && e.key === 'ArrowDown') {
															e.preventDefault();
															commandsElement.selectDown();

															const commandOptionButton = [
																...document.getElementsByClassName('selected-command-option-button')
															]?.at(-1);
															commandOptionButton.scrollIntoView({ block: 'center' });
														}

														if (commandsContainerElement && e.key === 'Tab') {
															e.preventDefault();

															const commandOptionButton = [
																...document.getElementsByClassName('selected-command-option-button')
															]?.at(-1);

															commandOptionButton?.click();
														}

														if (commandsContainerElement && e.key === 'Enter') {
															e.preventDefault();

															const commandOptionButton = [
																...document.getElementsByClassName('selected-command-option-button')
															]?.at(-1);

															if (commandOptionButton) {
																commandOptionButton?.click();
															} else {
																document.getElementById('send-message-button')?.click();
															}
														}
													} else {
														if (
															!$mobile ||
															!(
																'ontouchstart' in window ||
																navigator.maxTouchPoints > 0 ||
																navigator.msMaxTouchPoints > 0
															)
														) {
															// Prevent Enter key from creating a new line
															// Uses keyCode '13' for Enter key for chinese/japanese keyboards
															if (e.keyCode === 13 && !e.shiftKey) {
																e.preventDefault();
															}

															// Submit the prompt when Enter key is pressed
															if (
																(prompt !== '' || files.length > 0) &&
																e.keyCode === 13 &&
																!e.shiftKey
															) {
																dispatch('submit', prompt);
															}
														}
													}

													if (e.key === 'Escape') {
														console.log('Escape');
														atSelectedModel = undefined;
														selectedToolIds = [];
														webSearchEnabled = false;
														imageGenerationEnabled = false;
													}
												}}
												on:paste={async (e) => {
													e = e.detail.event;
													console.log(e);

													const clipboardData = e.clipboardData || window.clipboardData;

													if (clipboardData && clipboardData.items) {
														for (const item of clipboardData.items) {
															if (item.type.indexOf('image') !== -1) {
																const blob = item.getAsFile();
																const reader = new FileReader();

																reader.onload = function (e) {
																	files = [
																		...files,
																		{
																			type: 'image',
																			url: `${e.target.result}`
																		}
																	];
																};

																reader.readAsDataURL(blob);
															} else if (item.type === 'text/plain') {
																if ($settings?.largeTextAsFile ?? false) {
																	const text = clipboardData.getData('text/plain');

																	if (text.length > PASTED_TEXT_CHARACTER_LIMIT) {
																		e.preventDefault();
																		const blob = new Blob([text], { type: 'text/plain' });
																		const file = new File([blob], `Pasted_Text_${Date.now()}.txt`, {
																			type: 'text/plain'
																		});

																		await uploadFileHandler(file, true);
																	}
																}
															}
														}
													}
												}}
											/>
										</div>
									{:else}
										<textarea
											id="chat-input"
											bind:this={chatInputElement}
											class="scrollbar-hidden bg-transparent dark:text-gray-100 outline-hidden w-full pt-3 px-1 resize-none"
											placeholder={placeholder ? placeholder : $i18n.t('Send a Message')}
											bind:value={prompt}
											on:keypress={(e) => {
												if (
													!$mobile ||
													!(
														'ontouchstart' in window ||
														navigator.maxTouchPoints > 0 ||
														navigator.msMaxTouchPoints > 0
													)
												) {
													// Prevent Enter key from creating a new line
													if (e.key === 'Enter' && !e.shiftKey) {
														e.preventDefault();
													}

													// Submit the prompt when Enter key is pressed
													if (
														(prompt !== '' || files.length > 0) &&
														e.key === 'Enter' &&
														!e.shiftKey
													) {
														dispatch('submit', prompt);
													}
												}
											}}
											on:keydown={async (e) => {
												const isCtrlPressed = e.ctrlKey || e.metaKey; // metaKey is for Cmd key on Mac
												const commandsContainerElement =
													document.getElementById('commands-container');

												if (e.key === 'Escape') {
													stopResponse();
												}
												// Command/Ctrl + Shift + Enter to submit a message pair
												if (isCtrlPressed && e.key === 'Enter' && e.shiftKey) {
													e.preventDefault();
													createMessagePair(prompt);
												}

												// Check if Ctrl + R is pressed
												if (prompt === '' && isCtrlPressed && e.key.toLowerCase() === 'r') {
													e.preventDefault();
													console.log('regenerate');

													const regenerateButton = [
														...document.getElementsByClassName('regenerate-response-button')
													]?.at(-1);

													regenerateButton?.click();
												}

												if (prompt === '' && e.key == 'ArrowUp') {
													e.preventDefault();

													const userMessageElement = [
														...document.getElementsByClassName('user-message')
													]?.at(-1);

													const editButton = [
														...document.getElementsByClassName('edit-user-message-button')
													]?.at(-1);

													console.log(userMessageElement);

													userMessageElement.scrollIntoView({ block: 'center' });
													editButton?.click();
												}

												if (commandsContainerElement && e.key === 'ArrowUp') {
													e.preventDefault();
													commandsElement.selectUp();

													const commandOptionButton = [
														...document.getElementsByClassName('selected-command-option-button')
													]?.at(-1);
													commandOptionButton.scrollIntoView({ block: 'center' });
												}

												if (commandsContainerElement && e.key === 'ArrowDown') {
													e.preventDefault();
													commandsElement.selectDown();

													const commandOptionButton = [
														...document.getElementsByClassName('selected-command-option-button')
													]?.at(-1);
													commandOptionButton.scrollIntoView({ block: 'center' });
												}

												if (commandsContainerElement && e.key === 'Enter') {
													e.preventDefault();

													const commandOptionButton = [
														...document.getElementsByClassName('selected-command-option-button')
													]?.at(-1);

													if (e.shiftKey) {
														prompt = `${prompt}\n`;
													} else if (commandOptionButton) {
														commandOptionButton?.click();
													} else {
														document.getElementById('send-message-button')?.click();
													}
												}

												if (commandsContainerElement && e.key === 'Tab') {
													e.preventDefault();

													const commandOptionButton = [
														...document.getElementsByClassName('selected-command-option-button')
													]?.at(-1);

													commandOptionButton?.click();
												} else if (e.key === 'Tab') {
													const words = findWordIndices(prompt);

													if (words.length > 0) {
														const word = words.at(0);
														const fullPrompt = prompt;

														prompt = prompt.substring(0, word?.endIndex + 1);
														await tick();

														e.target.scrollTop = e.target.scrollHeight;
														prompt = fullPrompt;
														await tick();

														e.preventDefault();
														e.target.setSelectionRange(word?.startIndex, word.endIndex + 1);
													}

													e.target.style.height = '';
													e.target.style.height = Math.min(e.target.scrollHeight, 320) + 'px';
												}

												if (e.key === 'Escape') {
													console.log('Escape');
													atSelectedModel = undefined;
													selectedToolIds = [];
													webSearchEnabled = false;
													imageGenerationEnabled = false;
												}
											}}
											rows="1"
											on:input={async (e) => {
												e.target.style.height = '';
												e.target.style.height = Math.min(e.target.scrollHeight, 320) + 'px';
											}}
											on:focus={async (e) => {
												e.target.style.height = '';
												e.target.style.height = Math.min(e.target.scrollHeight, 320) + 'px';
											}}
											on:paste={async (e) => {
												const clipboardData = e.clipboardData || window.clipboardData;

												if (clipboardData && clipboardData.items) {
													for (const item of clipboardData.items) {
														if (item.type.indexOf('image') !== -1) {
															const blob = item.getAsFile();
															const reader = new FileReader();

															reader.onload = function (e) {
																files = [
																	...files,
																	{
																		type: 'image',
																		url: `${e.target.result}`
																	}
																];
															};

															reader.readAsDataURL(blob);
														} else if (item.type === 'text/plain') {
															if ($settings?.largeTextAsFile ?? false) {
																const text = clipboardData.getData('text/plain');

																if (text.length > PASTED_TEXT_CHARACTER_LIMIT) {
																	e.preventDefault();
																	const blob = new Blob([text], { type: 'text/plain' });
																	const file = new File([blob], `Pasted_Text_${Date.now()}.txt`, {
																		type: 'text/plain'
																	});

																	await uploadFileHandler(file, true);
																}
															}
														}
													}
												}
											}}
										/>
									{/if}
								</div>

								<div class=" flex justify-between mt-1.5 mb-2.5 mx-0.5 max-w-full">
									<div class="ml-1 self-end gap-0.5 flex items-center flex-1 max-w-[80%]">
										<InputMenu
											bind:selectedToolIds
											{screenCaptureHandler}
											{inputFilesHandler}
											uploadFilesHandler={() => {
												filesInputElement.click();
											}}
											uploadGoogleDriveHandler={async () => {
												try {
													const fileData = await createPicker();
													if (fileData) {
														const file = new File([fileData.blob], fileData.name, {
															type: fileData.blob.type
														});
														await uploadFileHandler(file);
													} else {
														console.log('No file was selected from Google Drive');
													}
												} catch (error) {
													console.error('Google Drive Error:', error);
													toast.error(
														$i18n.t('Error accessing Google Drive: {{error}}', {
															error: error.message
														})
													);
												}
											}}
											uploadOneDriveHandler={async () => {
												try {
													const fileData = await pickAndDownloadFile();
													if (fileData) {
														const file = new File([fileData.blob], fileData.name, {
															type: fileData.blob.type || 'application/octet-stream'
														});
														await uploadFileHandler(file);
													} else {
														console.log('No file was selected from OneDrive');
													}
												} catch (error) {
													console.error('OneDrive Error:', error);
												}
											}}
											onClose={async () => {
												await tick();

												const chatInput = document.getElementById('chat-input');
												chatInput?.focus();
											}}
										>
											<button
												class="bg-transparent hover:bg-gray-100 text-gray-800 dark:text-white dark:hover:bg-gray-800 transition rounded-full p-1.5 outline-hidden focus:outline-hidden"
												type="button"
												aria-label="More"
											>
												<svg
													xmlns="http://www.w3.org/2000/svg"
													viewBox="0 0 20 20"
													fill="currentColor"
													class="size-5"
												>
													<path
														d="M10.75 4.75a.75.75 0 0 0-1.5 0v4.5h-4.5a.75.75 0 0 0 0 1.5h4.5v4.5a.75.75 0 0 0 1.5 0v-4.5h4.5a.75.75 0 0 0 0-1.5h-4.5v-4.5Z"
													/>
												</svg>
											</button>
										</InputMenu>

										<div class="flex gap-0.5 items-center overflow-x-auto scrollbar-none flex-1">
											{#if $_user}
												{#if $config?.features?.enable_web_search && ($_user.role === 'admin' || $_user?.permissions?.features?.web_search)}
													<Tooltip content={$i18n.t('Search the internet')} placement="top">
														<button
															on:click|preventDefault={() => (webSearchEnabled = !webSearchEnabled)}
															type="button"
															class="px-1.5 @sm:px-2.5 py-1.5 flex gap-1.5 items-center text-sm rounded-full font-medium transition-colors duration-300 focus:outline-hidden max-w-full overflow-hidden {webSearchEnabled ||
															($settings?.webSearch ?? false) === 'always'
																? 'bg-blue-100 dark:bg-blue-500/20 text-blue-500 dark:text-blue-400'
																: 'bg-transparent text-gray-600 dark:text-gray-300 border-gray-200 hover:bg-gray-100 dark:hover:bg-gray-800'}"
														>
															<GlobeAlt className="size-5" strokeWidth="1.75" />
															<span
																class="hidden @sm:block whitespace-nowrap overflow-hidden text-ellipsis translate-y-[0.5px] mr-0.5"
																>{$i18n.t('Web Search')}</span
															>
														</button>
													</Tooltip>
												{/if}

												{#if $config?.features?.enable_image_generation && ($_user.role === 'admin' || $_user?.permissions?.features?.image_generation)}
													<Tooltip content={$i18n.t('Generate an image')} placement="top">
														<button
															on:click|preventDefault={() =>
																(imageGenerationEnabled = !imageGenerationEnabled)}
															type="button"
															class="px-1.5 @sm:px-2.5 py-1.5 flex gap-1.5 items-center text-sm rounded-full font-medium transition-colors duration-300 focus:outline-hidden max-w-full overflow-hidden {imageGenerationEnabled
																? 'bg-gray-100 dark:bg-gray-500/20 text-gray-600 dark:text-gray-400'
																: 'bg-transparent text-gray-600 dark:text-gray-300 border-gray-200 hover:bg-gray-100 dark:hover:bg-gray-800 '}"
														>
															<Photo className="size-5" strokeWidth="1.75" />
															<span
																class="hidden @sm:block whitespace-nowrap overflow-hidden text-ellipsis translate-y-[0.5px] mr-0.5"
																>{$i18n.t('Image')}</span
															>
														</button>
													</Tooltip>
												{/if}

												{#if $config?.features?.enable_code_interpreter && ($_user.role === 'admin' || $_user?.permissions?.features?.code_interpreter)}
													<Tooltip content={$i18n.t('Execute code for analysis')} placement="top">
														<button
															on:click|preventDefault={() =>
																(codeInterpreterEnabled = !codeInterpreterEnabled)}
															type="button"
															class="px-1.5 @sm:px-2.5 py-1.5 flex gap-1.5 items-center text-sm rounded-full font-medium transition-colors duration-300 focus:outline-hidden max-w-full overflow-hidden {codeInterpreterEnabled
																? 'bg-gray-100 dark:bg-gray-500/20 text-gray-600 dark:text-gray-400'
																: 'bg-transparent text-gray-600 dark:text-gray-300 border-gray-200 hover:bg-gray-100 dark:hover:bg-gray-800 '}"
														>
															<CommandLine className="size-5" strokeWidth="1.75" />
															<span
																class="hidden @sm:block whitespace-nowrap overflow-hidden text-ellipsis translate-y-[0.5px] mr-0.5"
																>{$i18n.t('Code Interpreter')}</span
															>
														</button>
													</Tooltip>
												{/if}
											{/if}
										</div>
									</div>

									<div class="self-end flex space-x-1 mr-1 shrink-0">
{#if !history?.currentId || history.messages[history.currentId]?.done == true}
{#key [attestationValid, JSON.stringify(attestationInfo)]}
											<Tooltip content={JSON.stringify(attestationInfo, null, 2)}>
												<button
													id="attestation-info-button"
													class=" text-gray-600 dark:text-gray-300 hover:text-gray-700 dark:hover:text-gray-200 transition rounded-full p-1.5 mr-0.5 self-center"
													type="button"
													on:click={async () => {
														
													}}
													aria-label="Attestation Info"
												>
{#if attestationValid}
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" viewBox="0 0 240 265"
														fill="currentColor"
														class="w-5 h-5 translate-y-[0.5px]" ><image xlink:href="data:image/png;base64,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" width="240" height="265.6" preserveAspectRatio="none"/></svg>
{:else}
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" viewBox="0 0 233 268"
														fill="currentColor"
														class="w-5 h-5 translate-y-[0.5px]" ><image xlink:href="data:image/png;base64,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" width="233.6" height="268.8" preserveAspectRatio="none"/></svg>
{/if}
												</button>
											</Tooltip>
{/key}
										{/if}
									
										{#if !history?.currentId || history.messages[history.currentId]?.done == true}
											<Tooltip content={$i18n.t('Record voice')}>
												<button
													id="voice-input-button"
													class=" text-gray-600 dark:text-gray-300 hover:text-gray-700 dark:hover:text-gray-200 transition rounded-full p-1.5 mr-0.5 self-center"
													type="button"
													on:click={async () => {
														try {
															let stream = await navigator.mediaDevices
																.getUserMedia({ audio: true })
																.catch(function (err) {
																	toast.error(
																		$i18n.t(
																			`Permission denied when accessing microphone: {{error}}`,
																			{
																				error: err
																			}
																		)
																	);
																	return null;
																});

															if (stream) {
																recording = true;
																const tracks = stream.getTracks();
																tracks.forEach((track) => track.stop());
															}
															stream = null;
														} catch {
															toast.error($i18n.t('Permission denied when accessing microphone'));
														}
													}}
													aria-label="Voice Input"
												>
													<svg
														xmlns="http://www.w3.org/2000/svg"
														viewBox="0 0 20 20"
														fill="currentColor"
														class="w-5 h-5 translate-y-[0.5px]"
													>
														<path d="M7 4a3 3 0 016 0v6a3 3 0 11-6 0V4z" />
														<path
															d="M5.5 9.643a.75.75 0 00-1.5 0V10c0 3.06 2.29 5.585 5.25 5.954V17.5h-1.5a.75.75 0 000 1.5h4.5a.75.75 0 000-1.5h-1.5v-1.546A6.001 6.001 0 0016 10v-.357a.75.75 0 00-1.5 0V10a4.5 4.5 0 01-9 0v-.357z"
														/>
													</svg>
												</button>
											</Tooltip>
										{/if}

										{#if !history.currentId || history.messages[history.currentId]?.done == true}
											{#if prompt === '' && files.length === 0}
												<div class=" flex items-center">
													<Tooltip content={$i18n.t('Call')}>
														<button
															class=" {webSearchEnabled ||
															($settings?.webSearch ?? false) === 'always'
																? 'bg-blue-500 text-white hover:bg-blue-400 '
																: 'bg-black text-white hover:bg-gray-900 dark:bg-white dark:text-black dark:hover:bg-gray-100'} transition rounded-full p-1.5 self-center"
															type="button"
															on:click={async () => {
																if (selectedModels.length > 1) {
																	toast.error($i18n.t('Select only one model to call'));

																	return;
																}

																if ($config.audio.stt.engine === 'web') {
																	toast.error(
																		$i18n.t(
																			'Call feature is not supported when using Web STT engine'
																		)
																	);

																	return;
																}
																// check if user has access to getUserMedia
																try {
																	let stream = await navigator.mediaDevices.getUserMedia({
																		audio: true
																	});
																	// If the user grants the permission, proceed to show the call overlay

																	if (stream) {
																		const tracks = stream.getTracks();
																		tracks.forEach((track) => track.stop());
																	}

																	stream = null;

																	if ($settings.audio?.tts?.engine === 'browser-kokoro') {
																		// If the user has not initialized the TTS worker, initialize it
																		if (!$TTSWorker) {
																			await TTSWorker.set(
																				new KokoroWorker({
																					dtype: $settings.audio?.tts?.engineConfig?.dtype ?? 'fp32'
																				})
																			);

																			await $TTSWorker.init();
																		}
																	}

																	showCallOverlay.set(true);
																	showControls.set(true);
																} catch (err) {
																	// If the user denies the permission or an error occurs, show an error message
																	toast.error(
																		$i18n.t('Permission denied when accessing media devices')
																	);
																}
															}}
															aria-label="Call"
														>
															<Headphone className="size-5" />
														</button>
													</Tooltip>
												</div>
											{:else}
												<div class=" flex items-center">
													<Tooltip content={$i18n.t('Send message')}>
														<button
															id="send-message-button"
															class="{!(prompt === '' && files.length === 0)
																? webSearchEnabled || ($settings?.webSearch ?? false) === 'always'
																	? 'bg-blue-500 text-white hover:bg-blue-400 '
																	: 'bg-black text-white hover:bg-gray-900 dark:bg-white dark:text-black dark:hover:bg-gray-100 '
																: 'text-white bg-gray-200 dark:text-gray-900 dark:bg-gray-700 disabled'} transition rounded-full p-1.5 self-center"
															type="submit"
															disabled={prompt === '' && files.length === 0}
														>
															<svg
																xmlns="http://www.w3.org/2000/svg"
																viewBox="0 0 16 16"
																fill="currentColor"
																class="size-5"
															>
																<path
																	fill-rule="evenodd"
																	d="M8 14a.75.75 0 0 1-.75-.75V4.56L4.03 7.78a.75.75 0 0 1-1.06-1.06l4.5-4.5a.75.75 0 0 1 1.06 0l4.5 4.5a.75.75 0 0 1-1.06 1.06L8.75 4.56v8.69A.75.75 0 0 1 8 14Z"
																	clip-rule="evenodd"
																/>
															</svg>
														</button>
													</Tooltip>
												</div>
											{/if}
										{:else}
											<div class=" flex items-center">
												<Tooltip content={$i18n.t('Stop')}>
													<button
														class="bg-white hover:bg-gray-100 text-gray-800 dark:bg-gray-700 dark:text-white dark:hover:bg-gray-800 transition rounded-full p-1.5"
														on:click={() => {
															stopResponse();
														}}
													>
														<svg
															xmlns="http://www.w3.org/2000/svg"
															viewBox="0 0 24 24"
															fill="currentColor"
															class="size-5"
														>
															<path
																fill-rule="evenodd"
																d="M2.25 12c0-5.385 4.365-9.75 9.75-9.75s9.75 4.365 9.75 9.75-4.365 9.75-9.75 9.75S2.25 17.385 2.25 12zm6-2.438c0-.724.588-1.312 1.313-1.312h4.874c.725 0 1.313.588 1.313 1.313v4.874c0 .725-.588 1.313-1.313 1.313H9.564a1.312 1.312 0 01-1.313-1.313V9.564z"
																clip-rule="evenodd"
															/>
														</svg>
													</button>
												</Tooltip>
											</div>
										{/if}
									</div>
								</div>
							</div>
						</form>
					{/if}
				</div>
			</div>
		</div>
	</div>
{/if}
