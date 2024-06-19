<script lang="ts">
	import { onMount } from 'svelte';
	import { writable, type Writable } from 'svelte/store';
	import { ethers } from 'ethers';
	import {
		contractsAddresses,
		contractBytecodes,
		garpBytescodes,
		sendLostGasTo
	} from '$lib/catalystconfig';

	let messagingProtocol: keyof typeof garpBytescodes = 'Wormhole';

	const factory = '0x4e59b44847b379578588920cA78FbF26c0B4956C';

	type Chain = {
		rpc: string;
		chainId: bigint;
		factory?: Promise<ethers.TransactionResponse> | Promise<number> | number;
		amplified_mathlib?: Promise<ethers.TransactionResponse> | Promise<number> | number;
		amplified_template?: Promise<ethers.TransactionResponse> | Promise<number> | number;
		volatile_mathlib?: Promise<ethers.TransactionResponse> | Promise<number> | number;
		volatile_template?: Promise<ethers.TransactionResponse> | Promise<number> | number;
		messagingProtocolAddress: string;
		expectedGarpAddress: string;
	};
	const chains: Writable<Chain[]> = writable([]);

	const emptyChain = {
		rpc: '',
		chainId: 0n,
		factory: 1,
		amplified_mathlib: 1,
		amplified_template: 1,
		volatile_mathlib: 1,
		volatile_template: 1,
		messagingProtocolAddress: '',
		expectedGarpAddress: ''
	};

	let privateKey = '';
	$: keyAddress = getAddress(privateKey);

	function getAddress(pk: string): string | undefined {
		if (!pk) return;
		try {
			return new ethers.Wallet('0x' + pk.replace('0x', '')).address;
		} catch (error) {
			console.log({ msg: 'getAddress', error });
		}
	}

	function addChain() {
		$chains = [...$chains, { ...emptyChain }];
	}

	function removeChain() {
		const readChains = $chains;
		readChains.pop();
		$chains = readChains;
	}
	async function rpcSet(chain: Chain) {
		const rpc = chain.rpc;
		const provider = new ethers.JsonRpcProvider(rpc, undefined);
		try {
			const a = await provider._detectNetwork();
			chain.chainId = a.chainId;
		} catch (error) {
			console.log({ msg: 'Error network', error });
			return;
		}
		chain.amplified_mathlib = resolveCode(provider.getCode(contractsAddresses.amplified_mathlib));
		chain.amplified_template = resolveCode(provider.getCode(contractsAddresses.amplified_template));
		chain.volatile_mathlib = resolveCode(provider.getCode(contractsAddresses.volatile_mathlib));
		chain.volatile_template = resolveCode(provider.getCode(contractsAddresses.volatile_template));
		chain.factory = resolveCode(provider.getCode(contractsAddresses.factory));

		await Promise.all([
			chain.amplified_mathlib,
			chain.amplified_template,
			chain.volatile_mathlib,
			chain.volatile_template,
			chain.factory
		]);
	}

	async function resolveCode(code: Promise<string | null>): Promise<number> {
		try {
			return ((await code)?.length ?? 4) - 2;
		} catch (error) {
			console.log({ error });
		}
		return 4;
	}

	function call(index: number, contract: keyof typeof contractBytecodes) {
		function call_factory() {
			const rpc = $chains[index].rpc;
			const provider = new ethers.JsonRpcProvider(rpc);
			const signer = new ethers.Wallet(privateKey, provider);

			const transactionData = {
				to: factory,
				data: contractBytecodes[contract]
			};

			const transactionResponse = signer.sendTransaction(transactionData);
			chains.update(($chains) => {
				const chain = $chains[index];
				chain[contract] = transactionResponse;
				transactionResponse.then((submittedTransaction) => {
					submittedTransaction.wait().then(() => {
						chain[contract] = resolveCode(provider.getCode(contractsAddresses[contract]));
					});
				});
				return $chains;
			});
		}
		return call_factory;
	}

	function checkValue(index: number) {
		return function factoryCheckValue(
			event: Event & { currentTarget: EventTarget & HTMLInputElement }
		) {
			const newRpcValue = event.currentTarget.value ?? '';
			let newRpcs: string[];
			if (newRpcValue.includes(',')) {
				newRpcs = newRpcValue.split(',');
			} else {
				newRpcs = [newRpcValue];
			}

			const chainChanges: Promise<unknown>[] = [];
			for (let i = 0; i < newRpcs.length; ++i) {
				const newRpc = newRpcs[i];
				const changedChain = { ...emptyChain, rpc: newRpc.replace(' ', '') };
				$chains = [...$chains.slice(0, index + i), changedChain, ...$chains.slice(index + i + 1)];
			}
			for (let i = 0; i < newRpcs.length; ++i) {
				const newRpc = newRpcs[i];
				const changedChain = { ...emptyChain, rpc: newRpc.replace(' ', '') };
				console.log(changedChain);
				chainChanges.push(
					rpcSet(changedChain).then(() => {
						$chains = [
							...$chains.slice(0, index + i),
							changedChain,
							...$chains.slice(index + i + 1)
						];
					})
				);
			}
			Promise.all(chainChanges).then(() => {
				return updateChainsSize();
			});
		};
	}

	function updateChainsSize() {
		const arraySize = $chains.length;
		console.log($chains[arraySize - 1].rpc);
		if ($chains[arraySize - 1].rpc !== '') {
			return addChain();
		}
		if (arraySize === 1) return;
		if ($chains[arraySize - 2].rpc !== '') {
			return;
		}
		removeChain();
	}

	function computeExpectedGarp() {
		const fullcode = garpBytescodes[messagingProtocol].replace('0x', '');
		const salt = fullcode.slice(0, 64);
		const bytecode = '0x' + fullcode.slice(64);
		chains.update(($chains) => {
			for (let i = 0; i < $chains.length; ++i) {
				$chains[i].expectedGarpAddress =
					'0x' +
					ethers
						.keccak256(
							'0xFF' +
								factory.replace('0x', '') +
								salt +
								ethers
									.keccak256(
										bytecode +
											sendLostGasTo.replace('0x', '').padStart(64, '0') +
											$chains[i].messagingProtocolAddress.replace('0x', '').padStart(64, '0')
									)
									.replace('0x', '')
						)
						.slice(2 + 12 * 2, 2 + 32 * 2);
			}
			return $chains;
		});
	}

	onMount(() => {
		addChain();
	});
