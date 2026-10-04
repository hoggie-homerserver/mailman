<!-- yes you are in the right file -->

<script lang="ts">
	import Dots from '$lib/Dots.svelte';
	import autoAnimate from '@formkit/auto-animate';
	import { MultiSelect } from 'svelte-multiselect';
	import IconCheck from '~icons/tabler/Check.svelte';
	import IconDeviceFloppy from '~icons/tabler/DeviceFloppy.svelte';
	import IconEye from '~icons/tabler/Eye.svelte';
	import IconFile from '~icons/tabler/File.svelte';
	import IconFileDescription from '~icons/tabler/FileDescription.svelte';
	import IconFilter from '~icons/tabler/Filter.svelte';
	import IconHeading from '~icons/tabler/Heading.svelte';
	import IconHelp from '~icons/tabler/Help.svelte';
	import IconListNumbers from '~icons/tabler/ListNumbers.svelte';
	import IconLogin2 from '~icons/tabler/Login-2.svelte';
	import IconMoodHappy from '~icons/tabler/MoodHappy.svelte';
	import IconPaperclip from '~icons/tabler/Paperclip.svelte';
	import IconPlus from '~icons/tabler/Plus.svelte';
	import IconUpload from '~icons/tabler/Upload.svelte';
	import IconVersions from '~icons/tabler/Versions.svelte';

	let borderStyle = "border-zinc-800"
	let is_drag =false

	let authed = 0;
	let page = 0;
	let body: HTMLDivElement;
	let dpfile: File | null | undefined;
	let rpfile: File | null | undefined;

	let dphSelect: HTMLSelectElement;
	let smiSelect: HTMLSelectElement;
	let modSelect: HTMLSelectElement;

	let selectedDphPack: number;
	let selectedSmiPack: string;
	let selectedModPack: string;

	let selectedDphIndex: any;
	let selectedSmiIndex: any;
	let selectedModIndex: any;

	// Project Selection
	function selectDph() {
		// selectedDphIndex = dphSelect.selectedOptions[0].value
		console.log("Selected Datapack Hub Pack: " + JSON.stringify(dphPacks[dphSelect.selectedIndex - 1]))
		selectedDphPack = dphPacks[dphSelect.selectedIndex - 1].ID;
	}

	// Project Selection
	function selectSmi() {
		// selectedSmiIndex = smiSelect.selectedOptions[0].value
		console.log("Selected Smithed Pack: " + JSON.stringify(smithedPacks[smiSelect.selectedIndex - 1]))
		selectedSmiPack = smithedPacks[smiSelect.selectedIndex - 1].id;
	}

	// Project Selection
	function selectMod() {
		// selectedModIndex = modSelect.selectedOptions[0].value
		console.log("Selected Modrinth Pack: " + JSON.stringify(modPacks[modSelect.selectedIndex - 1]))
		selectedModPack = modPacks[modSelect.selectedIndex - 1].id;
	}

	// Smithed stuff
	import { initializeApp } from 'firebase/app';
	import { getAuth, signInWithEmailAndPassword, getIdToken, type User } from 'firebase/auth';
	import Button from '$lib/Button.svelte';
	import { version } from '$app/environment';

	initializeApp({
		databaseURL: 'https://mc-smithed-default-rtdb.firebaseio.com',
		apiKey: 'AIzaSyDX-vLCBhO8StKAxnpvQ2EW8lz3kzYn4Qk',
		authDomain: 'mc-smithed.firebaseapp.com',
		projectId: 'mc-smithed',
		storageBucket: 'mc-smithed.appspot.com',
		messagingSenderId: '574184244682',
		appId: '1:574184244682:web:498d168c09b39e4f0d7b33',
		measurementId: 'G-40SRKC35Z0'
	});

	let smithedToken: string | null = null;
	let smithedUser: User | null = null;

	async function logIntoSmithed(email: string, password: string) {
		const auth = getAuth();

		try {
			const userCredential = await signInWithEmailAndPassword(auth, email, password);
			const user = userCredential.user;
			const tok = await getIdToken(user);
			const smithedApiTokenReq = await fetch(
				`https://api.smithed.dev/v2/token?token=${tok}&expires=1d`
			);
			const smithedApiToken = await smithedApiTokenReq.text();
			smithedToken = smithedApiToken;
			smithedUser = userCredential.user;
		} catch (error: any) {
			alert('There was an error: ' + error.message);
		}

		return { token: smithedToken, user: smithedUser };
	}

	// Auth
	let smithedEmail: string;
	let smithedPassword: string;

	let authedSmithed: null | {
		displayName: string;
		cleanName: string;
		creationTime: number;
		uid: string;
		pfp: string | undefined;
	} = null;
	let smithedPacks: {
		id: string;
		display: {
			name: string;
			description: string;
			icon: string;
			hidden: boolean;
			webPage: string | undefined;
		};
		versions: string[];
		categories: string[];
	}[] = [];

	async function authSmithed() {
		let smithed = await logIntoSmithed(smithedEmail, smithedPassword);
		if (smithed.user) {
			let smithedUserData = await fetch('https://api.smithed.dev/v2/users/' + smithed.user.uid);
			if (smithedUserData.ok) {
				let smithedUserParsed = await smithedUserData.json();

				let smithedUserPacks = await fetch(
					`https://api.smithed.dev/v2/users/${smithed.user.uid}/packs`
				);
				if (smithedUserPacks.ok) {
					let tempPacks = smithedPacks;
					let smithedUserPacksParsed = (await smithedUserPacks.json()) as string[];
					for (let id in smithedUserPacksParsed) {
						let smithedPack = await fetch(
							`https://api.smithed.dev/v2/packs/${smithedUserPacksParsed[id]}`
						);
						if (smithedPack.ok) {
							let smithedPackParsed = await smithedPack.json();
							tempPacks.push(smithedPackParsed);
						}
					}
					smithedPacks = tempPacks;
					console.log('Smithed Packs: ' + smithedPacks);
				} else {
					alert('Error fetching your Smithed packs.');
				}

				authedSmithed = smithedUserParsed;
				authed++;
			} else {
				alert('Error loading your Smithed user data.');
			}
		}
	}

	let dphToken: string;

	let authedDph: null | {
		id: number;
		bio: string;
		profile_icon: string;
		role: string;
		username: string;
		badges: string[];
	} = null;

	let dphPacks: {
		title: string;
		ID: number;
	}[] = [];

	async function authDph() {
		let dphUser = await fetch(`https://api.datapackhub.net/user/me`, {
			headers: {
				Authorization: 'Basic ' + dphToken
			}
		});
		if (dphUser.ok) {
			let dphUserParsed = await dphUser.json();
			authedDph = dphUserParsed;

			let dphProjs = await fetch(
				`https://api.datapackhub.net/user/${dphUserParsed.username}/projects`
			);
			if (dphProjs.ok) {
				let dphProjsParsed = await dphProjs.json();
				dphPacks = dphProjsParsed.result;
			}
			authed++;
		}
	}

	let modToken: string;

	let authedModrinth: null | {
		username: string;
		id: string;
	} = null;

	let modPacks: {
		title: string;
		id: string;
	}[] = [];

	async function authModrinth() {
		let modUser = await fetch(`https://api.modrinth.com/v2/user`, {
			headers: {
				Authorization: modToken
			}
		});
		if (modUser.ok) {
			let modUserParsed = (await modUser.json()) as { username: string; id: string };
			authedModrinth = modUserParsed;

			let modProjs = await fetch(`https://api.modrinth.com/v2/user/${modUserParsed.id}/projects`, {
				headers: {
					Authorization: modToken
				}
			});
			if (modProjs.ok) {
				let modProjsParsed = (await modProjs.json()) as { title: string; id: string }[];
				modPacks = modProjsParsed;
			}
			authed++;
		}
	}

	// MODRINTH DEPENDENCIES
	let modDeps: {
		title: string;
		id: string;
		icon_url: string;
		dep_type: 'required' | 'optional' | 'incompatible' | 'embedded';
	}[] = [];

	let addModDep = false;
	let modDepType: 'required' | 'optional' | 'incompatible' | 'embedded' = 'required';
	let modDepValue: string = '';

	async function createModDep() {
		if (modDepValue == '') return alert('Make sure to set a value!');
		let projReq = await fetch('https://api.modrinth.com/v2/project/' + modDepValue);
		if (projReq.ok) {
			let projReqParsed = (await projReq.json()) as {
				title: string;
				id: string;
				icon_url: string;
				dep_type: 'required' | 'optional' | 'incompatible' | 'embedded';
			};
			projReqParsed.dep_type = modDepType;
			modDeps.push(projReqParsed);
			console.log(modDeps);
			addModDep = false;
		} else {
			alert('Could not get the Modrinth project. Does it exist?');
		}
	}

	// SMITHED DEPENDENCIES
	let addSmiDep = false;
	let smiDepValue: string = '';
	let smiDepVerValue: string = '';
	let smiDeps: {
		id: string;
		version: string;
		name: string;
		icon: string;
	}[] = [];

	async function createSmiDep() {
		if (smiDepValue == '' || smiDepVerValue == '')
			return alert('Make sure to set a value and version ID!');
		let projReq = await fetch('https://api.smithed.dev/v2/packs/' + smiDepValue + '/versions');
		let projDataReq = await fetch('https://api.smithed.dev/v2/packs/' + smiDepValue);
		if (projReq.ok && projDataReq.ok) {
			let projReqParsed = (await projReq.json()) as { name: string }[];
			let projDataParsed = (await projDataReq.json()) as {
				id: string;
				display: { name: string; icon: string };
			};

			const versionExists = projReqParsed.some((versionObj) => versionObj.name === smiDepVerValue);
			if (!versionExists) return alert('Are you sure the version exists?');

			smiDeps.push({
				id: projDataParsed.id,
				version: smiDepVerValue,
				name: projDataParsed.display.name,
				icon: projDataParsed.display.icon
			});
			addSmiDep = false;
		} else {
			return alert('Could not get the Smithed project. Does it exist?');
		}
	}

	let rpLabel: HTMLLabelElement;
	function uploadRP() {
		rpfile = rpInput?.files?.item(0);
	}

	// --------------- VERSION DETAILS ---------------
	let fileInput: HTMLInputElement;

	let title: string = '';
	let versionCode: string = '';
	let changelog: string = '';
	let mcVersions: string[] = [];
	// dependencies are separate
	let releaseChannel: 'release' | 'beta' | 'alpha' = 'release';
	let rpInput: HTMLInputElement;

	// ------------ THE RELEVANT FUNCTION ------------

	// Status
	let dphStatus = 'Waiting...';
	let modStatus = 'Waiting...';
	let smiStatus = 'Waiting...';

	async function postFunction() {
		if (!dpfile)
			return alert(
				'Something went wrong. Please upload your project again and try again. If the issue persists, please contact us on our Discord.'
			);

		let file_link_data: {
			datapack?: string,
			resourcepack?: string
		} = {};

		if (authedDph && selectedDphPack) {
			console.log("Posting DPH version: " + selectedDphPack)
			dphStatus = 'Uploading...';
			const formData = new FormData();
			formData.append('name', title);
			formData.append('description', changelog);
			formData.append('minecraft_versions', JSON.stringify(generate_datapackhub_versions(mcVersions)));
			formData.append('version_code', versionCode);
			formData.append('filename', dpfile.name);
			formData.append('primary_download', dpfile, dpfile.name);

			if (rpfile) {
				formData.append('v_rp', rpfile, rpfile.name);
			}

			let dphReq = await fetch(`https://api.datapackhub.net/versions/new/${selectedDphPack}`, {
				method: 'POST',
				headers: {
					Authorization: 'Basic ' + dphToken
				},
				body: formData
			});

			if (dphReq.ok) {
				dphStatus = 'Done!';
				let dph_data = await dphReq.json();
				file_link_data.datapack = dph_data.primary_download
				if (rpfile) file_link_data.resourcepack = dph_data.resource_pack_download
			} else dphStatus = 'Failed.';
		}

		if (authedModrinth && selectedModPack) {
			console.log("Posting MOD version: " + selectedModPack)
			modStatus = 'Uploading...';
			const formData = new FormData();

			var modrinth_dependencies = []

			for (const i of modDeps) {
				modrinth_dependencies.push({
					project_id: i.id,
					dependency_type: i.dep_type
				})
			}

			let data = {
				name: title,
				version_number: versionCode,
				changelog: changelog,
				game_versions: mcVersions,
				loaders: ["datapack"],
				featured: true,
				project_id: selectedModPack,
				file_types: {},
				file_parts: [dpfile.name + '-primary'],
				version_type: releaseChannel,
				dependencies: modrinth_dependencies
			}

			if (rpfile) {
				data.file_types = {"resource_pack":"required-resource-pack"}
				data.file_parts.push("resource_pack")
			}

			formData.append("data", JSON.stringify(data))
			formData.append(dpfile.name + '-primary', dpfile, dpfile.name);
			if(rpfile) formData.append("resource-pack",rpfile,rpfile?.name)

			let modReq = await fetch(`https://api.modrinth.com/v2/version`, {
				method: 'POST',
				headers: {
					Authorization: modToken
				},
				body: formData
			});

			if (modReq.ok) {
				modStatus = 'Done!';
				if(file_link_data.datapack) {
					let mod_out = await modReq.json()
					file_link_data.datapack = mod_out.files[0].url
					if (rpfile) file_link_data.resourcepack = mod_out.files[1].url
				}
				
				
			} else {
				let mod_out = await modReq.json()
				modStatus = `(${modReq.status}) Failed: ` + mod_out.description
			};
		}

		if (authedSmithed && selectedSmiPack) {
			if (file_link_data.datapack) {
				console.log("Posting SMI version: " + selectedSmiPack)
				smiStatus = 'Uploading...';
				let smiUploadData: {
					name: string;
					downloads: { datapack?: string; resourcepack?: string };
					supports: string[];
					dependencies?: { id: string; version: string }[];
				} = {
					name: versionCode,
					downloads: {
						datapack: file_link_data.datapack
					},
					supports: generate_smithed_versions(mcVersions),
					dependencies: smiDeps.map(i => {return {id: i.id, version: i.version}})
				};

				if (rpfile) smiUploadData.downloads.resourcepack = file_link_data.resourcepack

				let smiReq = await fetch(
					`https://api.smithed.dev/v2/packs/${selectedSmiPack}/versions?token=${smithedToken}&version=${versionCode}`,
					{
						method: 'POST',
						headers: {
							'content-type': 'application/json'
						},
						body: JSON.stringify({
							data: smiUploadData
						})
					}
				);

				if (smiReq.ok) {
					smiStatus = 'Done!';
				} else {
					smiStatus = 'Failed.';
				}
			} else {
				smiStatus = 'Failed: Uploading to smithed cannot work if uploading to the other sites did not work.';
			}
		}
	}

	function drop(e: DragEvent) {
		page = 1;
		dpfile = e.dataTransfer?.items[0].getAsFile();
		borderStyle = "border-zinc-800"
	}

	function upload() {
		page = 1;
		const dpFileInput = document.getElementById('upload') as HTMLInputElement;
		dpfile = dpFileInput.files?.item(0);
	}

	const minecraftVersions = [
		{"format":4,"label":"1.13"},
		{"format":5,"label":"1.15"},
		{"format":6,"label":"1.16.2"},
		{"format":7,"label":"1.17"},
		{"format":8,"label":"1.18"},
		{"format":9,"label":"1.18.2"},
		{"format":10,"label":"1.19"},
		{"format":12,"label":"1.19.4"},
		{"format":15,"label":"1.20"},
		{"format":18,"label":"1.20.2"},
		{"format":26,"label":"1.20.3"},
		{"format":41, "label":"1.20.5"},
		{"format":41, "label":"1.20.6"},
		{"format":48, "label":"1.21"},
		{"format":48, "label":"1.21.1"},
		{"format":57, "label":"1.21.2"},
		{"format":57, "label":"1.21.3"},
		{"format":61, "label":"1.21.4"},
		{"format":71, "label":"1.21.5"},
		{"format":88, "label":"1.21.9"},
		{"format":94, "label":"1.21.11"},
		{"format":121, "label":"26.3"}
	];

	const generate_datapackhub_versions = function(selected: Array<string>) {return selected.map(i => {if(i == "1.17") {return "1.17.x"} else return i })}

	const generate_smithed_versions = function(selected: Array<string>) {
		let temp = selected
		if(temp.includes("1.17")) temp.push("1.17.1")
		if(temp.includes("1.18")) temp.push("1.18.1")
		if(temp.includes("1.20")) temp.push("1.20.1")
		return temp
	}
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<div
	class="flex justify-around items-center min-h-screen h-full py-3"
	on:dragover|preventDefault={() => {borderStyle = "border-blue-500"}}
	on:dragenter|preventDefault={() => {borderStyle = "border-blue-500"}}
	on:dragexit|preventDefault={() => {borderStyle = "border-zinc-800"}}
	on:dragleave|preventDefault={() => {borderStyle = "border-zinc-800"}}
	on:drop|preventDefault={(event) => drop(event)}
