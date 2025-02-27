<script>
	import { supabase, fetchMessages, insertMessage, getBannedUserIPAddresses, uploadImage} from '$lib/supabaseClient.js'
	import { changeTheme } from '$lib/main.js'
	import messageStore from '$lib/stores/messageStore';
	import timestamp from 'unix-timestamp';
	import { Spinner } from 'flowbite-svelte'
	import { onMount } from 'svelte';
	import Settings from '$lib/components/settings.svelte'

	import Sidebar from '$lib/components/sidebar.svelte';
	import JoinBoard from '$lib/components/joinboard.svelte';
	import DeleteBoard from '$lib/components/deleteboard.svelte';
	import Banned from '$lib/components/banned.svelte';
	import About from '$lib/components/about.svelte';

	import CreateBoard from '$lib/components/createboard.svelte';
	import MessageWindow from '$lib/components/message-window.svelte';
	import MessageInput from '$lib/components/message-input.svelte';

	import User from "$lib/components/user.svelte"
    import { Toaster, toast } from 'svelte-sonner';

	import { persisted } from 'svelte-persisted-store';
	import { get } from 'svelte/store'

	import { MenuIcon, XIcon } from 'svelte-feather-icons';
    import { Button } from "$lib/components/ui/button";


	const currentDate = new Date();

	let oldUI = false;
	let showSidebar = false;

	let themeColor = "auto";
	let themesCSS = ""

	let fontCSS = "font-apple"
	let isPrivate = false

	let isReplying = false;
	let replyTo = null;
	let isEditing = false;

	$: newButtonClass = `${oldUI ? "" : "rounded-full border-gray-800 border-2" } ${themesCSS}`

	let message = "";

	let currentBoard = "general";
	let boards = [];

	let username = '';
	let showUsername = false;
	let persistedUsernameStore = persisted('username', { key: 'username' });
	let persistedBoardsStore = persisted('boards', {key: 'boards' });

	onMount(() => {

    	try {
			const storedUsername = get(persistedUsernameStore);
			const storedBoards = get(persistedBoardsStore);

			// Handle username
			if (typeof storedUsername === 'string') {
				username = storedUsername;
			} else if (storedUsername && storedUsername.username) {
				username = storedUsername.username;
			} else {
				username = '';
			}

			// Handle boards
			if (Array.isArray(storedBoards)) {
				boards = storedBoards;
			} else if (storedBoards && Array.isArray(storedBoards.boards) && storedBoards.boards.length > 0) {
				boards = storedBoards.boards;
			} else {
				boards = [		
				"general",
				"programming",
				"technology",
				"photography",
				"art",
				"music",
				"gaming",
				'school'
				];
			}

			// Determine if username input form should be shown
			showUsername = !username;
		} catch (error) {
			console.error('Error retrieving data:', error);
			username = '';
			showUsername = true;
			boards = [		
				"general",
				"programming",
				"technology",
				"photography",
				"art",
				"music",
				"gaming",
				'school'
			];
		}
	})

	let showLogin = false;
	let showSettings = false;
	let showDate = false;
	let showHighlight = true;

	let createBoard = false;
	let joinBoard = false;
	let isDeleting = false;
	let isCreator = false;
	let goAbout = false;
	let showImages = true;

	let messages = [{}];

	let emojiPickerOpen = true;

	let iP;
	let banned_IPS = [];

	let isMobile = false;

	function detectMobile() {
		const userAgent = navigator.userAgent || navigator.vendor || window.opera;
		isMobile = /android|iPad|iPhone|iPod/i.test(userAgent);
	}
	
	getBannedUserIPAddresses().then(ipAddresses => {
		banned_IPS = ipAddresses;
	});

	let isBanned = false;
		


  	onMount(async () => {
		detectMobile();
		await getBannedUserIPAddresses();
		try {
			const response = await fetch('https://api.ipify.org?format=json');
			const data = await response.json();
			
			iP = data.ip;

			if (banned_IPS.includes(iP)) {

				isBanned = true;
			}
		} catch (error) {
			console.error('Error fetching IP address:', error);
		}
	});

	boards = [
		"general",
		"programming",
		"technology",
		"photography",
		"art",
		"music",
		"gaming",
		'school'
	];	

	let promise = new Promise(() => {});


	onMount(() => {
		themesCSS = changeTheme(themeColor)
		promise = (async () => {
			const { data, error } = await supabase
				.from("messages")
				.select()
			messageStore.set(data.reverse());
		})()
	})

	
	supabase.channel('custom-all-channel')
	.on(
		'postgres_changes',
		{ event: '*', schema: 'public', table: 'messages' },
		() => {
			fetchMessages()
			.then((data) => {
				messageStore.set(data.reverse());
			})
		}
	)
	.subscribe()

	messageStore.subscribe((data) => {
		messages = data.sort((a,b) => a.sent_at - b.sent_at).reverse();	
	})
	

	const sendMessage = async (message, file) => {
		if (!isBanned) {
			// Add logic to handle banned users if needed
		}

		if (message === "" && file == null) {
			toast.error("message content cannot be empty!");
			return;
		}

		if (messages[0]) {
			if (message === messages[0].content && messages[0].sender === username) {
				toast.error("you can't send the same message twice :)");
				return;
			}
			if (timestamp.now() - messages[0].sent_at < 1 && username === messages[0].sender) {
				toast.error("you can't send messages that fast :)");
				return;
			}
		}

		let fileUrl = null;
		let fileType = null;

		if (file) {
			fileUrl = await uploadImage(file); 

			if (!fileUrl) {
				toast.error('Error uploading file. Please try again.');
				return;
			} else {
				toast.success("Successfully uploaded file.");
			}

			fileType = file.type; 
		}

		let newMessage = {
			content: message,
			sent_at: timestamp.now(),
			date: currentDate,
			sender: username,
			board: currentBoard,
			sender_iP: iP,
			file_url: fileUrl,
			file_type: fileType,
			played_sound: false,
			replyTo: replyTo
		};

		console.log('New message object:', newMessage);

		messageStore.update(messages => {
			return [newMessage, ...messages];
		});

		replyTo = null;
		isReplying = false;

		await insertMessage(newMessage);
	};