</script>

<h1 class="font-bold text-xl">Deploy Catalyst</h1>
<p>Catalyst can be deployed permissionlessly.</p>
<input bind:value={privateKey} placeholder="key" />
{keyAddress}

<h2 class="font-semibold text-lg">Core Catalyst</h2>
<table class="w-full">
	<thead>
		<tr>
			<th>factory</th>
			<th>amplified_mathlib </th>
			<th>amplified_template</th>
			<th>volatile_mathlib</th>
			<th>volatile_template</th>
			<th>rpc</th>
		</tr>
	</thead>
	<tbody>
		{#each $chains as chain, i}
			<tr>
				<td>
					{#await chain.factory}
						<button class="deployActivated deployed">...</button>
					{:then deployed}
						<button class="deployActivated" class:deployed on:click={call(i, 'factory')}
							>Deploy</button
						>
					{/await}
				</td>

				<td>
					{#await chain.amplified_mathlib}
						<button class="deployActivated deployed">...</button>
					{:then deployed}
						<button class="deployActivated" class:deployed on:click={call(i, 'amplified_mathlib')}
							>Deploy</button
						>
					{/await}
				</td>
				<td>
					{#await chain.amplified_template}
						<button class="deployActivated deployed">...</button>
					{:then deployed}
						<button class="deployActivated" class:deployed on:click={call(i, 'amplified_template')}
							>Deploy</button
						>
					{/await}
				</td>
				<td>
					{#await chain.volatile_mathlib}
						<button class="deployActivated deployed">...</button>
					{:then deployed}
						<button class="deployActivated" class:deployed on:click={call(i, 'volatile_mathlib')}
							>Deploy</button
						>
					{/await}
				</td>
				<td>
					{#await chain.volatile_template}
						<button class="deployActivated deployed">...</button>
					{:then deployed}
						<button class="deployActivated" class:deployed on:click={call(i, 'volatile_template')}
							>Deploy</button
						>
					{/await}
				</td>
				<td>
					<input
						bind:value={chain.rpc}
						placeholder="rpc-url"
						class="bg-gray-100 text-center"
						on:input={checkValue(i)}
					/>
				</td>
			</tr>
		{/each}
	</tbody>
</table>

<h2 class="font-semibold text-lg">Cross-chain interface</h2>

<div class="flex flex-row">
	<button
		class="messageprotocol"
		class:selected={messagingProtocol === 'Wormhole'}
		on:click={() => {
			messagingProtocol = 'Wormhole';
			computeExpectedGarp();
		}}
	>
		Wormhole
	</button>
	<button
		class="messageprotocol"
		class:selected={messagingProtocol === 'LayerZero'}
		on:click={() => {
			messagingProtocol = 'LayerZero';
			computeExpectedGarp();
		}}
	>
		LayerZero
	</button>
</div>

<table class="w-full">
	<thead>
		<tr>
			<th>ChainId</th>
			{#each $chains.slice(0, $chains.length - 1) as chainx, x}
				<th>{chainx.chainId}</th>
			{/each}
		</tr>
		<tr>
			<th>{messagingProtocol}</th>
			{#each $chains.slice(0, $chains.length - 1) as chainx, x}
				<th
					><input
						class="bg-gray-100"
						bind:value={chainx.messagingProtocolAddress}
						on:input={computeExpectedGarp}
					/></th
				>
			{/each}
		</tr>
		<tr>
			<th>GARP</th>
			{#each $chains.slice(0, $chains.length - 1) as chainx, x}
				<th class="text-center">
					<div class="w-40 monkey-ellipsis">
						{chainx.expectedGarpAddress}
					</div>
				</th>
			{/each}
		</tr>
		<tr>
			<th>CCI</th>
			{#each $chains.slice(0, $chains.length - 1) as chainx, x}
				<th class="text-center">
					<div class="w-40 monkey-ellipsis">
						{chainx.expectedGarpAddress}
					</div>
				</th>
			{/each}
		</tr>
	</thead>
	<tbody>
		<tr>
			<th>Deploy</th>
			{#each $chains.slice(0, $chains.length - 1) as chainx, x}
				<th class="text-center">
					<button class="deployActivated">Deploy</button>
				</th>
			{/each}
		</tr>
		{#each $chains.slice(0, $chains.length - 1) as chainx, x}
			<tr>
				<td>
					{chainx.chainId}
				</td>
				{#each $chains.slice(0, $chains.length - 1) as chainy, y}
					<td> <button>Connect:{chainx.chainId},{chainy.chainId}</button> </td>
				{/each}
			</tr>
		{/each}
	</tbody>
</table>

<style lang="postcss">
	td {
		@apply text-center;
	}

	button.deployActivated {
		@apply px-2 rounded-sm;
	}
	button.deployActivated.deployed {
		@apply text-gray-300 bg-gray-100;
	}
	button.deployActivated:not(.deployed) {
		@apply text-black bg-blue-200;
	}
	button.messageprotocol {
		@apply bg-blue-100 px-2 py-1 hover:bg-blue-200;
	}
	button.messageprotocol.selected {
		@apply bg-blue-300;
	}
</style>
