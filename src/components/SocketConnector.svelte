<script lang="ts">
	import { mdiConnection } from '@mdi/js';
	import { tap, throttleTime } from 'rxjs';
	import { onMount } from 'svelte';
	import { SocketConnection } from '../socket';
	import { isPaused$, openDialog$, socketState$, websocketUrl$ } from '../stores/stores';
	import { reduceToEmptyString } from '../util';
	import Icon from './Icon.svelte';

	let socketConnection: SocketConnection | undefined;
	let intitialAttemptDone = false;
	let wasConnected = false;
	let closeRequested = false;

	const urlHandler$ = websocketUrl$.pipe(
		throttleTime(1000),
		tap((websocketUrl) => {
			if (wasConnected && !closeRequested && websocketUrl !== socketConnection?.getCurrentUrl()) {
				toggleSocket();
			}
		}),
		reduceToEmptyString()
	);

	$: switch ($socketState$) {
		case 0:
			wasConnected = false;
			closeRequested = false;
			break;
		case 1:
			intitialAttemptDone = true;
			wasConnected = true;
			break;
		case 3:
			if (!closeRequested && intitialAttemptDone) {
				$openDialog$ = {
					type: 'error',
					message: wasConnected ? `Lost Connection to Websocket` : 'Unable to connect to Websocket',
					showCancel: false,
				};

				if (wasConnected) {
					$isPaused$ = true;
				}

				// NOTE: Can't implement automatic (re-)connection due to https://stackoverflow.com/a/59548772
			}

			intitialAttemptDone = true;
			wasConnected = false;
			break;

		default:
			break;
	}

	$: connectedWithLabel = updateConnectedWithLabel(wasConnected);

	onMount(() => {
		toggleSocket();

		return () => {
			closeRequested = true;
			socketConnection?.disconnect();
		};
	});

	function updateConnectedWithLabel(hasConnection: boolean) {
		return hasConnection ? `Connected with ${$websocketUrl$}` : 'Not Connected';
	}

	async function toggleSocket() {
		if ($socketState$ === 1 && socketConnection) {
			closeRequested = true;
			socketConnection.disconnect();
		} else {
			socketConnection = socketConnection || new SocketConnection();
			socketConnection.connect();
		}
	}
</script>

{$urlHandler$ ?? ''}
{#if $socketState$ !== 0}
	<div
		class="hover:text-primary"
		class:text-red-500={$socketState$ !== -1 && $socketState$ !== 1}
		class:text-green-700={$socketState$ === 1}
		title={connectedWithLabel}
	>
		<Icon path={mdiConnection} class="cursor-pointer mx-2" on:click={toggleSocket} />
	</div>
{:else}
	<span class="animate-ping relative inline-flex rounded-full h-3 w-3 mx-3 bg-primary" />
{/if}
