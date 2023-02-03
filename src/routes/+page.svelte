<script lang="ts">
	import { initializeApp } from 'firebase/app';
	import { getStorage, ref, uploadBytes } from 'firebase/storage';
	import { getFirestore, addDoc, collection, query, onSnapshot } from 'firebase/firestore';
	import { onMount } from 'svelte';

	import {
		PUBLIC_APIKEY,
		PUBLIC_AUTHDOMAIN,
		PUBLIC_PROJECTID,
		PUBLIC_STORAGEBUCKET,
		PUBLIC_MESSAGINGSENDERID,
		PUBLIC_APPID
	} from '$env/static/public';

	const firebaseConfig = {
		apiKey: PUBLIC_APIKEY,
		authDomain: PUBLIC_AUTHDOMAIN,
		projectId: PUBLIC_PROJECTID,
		storageBucket: PUBLIC_STORAGEBUCKET,
		messagingSenderId: PUBLIC_MESSAGINGSENDERID,
		appId: PUBLIC_APPID
	};

	const app = initializeApp(firebaseConfig);
	const storage = getStorage();
	const db = getFirestore(app);
	const fileCollectionRef = collection(db, `files`);

	let uploadedFiles: string[] = [];
	let inputEl: HTMLElement;

	const maxAge = 60 * 60 * 24 * 365 * 100; //100 years
	const file_upload = async (file: File): Promise<string | undefined> => {
		try {
			const fileName = file.name;
			const path = `files/${fileName}`;
			const metadata = {
				cacheControl: `public,max-age=${maxAge}`,
				contentType: file.type
			};

			// Note: Upload file
			const storageRef = ref(storage, path);
			await uploadBytes(storageRef, file, metadata);

			const encodedPath = encodeURIComponent(path);

			return encodedPath;
		} catch (e) {
			console.error(e);
			return undefined;
		}
	};

	const handleFileUpload = async (e: Event) => {
		const target = e.target as HTMLInputElement;
		const fileToUpload = target.files?.[0];

		if (!fileToUpload) {
			return;
		}

		const storagePath = await file_upload(fileToUpload);

		if (storagePath) {
			// Note: Add document to firebase firestore
			addDoc(fileCollectionRef, {
				storagePath
			});
		}
	};

	onMount(() => {
		const q = query(fileCollectionRef);

		// Note: Listen for changes in the firestore collection, then unsubsribe
		// when the component unmounts
		return onSnapshot(q, (snapshot) => {
			const latestFiles: string[] = [];
			snapshot.forEach((doc) => {
				const fileName = decodeURIComponent(doc.data().storagePath).split('/')[1];
				latestFiles.push(fileName);
			});
			uploadedFiles = latestFiles;
		});
	});
</script>

<div class="wrapper">
	<input
		bind:this={inputEl}
		on:change={handleFileUpload}
		type="file"
		id="upload"
		style="display:none;"
	/>
	<button id="uploadButton" on:click={() => inputEl.click()}> upload file </button>

	<h2>uploaded files:</h2>
	<ul>
		{#each uploadedFiles as fileName (fileName)}
			<li>
				{fileName}
			</li>
		{/each}
	</ul>

	<p>by <a href="https://xanderjakeq.page/">xanderjakeq</a></p>
</div>

<style>
	.wrapper {
		font-family: sans-serif;
		display: flex;
		flex-direction: column;
		align-items: start;
		justify-content: center;
		max-width: 50rem;
		margin: auto;
		height: 100vh;
	}
	button {
		align-self: center;
		height: 10rem;
		width: 20rem;
		font-size: 3rem;
		font-weight: bold;
		margin: 10px;
		border-radius: 1rem;
		border: none;
		background-color: #a2c95f;
		cursor: pointer;
	}
</style>