</script>

<svelte:window on:keypress={(e) => {
	if (e.key == "Escape") {
		showLogin = false;
		showSettings = false;
		createBoard = false;
	}
}} />

{#if oldUI}
	<link rel="stylesheet" href="https://unpkg.com/98.css" />
{/if}

<Toaster position="bottom-right" richColors />

<main class="{fontCSS} h-screen w-screen {oldUI ? "bg-gray-300" : "bg-slate-50"} {themesCSS}">
	<!-- account modal -->
	{#if showUsername}
		<button class="z-10 fixed flex h-screen w-screen items-center justify-center {themesCSS}"
		on:click={() => {
			 //cant close until username is fixed
		}}>
		<div on:click|stopPropagation>
			<User 
				{isMobile} 
				bind:showUsername 
				bind:persistedUsernameStore 
				bind:username 
			/>
		</div>
		
	</button>
	{/if}
	{#if goAbout}
		<button class="z-10 fixed flex h-screen w-screen items-center justify-center {themesCSS}"
		on:click={() => {
			goAbout = false
		}}>
		<div on:click|stopPropagation>
			<About {isMobile} />
		</div>
	</button>

	{/if}
	{#if isBanned}
		<button class="z-10 fixed flex h-screen w-screen items-center justify-center {themesCSS}"
		on:click={() => {
			showLogin = false
		}}>
		<div on:click|stopPropagation>
			<Banned {iP}/>
		</div>
	</button>
	{/if}
	{#if showSettings}
		<button class="z-10 fixed flex h-screen w-screen items-center justify-center {themesCSS}"
		on:click={() => {
			showSettings = false
		}}>
		<div on:click|stopPropagation>
			<Settings {isMobile} bind:themeColor={themeColor} bind:themesCSS={themesCSS} bind:fontCSS={fontCSS} bind:isPrivate={isPrivate} bind:oldUI={oldUI} bind:showDate={showDate} bind:showHighlight={showHighlight} bind:showImages={showImages}/>
		</div>
	</button>	
	{/if}
	<!-- create board modal-->
	{#if createBoard}
		<button class="z-10 fixed flex h-screen w-screen items-center justify-center {themesCSS}"
		on:click={() => {
			createBoard = false
		}}>
		<div on:click|stopPropagation>
			<CreateBoard {isMobile} {themesCSS} {username} bind:createBoard={createBoard} bind:boards={boards}/>
		</div>
	</button>
	{/if}
	<!-- join board modal -->
	{#if joinBoard}
		<button class="z-10 fixed flex h-screen w-screen items-center justify-center {themesCSS}"
		on:click={() => {
			joinBoard = false
		}}>
		<div on:click|stopPropagation>
			<JoinBoard {isMobile} {themesCSS} bind:joinBoard={joinBoard} bind:boards={boards} />
		</div>
	</button>
	{/if}
	{#if isDeleting}
		<button class="z-10 fixed flex h-screen w-screen items-center justify-center {themesCSS}"
		on:click={() => {
			isDeleting = false
		}}>
		<div on:click|stopPropagation>
			<DeleteBoard {isMobile} {themesCSS} bind:isCreator={isCreator} bind:isDeleting={isDeleting} bind:boards={boards} bind:board={currentBoard}/>
		</div>
	</button>
	{/if}
	{#if !showSidebar}
	<button on:click={() => showSidebar = true} class="fixed top-4 right-4 z-20 p-2 bg-white text-white rounded-full lg:hidden">
		<Button class="w-8 h-8 p-0" variant="ghost">
			<MenuIcon size="20" class="stroke-gray-400"/> 
		</Button>
	</button>	 
	{/if}
	{#if (!isMobile && !showSidebar) || isMobile && !showSidebar}
	<div class="fixed space-x-10 flex flex-row {themesCSS}" class:blur-md={showLogin || showSettings}> 
		{#if !isMobile}
			<Sidebar bind:isCreator={isCreator} bind:isDeleting={isDeleting} bind:boards={boards} bind:createBoard={createBoard} bind:joinBoard={joinBoard} bind:currentBoard={currentBoard} bind:oldUI={oldUI} bind:goAbout={goAbout} bind:showSettings={showSettings} {isPrivate} {fontCSS} {themeColor} {username} {themesCSS} {newButtonClass}/>
		{/if}
		{#await promise}
			<div class="flex w-screen h-screen justify-center items-center">
				<Spinner color="blue" />
			</div>
		{:then}
			<div class="px-4 justify-start flex">
				<div class="{isMobile ? 'lg:w-[100vw] w-[100vw]' : 'lg:w-[75vw] w-[60vw]'} h-[80vh] justify-center p-4">
					<MessageWindow {isMobile} {username} {oldUI} {showDate} {showImages} {messages} {currentBoard} bind:isEditing={isEditing} bind:isReplying={isReplying} bind:replyTo={replyTo}/>
					<MessageInput {isMobile} {isEditing} {message} {messages} {username} {themesCSS} bind:emojiPickerOpen={emojiPickerOpen} bind:isReplying={isReplying} bind:replyTo={replyTo} {sendMessage}/>
				</div>
			</div>
		{/await}
	</div>
	{:else}
		{#if isMobile && showSidebar}
			<div class="fixed inset-0 z-30 flex items-center justify-center bg-opacity-75">
				<Sidebar bind:isCreator={isCreator} bind:isDeleting={isDeleting} bind:boards={boards} bind:createBoard={createBoard} bind:joinBoard={joinBoard} bind:currentBoard={currentBoard} bind:oldUI={oldUI} bind:goAbout={goAbout} bind:showSettings={showSettings} {isPrivate} {fontCSS} {themeColor} {username} {themesCSS} {newButtonClass}/>
				<button on:click={() => showSidebar = false} class="absolute top-4 right-4 z-40 p-2 bg-white text-white rounded-full">
					<Button class="w-8 h-8 p-0" variant="ghost">
						<XIcon size="20" class="stroke-gray-400"/> 
					</Button>
				</button>
			</div>
		{/if}
	{/if}
	
</main>