>
	<div class="flex flex-col items-center space-y-10">
		<div class="flex flex-col items-center space-y-2">
			<div class="flex items-center">
				<img src="./mailman.png" class="h-20 w-20 mr-2" alt="logo" />
				<h1>
					<b class="text-5xl md:text-7xl">Mailman</b>
					<span class="text-base hidden md:inline">by <a href="https://datapackhub.net">Datapack Hub</a></span>
				</h1>
			</div>
			<p>
				Upload your datapack versions to
				<a class="text-green-600" href="https://modrinth.com" target="_blank"
					>Modrinth</a
				>, and
				<a class="text-blue-500" href="https://smithed.net" target="_blank"
					>Smithed</a
				> at the same time!
			</p>
		</div>
		<div>
			{#if page > 0}
			<div class="bg-black rounded-t-xl border-zinc-800 border-t-2 border-l-2 border-r-2 p-3 max-h-[50%] overflow-y-auto styled-scrollbar relative max-w-screen-lg font-bold flex items-center space-x-2">
				{#if page == 1}
				<IconLogin2 class="text-xl"/>
				<span>Authenticate and choose your projects</span>
				{:else if page == 2}
				<IconFileDescription class="text-xl"/>
				<span>Enter version details</span>
				{:else if page == 3}
				<IconEye class="text-xl" />
				<span>Preview your new version</span>
				{:else if page == 4}
				<IconUpload class="text-xl" />
				<span>Uploading version...</span>
				{/if}
			</div>
			{/if}
			<div
				class="bg-zinc-950 {borderStyle} {page == 0 ? "border-2 rounded-xl" : "border-b-2 border-l-2 border-r-2 rounded-b-xl"} p-3 max-h-[50%] overflow-y-auto styled-scrollbar relative max-w-screen-lg"
				use:autoAnimate
				bind:this={body}
			>
				<input name="upload" id="upload" type="file" class="hidden" accept=".zip" on:change={upload} />
				{#if page == 0}
					<label for="upload" class="cursor-pointer flex">
						<div class="p-5 px-10 hover:px-12 transition-all flex flex-col items-center">
							<div class="flex space-x-1">
								<IconUpload />
								<p class=""><b>Click</b> to upload a datapack.</p>
							</div>
							<p class="text-sm text-zinc-500">You can also drop a zip file anywhere on this page.
						</div>
						
					</label>
				{:else if page == 1}
					<div
						class="flex flex-col md:flex-row justify-around space-y-2 md:space-y-0 md:space-x-2 w-full"
					>

						<!-- MODRINTH -->
						<div class="bg-zinc-950 w-full rounded-xl border-zinc-800 border-2 p-3 space-y-2">
							<div class="flex items-center space-x-2">
								<img src="/modrinth.svg" class="h-8" alt="logo" />
								<b>Modrinth</b>
							</div>
							{#if !authedModrinth}
								<input
									class="focus:outline-green-600"
									placeholder="modrinth.com API token"
									bind:value={modToken}
								/>
								<Button click={authModrinth}
									><IconLogin2 />
									<p>Authenticate</p></Button
								>
							{:else}
								<p><b>User:</b> <span>{authedModrinth.username}</span></p>
								<select class="focus:outline-green-500" on:change={selectMod} bind:this={modSelect} bind:value={selectedModIndex}>
									<option>-- Select a pack --</option>
									{#each modPacks as pack}
										<option>{pack.title}</option>
									{/each}
								</select>
							{/if}
						</div>

						<!-- SMITHED -->
						<div class="bg-zinc-950 w-full rounded-xl border-zinc-800 border-2 p-3 space-y-2">
							<div class="flex items-center space-x-2">
								<img src="/smithed.svg" class="h-8 rounded-full bg-[#1b48c4]" alt="logo" />
								<b>Smithed</b>
							</div>
							{#if !authedSmithed}
								<input
									class="focus:outline-blue-500"
									placeholder="smithed.dev email"
									bind:value={smithedEmail}
								/>
								<input
									type="password"
									class="focus:outline-blue-500"
									placeholder="smithed.dev password"
									bind:value={smithedPassword}
								/>
								<Button click={authSmithed}><IconLogin2 /><span>Log In</span></Button>
							{:else}
								<p><b>User:</b> <span>{authedSmithed.displayName}</span></p>
								<select class="focus:outline-blue-500" on:change={selectSmi} bind:this={smiSelect} bind:value={selectedSmiIndex}>
									<option>-- Select a pack --</option>
									{#each smithedPacks as pack}
										<option>{pack.display.name}</option>
									{/each}
								</select>
							{/if}
						</div>
					</div>
					<p class="text-sm text-zinc-500 mt-2">
						Nothing entered here is sent to us. All code is client-side, which means that we can't
						view anything you enter here, including API tokens.
					</p>
				{:else if page == 2}
					<h2 class="text-sm -1 flex space-x-1 items-center">
						<IconHeading /><span>Title</span>
						<Dots dots={['mod', 'dph']} />
					</h2>
					<input placeholder="The Snails Update" bind:value={title} />

					<h2 class="text-sm mt-5 mb-1 flex space-x-1 items-center">
						<IconListNumbers /><span>Version Code</span>
						<Dots dots={['mod', 'smi', 'dph']} />
					</h2>
					<input placeholder="v1.2.3" bind:value={versionCode} />

					<h2 class="text-sm mt-5 mb-1 flex space-x-1 items-center">
						<IconFileDescription /><span>Changelog</span>
						<Dots dots={['mod', 'dph']} />
					</h2>
					<textarea
						placeholder="This version adds a snail you can worship, among other bug fixes"
						bind:value={changelog}
					/>
					<h2 class="text-sm mb-1 mt-3 flex space-x-1 items-center">
						<IconVersions /><span
							>Supported Minecraft Versions <Dots dots={['mod', 'dph', 'smi']} /></span
						>
					</h2>
					<MultiSelect
						options={minecraftVersions.map(entry => entry.label)}
						placeholder="1.17, 1.18"
						bind:value={mcVersions}
					/>
					

					<div class="flex">
						<div class="w-1/3">
							<h2 class="text-sm mt-5 mb-1 flex space-x-1 items-center">
								<IconFilter /><span>Modrinth Dependencies</span>
								<Dots dots={['mod']} />
							</h2>
							{#if addModDep == false}
								<button
									class="bg-zinc-900 rounded-md h-8 p-1 px-2 text-zinc-400 hover:text-zinc-300 flex items-center space-x-1"
									on:click={() => (addModDep = true)}
									><IconPlus /><span>Add Dependency</span></button
								>
								{#key modDeps}
									{#each modDeps as dep}
										<!-- svelte-ignore a11y-click-events-have-key-events -->
										<!-- svelte-ignore a11y-no-noninteractive-element-interactions -->
										<!-- svelte-ignore a11y-no-noninteractive-element-interactions -->
										<div class="flex items-center space-x-2 mb-1">
											<!-- svelte-ignore a11y-missing-attribute -->
											<img src={dep.icon_url} class="h-8 rounded-md" />
											<!-- svelte-ignore a11y-click-events-have-key-events -->
											<h1
												class="hover:line-through cursor-pointer"
												on:click={() => (modDeps = modDeps.filter((item) => item != dep))}
											>
												{dep.title}
											</h1>
											{#if dep.dep_type == 'required'}
												<h1 class="font-light text-sm text-green-500">• Required</h1>
											{:else if dep.dep_type == 'optional'}
												<h1 class="font-light text-sm text-yellow-500">• Optional</h1>
											{:else if dep.dep_type == 'incompatible'}
												<h1 class="font-light text-sm text-red-500">• Incompatible</h1>
											{:else if dep.dep_type == 'embedded'}
												<h1 class="font-light text-sm text-pink-500">• Embedded</h1>
											{/if}
										</div>
									{/each}
								{/key}
							{:else}
								<div class="space-y-1 w-2/3">
									<input
										placeholder="Project ID or slug"
										bind:value={modDepValue}
									/>
									<select bind:value={modDepType}>
										<option value="required">Required Dependency</option>
										<option value="optional">Optional Dependency</option>
										<option value="incompatible">Incompatible</option>
										<option value="embedded">Embedded</option>
									</select>
									<div class="flex space-x-1">
										<button
											class="bg-orange-600 rounded-md p-1 px-2 text-md"
											on:click={createModDep}>Add</button
										>
										<button
											class="bg-zinc-800 rounded-md p-1 px-2 text-md"
											on:click={() => (addModDep = false)}>Cancel</button
										>
									</div>
								</div>
							{/if}
						</div>
						<div class="w-1/3">
							<h2 class="text-sm mt-5 mb-1 flex space-x-1 items-center">
								<IconFilter /><span>Smithed Dependencies</span>
								<Dots dots={['smi']} />
							</h2>
							{#each [] ?? [] as dep}
								<h1>{dep}</h1>
							{/each}
							{#if addSmiDep == false}
								<button
									class="bg-zinc-900 rounded-md h-8 p-1 px-2 text-zinc-400 hover:text-zinc-300 flex items-center space-x-1"
									on:click={() => (addSmiDep = true)}
									><IconPlus /><span>Add Dependency</span></button
								>
								{#key smiDeps}
									{#each smiDeps as dep}
										<div class="flex items-center space-x-2 mb-1">
											<!-- svelte-ignore a11y-missing-attribute -->
											<img src={dep.icon} class="h-8 rounded-md" />
											<h1 class="hover:line-through cursor-pointer"
											on:click={() => (smiDeps = smiDeps.filter((item) => item != dep))}>{dep.name}</h1>
											<h1 class="font-light text-sm">(v{dep.version})</h1>
										</div>
									{/each}
								{/key}
							{:else}
								<div class="space-y-1 w-2/3">
									<input placeholder="Project ID" bind:value={smiDepValue} />
									<input placeholder="Version code" bind:value={smiDepVerValue} />
									<div class="flex space-x-1">
										<button
											class="bg-orange-600 rounded-md p-1 px-2 text-md"
											on:click={createSmiDep}>Add</button
										>
										<button
											class="bg-zinc-800 rounded-md p-1 px-2 text-md"
											on:click={() => (addSmiDep = false)}>Cancel</button
										>
									</div>
								</div>
							{/if}
						</div>
					</div>

					<h2 class="text-sm mt-5 mb-1 flex space-x-1 items-center">
						<IconDeviceFloppy /><span>Release Channel</span>
						<Dots dots={['mod']} />
					</h2>
					<select bind:value={releaseChannel}>
						<option value="release">Stable Release</option>
						<option value="beta">Beta</option>
						<option value="alpha">Alpha</option>
					</select>

					<h2 class="text-sm mt-5 mb-1 flex space-x-1 items-center">
						<IconPaperclip /><span>Resource Pack</span>
						<Dots dots={['mod', 'smi', 'dph']} />
					</h2>

					<input
						type="file"
						class="hidden"
						name="rp"
						id="rp"
						bind:this={rpInput}
						on:change={uploadRP}
					/>
					<label
						for="rp"
						bind:this={rpLabel}
						class="items-center flex w-fit bg-zinc-900 text-zinc-400 rounded-md h-8 p-2 px-2 cursor-pointer  hover:text-zinc-300 mb-1 space-x-1"
						><IconUpload /><span>{#if !rpfile}Add Resource Pack file{:else}{rpfile.name} (click to change){/if}</span></label
					>

					<p class="text-sm text-zinc-500 mt-2 flex space-x-1 items-center">
						<IconMoodHappy /><span
							>Nothing entered here is sent to us. All code is client-side, which means that we
							can't view anything you enter here, including API tokens.</span
						>
					</p>
				{:else if page == 3}
					<div class="p-3 bg-zinc-900 rounded-xl max-w-full">
						<p class="text-zinc-400 flex items-center space-x-1">
							<IconHeading class="text-zinc-200" /><b class="text-zinc-200">Title: </b>
							<span>{title}</span>
						</p>
						<p class="text-zinc-400 flex items-center space-x-1">
							<IconListNumbers class="text-zinc-200" /><b class="text-zinc-200">Version Code: </b>
							<span>{versionCode}</span>
						</p>
						{#if rpfile}
							<p class="text-zinc-400 flex items-center space-x-1">
								<IconFile class="text-zinc-200" /><b class="text-zinc-200">Resource Pack: </b>
								<span>{rpfile.name} (Required)</span>
							</p>
						{/if}
						<p class="text-zinc-400 flex items-center space-x-1 mb-3">
							<IconVersions class="text-zinc-200" /><b class="text-zinc-200"
								>Supports Minecraft Versions:
							</b> <span>{mcVersions.join(', ')}</span>
						</p>

						<p class="text-zinc-400 flex items-center space-x-1 mb-3">
							<IconFileDescription class="text-zinc-200" /><b class="text-zinc-200">Changelog: </b>
							<span>{changelog}</span>
						</p>

						{#if modDeps.length > 0}
							<p class="text-zinc-400 flex items-center space-x-1">
								<IconFilter class="text-zinc-200" /><b class="text-zinc-200"
									>Modrinth Dependencies:
								</b> <span>{modDeps.map((obj) => obj.title).join(', ')}</span>
							</p>
						{/if}
						{#if smiDeps.length > 0}
							<p class="text-zinc-400 flex items-center space-x-1 mb-3">
								<IconFilter class="text-zinc-200" /><b class="text-zinc-200"
									>Smithed Dependencies:
								</b> <span>{smiDeps.map((obj) => obj.name).join(', ')}</span>
							</p>
						{/if}

						<p class="text-zinc-400 flex items-center space-x-1">
							<IconDeviceFloppy class="text-zinc-200" /><b class="text-zinc-200"
								>Release Channel:
							</b> <span>{releaseChannel}</span>
						</p>
					</div>
				{:else if page == 4}
					<div class="p-3 bg-zinc-900 rounded-xl max-w-full space-y-3">
						{#if authedDph}
						<div class="flex items-center space-x-2">
							<img src="./dph.png" class="h-8 w-8" alt="logo" />
							<b class="text-orange-600">Datapack Hub: </b>
							<span class="italic text-gray-400">{dphStatus}</span>
						</div>
						{/if}
						{#if authedModrinth}
						<div class="flex items-center space-x-2">
							<img src="/modrinth.svg" class="h-8" alt="logo">
							<b class="text-green-400">Modrinth: </b> 
							<span class="italic text-gray-400">{modStatus}</span>
						</div>
						{/if}
						{#if authedSmithed}
						<div class="flex items-center space-x-2">
							<img src="/smithed.svg" class="h-8 rounded-full bg-[#1b48c4]" alt="logo" />
							<b class="text-blue-600">Smithed: </b>
							<span class="italic text-gray-400">{smiStatus}</span>
						</div>
						{/if}
					</div>
				{/if}
			</div>
			{#if page != 0 && page != 4}
				<div class="mt-3 flex lg:min-w-[50vw] min-w-[90vw]">
					<div class="w-1/3">
						<button
							class="bg-zinc-700 rounded-md p-2 px-3 text-lg"
							on:click={() => {
								if (page > 0) page--;
								if (page == 0) fileInput.files = null;
							}}>Back</button
						>
					</div>
					<div class="w-1/3 flex items-center justify-around text-blue-400">
						<div class="flex space-x-1 items-center">
							<IconHelp />
							<a href="https://discord.datapackhub.net">Help</a>
						</div>
					</div>
					<div class="w-1/3">
						<button
							class="transition-all {authed > 0
								? 'bg-orange-600 hover:px-4 hover:bg-orange-500'
								: 'bg-orange-600/40 cursor-default'} {page == 3
								? 'bg-green-700'
								: ''} rounded-md p-2 px-3 float-right text-lg font-semibold flex items-center space-x-1"
							on:click={() => {
								if (page == 3) postFunction();
								if (authed > 0) page++;
							}}
							>{#if page == 3}<IconCheck />{/if}<span>{page == 3 ? 'Upload Version' : 'Next'}</span
							></button
						>
					</div>
				</div>
			{/if}
		</div>
	</div>
</div>

<div class="fixed top-0 left-0 w-screen h-screen {is_drag ? "" : "pointer-events-none"}" 
	on:dragover|preventDefault={() => {borderStyle = "border-blue-500"; is_drag = true}}
	on:dragenter|preventDefault={() => {borderStyle = "border-blue-500"; is_drag = true}}
	on:dragexit|preventDefault={() => {borderStyle = "border-zinc-800"; is_drag = false}}
	on:dragleave|preventDefault={() => {borderStyle = "border-zinc-800"; is_drag = false}}
	on:drop|preventDefault={(event) => {drop(event); is_drag = false}}>

</div>

<footer class="fixed bottom-3 left-3 pr-7 text-zinc-500 flex w-full justify-between">
	<span>© Datapack Hub, 2023</span>
	<a href="https://github.com/Datapack-Hub/mailman" target="_blank">This is open source!</a>
</footer>

<style>
	:root {
		--sms-bg: theme(colors.zinc.900);
		--sms-border: 0px solid theme(colors.zinc.200);
		--sms-selected-bg: theme(colors.zinc.700);
		--sms-remove-btn-hover-bg: theme(colors.orange.500);
		--sms-options-bg: theme(colors.zinc.800);
		--sms-text-color: theme(colors.slate.100);
		--sms-min-height: height: 3rem;
		--sms-border-radius: 0.375rem;
		--sms-placeholder-color: theme(colors.zinc.600);
	}
</style>
