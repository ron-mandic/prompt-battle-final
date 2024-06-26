<script lang="ts">
	import { onMount } from 'svelte';
	import { goto } from '$app/navigation';
	import { page } from '$app/stores';
	import { TEXT_H1, UNKNOWN, URL_SERVER } from '$lib/ts/constants';
	import { Socket, io } from 'socket.io-client';
	// TODO: Uncomment this line when using PUBLIC_ID
	// import { PUBLIC_ID } from '$env/static/public';

	const socket: Socket = io(URL_SERVER, {
		reconnection: true
	});

	let refTerminal: HTMLDivElement;
	let refInput: HTMLInputElement;

	let hasEntered = false;
	let isStarting = false;

	let mode: string;
	let playerName: string;
	let playerUUID: string;
	let playerNumber = UNKNOWN;

	onMount(() => {
		sessionStorage.clear();

		socket.on('connect', () => {
			socket.emit('c:initClient' /*, PUBLIC_ID*/);
		});

		socket.on('s:initClient', ({ id, uuid }) => {
			playerUUID = uuid;
			playerNumber = id;

			$page.url.searchParams.set('id', id);
			$page.url.searchParams.set('uuid', uuid);
			$page.url.searchParams.delete('mode');

			goto(`?${$page.url.searchParams.toString()}`);
		});

		socket.on('s:start', () => {
			isStarting = true;
		});

		socket.on('s:setMode', (data) => {
			mode = data;
			$page.url.searchParams.set('mode', data);
			goto(`?${$page.url.searchParams.toString()}`);
		});

		refInput.focus();

		return () => {
			socket.disconnect();
		};
	});

	function handleBlur() {
		refInput.focus();
		return false;
	}

	function handleInput() {
		refTerminal.style.setProperty('--offset', `${refInput.value.length * 30}px`);
		playerName = refInput.value;
		socket.emit('c:setPlayerNames', {
			id: playerNumber,
			name: playerName
		});
	}

	function handleKeyDown(e: KeyboardEvent) {
		if (e.key === 'Enter') {
			if (refInput.value === '') return;

			hasEntered = true;
			socket.emit('c:setPlayerReadiness', playerNumber);

			// Alternative solution to undefined in scribble and results-views
			sessionStorage.setItem(playerNumber, playerName);
		}
	}

	$: {
		if (isStarting && mode) {
			setTimeout(() => {
				goto(`prompt?id=${playerNumber}&uuid=${playerUUID}&mode=${mode}`);
			}, 0); // 3000
		}
	}
</script>

<svelte:head>
	<title>/</title>
</svelte:head>

<div class="w-full max-w-[1419px] m-auto">
	<h1 class="uppercase text-center w-full">{@html TEXT_H1}</h1>
	<section id="terminal" class="p-2">
		<p>/Player {playerNumber} &gt; ./Prompt_Battle</p>
		<p>Prompt_Battle loading...</p>
		<p>Loading complete!</p>
		<p class="mt-16">Enter your name!</p>
		<div
			id="terminal-input"
			class="relative mt-16"
			class:changed={isStarting}
			bind:this={refTerminal}
		>
			<label>
				<span>/Player {playerNumber} &gt;</span>
				<input
					type="text"
					name="player"
					maxlength="20"
					autocomplete="off"
					autocorrect="off"
					on:blur={handleBlur}
					on:input={handleInput}
					on:keydown={handleKeyDown}
					bind:this={refInput}
				/>
			</label>
		</div>
		{#if hasEntered && isStarting && mode}
			<p id="terminal-indicator" class="w-full blink ellipsis">Starting</p>
		{:else if hasEntered}
			<p id="terminal-indicator" class="w-full ellipsis">Waiting</p>
		{/if}
	</section>
</div>

<style lang="scss">
	h1 {
		font-size: clamp(4rem, 12vw, 170px);
	}

	#terminal {
		border: 2px solid #6eebea;
		background-color: #1c1f22;
		animation: slide-in-up 1s ease;
		will-change: transform;

		p,
		label {
			color: #6eebea;
			font-family: 'JetBrains Mono';
			font-size: 50px;
			font-style: normal;
			font-weight: 700;
			line-height: normal;
		}
	}

	#terminal-input {
		input {
			position: relative;
			color: #fff;
			background-color: transparent;
			font-family: 'JetBrains Mono';
			font-size: 50px;
			font-style: normal;
			font-weight: 700;
			line-height: normal;

			caret-color: transparent;

			&::after {
				content: '|';
				animation: blink 1s infinite;
			}

			&:focus {
				outline: none;
			}
		}

		&:not(.changed)::after {
			content: '';
			position: absolute;
			top: 0;
			left: calc(22.5rem + var(--offset, 0px));
			display: inline-block;
			background-color: #fff;
			vertical-align: top;
			width: 27.5px;
			height: 66px;
			-webkit-animation: blink 1s step-end infinite;
			animation: blink 1s step-end infinite;
			border: none;
		}
	}

	#terminal-indicator {
		&.blink {
			animation: blink 1s infinite;
		}
		&.ellipsis::after {
			content: '';
			animation: ellipsis 3s infinite 1s;
		}
	}

	@keyframes ellipsis {
		0% {
			content: '';
		}
		33% {
			content: '.';
		}
		66% {
			content: '..';
		}
		100% {
			content: '...';
		}
	}

	@keyframes slide-in-up {
		0% {
			transform: translateY(-5rem);
			opacity: 0;
		}
		100% {
			transform: translateY(0);
			opacity: 1;
		}
	}

	@keyframes blink {
		0% {
			opacity: 1;
		}
		50% {
			opacity: 0;
		}
		100% {
			opacity: 1;
		}
	}
</style>
