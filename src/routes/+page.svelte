<script lang="ts">
	import { onMount } from 'svelte';
	import { writable, type Writable } from 'svelte/store';
	import { ethers } from 'ethers';
    import { contractsAddresses, contractBytecodes } from '$lib/catalystconfig';

	type Chain = {
		rpc: string;
		factory?: Promise<ethers.TransactionResponse> | Promise<number> | number;
		amplified_mathlib?: Promise<ethers.TransactionResponse> | Promise<number> | number;
		amplified_template?: Promise<ethers.TransactionResponse> | Promise<number> | number;
		volatile_mathlib?: Promise<ethers.TransactionResponse> | Promise<number> | number;
		volatile_template?: Promise<ethers.TransactionResponse> | Promise<number> | number;
	};
	const chains: Writable<Chain[]> = writable([]);

	const emptyChain = {
		rpc: '',
		factory: 1,
		amplified_mathlib: 1,
		amplified_template: 1,
		volatile_mathlib: 1,
		volatile_template: 1
	};

	let privateKey = '';

	function addChain() {
		$chains = [...$chains, { ...emptyChain }];
	}

	function removeChain() {
		const readChains = $chains;
		readChains.pop();
		$chains = readChains;
	}
	// https://base-sepolia-rpc.publicnode.com, https://polygon-testnet.public.blastapi.io
	async function rpcSet(chain: Chain) {
		const rpc = chain.rpc;
		const provider = new ethers.JsonRpcProvider(rpc, undefined);
		try {
			const a = await provider._detectNetwork();
			console.log(a);
		} catch (error) {
			console.log('Error network');
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
                to: '0x4e59b44847b379578588920cA78FbF26c0B4956C',
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
			if (newRpcValue.includes(',')) {
				const newRpcs = newRpcValue.split(',');
				for (let i = 0; i < newRpcs.length; ++i) {
					const newRpc = newRpcs[i];
					const changedChain = { ...emptyChain, rpc: newRpc.replace(' ', '') };
					console.log(changedChain);
					rpcSet(changedChain).then(() => {
						$chains = [
							...$chains.slice(0, index + i),
							changedChain,
							...$chains.slice(index + i + 1)
						];
						updateChainsSize();
					});
				}
			} else {
				const newRpc = newRpcValue;
				const changedChain = { ...emptyChain, rpc: newRpc.replace(' ', '') };
				rpcSet(changedChain).then(() => {
					$chains = [...$chains.slice(0, index), changedChain, ...$chains.slice(index + 1)];
					updateChainsSize();
				});
			}
		};
	}

	function updateChainsSize() {
		console.log('Hello');
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

	onMount(() => {
		addChain();
	});
</script>

<h1 class="font-bold text-lg">Deploy Catalyst</h1>
<p>Catalyst can be deployed permissionlessly.</p>
<input bind:value={privateKey} placeholder="key" />

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
						<button class="deployActivated" class:deployed on:click={call(i, 'factory')}>Deploy</button>
					{/await}
				</td>

				<td>
					{#await chain.amplified_mathlib}
						<button class="deployActivated deployed">...</button>
					{:then deployed}
						<button class="deployActivated" class:deployed on:click={call(i, 'amplified_mathlib')}>Deploy</button>
					{/await}
				</td>
				<td>
					{#await chain.amplified_template}
						<button class="deployActivated deployed">...</button>
					{:then deployed}
						<button class="deployActivated" class:deployed on:click={call(i, 'amplified_template')}>Deploy</button>
					{/await}
				</td>
				<td>
					{#await chain.volatile_mathlib}
						<button class="deployActivated deployed">...</button>
					{:then deployed}
						<button class="deployActivated" class:deployed on:click={call(i, 'volatile_mathlib')}>Deploy</button>
					{/await}
				</td>
				<td>
					{#await chain.volatile_template}
						<button class="deployActivated deployed">...</button>
					{:then deployed}
						<button class="deployActivated" class:deployed on:click={call(i, 'volatile_template')}>Deploy</button>
					{/await}
				</td>
				<td>
					<input
						value={chain.rpc}
						placeholder="rpc-url1"
						class="bg-gray-100 text-center"
						on:input={checkValue(i)}
					/>
				</td>
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
</style>
