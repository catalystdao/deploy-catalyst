<script lang="ts">
	import { onMount } from 'svelte';
	import { writable, type Writable } from 'svelte/store';
	import { ethers } from 'ethers';
	import {
		contractsAddresses,
		contractBytecodes,
		garpBytescodes,
		cciBytescodes,
		sendLostGasTo,
		type MessagingVerions,
		type InterfaceTypes,
		cciABI
	} from '$lib/catalystconfig';
	import { chains as chainChannels } from '@catalabs/catalyst-channel-lists';

	let messagingProtocol: keyof typeof garpBytescodes = 'Wormhole';

	const factory = '0x4e59b44847b379578588920cA78FbF26c0B4956C' as const;

	type interfaces = {
		[type in InterfaceTypes]: {
			[version in MessagingVerions]: Promise<number> | number;
		};
	};

	type Chain = {
		rpc: string;
		chainId: bigint;
		factory?: Promise<number> | number;
		amplified_mathlib?: Promise<number> | number;
		amplified_template?: Promise<number> | number;
		volatile_mathlib?: Promise<number> | number;
		volatile_template?: Promise<number> | number;
		provider?: ethers.JsonRpcProvider;
		messagingProtocolAddress: string;
		messagingProtocolDeployed?: Promise<number> | number;
		interfaces: interfaces;
		expectedGarpAddress: string;
		expectedCCIAddress: string;
	};
	const chains: Writable<Chain[]> = writable([]);

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
		$chains = [
			...$chains,
			{
				rpc: '',
				chainId: 0n,
				factory: 1,
				amplified_mathlib: 1,
				amplified_template: 1,
				volatile_mathlib: 1,
				volatile_template: 1,
				messagingProtocolAddress: '',
				interfaces: {
					cci: {},
					garp: {},
					protocol: {}
				} as interfaces,
				expectedGarpAddress: '',
				expectedCCIAddress: ''
			}
		];
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
		chain.provider = provider;
		chain.amplified_mathlib = getCode(provider, contractsAddresses.amplified_mathlib);
		chain.amplified_template = getCode(provider, contractsAddresses.amplified_template);
		chain.volatile_mathlib = getCode(provider, contractsAddresses.volatile_mathlib);
		chain.volatile_template = getCode(provider, contractsAddresses.volatile_template);
		chain.factory = getCode(provider, contractsAddresses.factory);

		await Promise.all([
			chain.amplified_mathlib,
			chain.amplified_template,
			chain.volatile_mathlib,
			chain.volatile_template,
			chain.factory
		]);
	}

	async function getCode(provider: ethers.JsonRpcProvider, address: string): Promise<number> {
		return resolveCode(provider.getCode(address));
	}

	async function resolveCode(code: Promise<string | null>): Promise<number> {
		try {
			return ((await code)?.length ?? 4) - 2;
		} catch (error) {
			console.log({ error });
		}
		return 4;
	}

	function deployCore(index: number, contract: keyof typeof contractBytecodes) {
		function deploy_factory() {
			const rpc = $chains[index].rpc;
			const provider = new ethers.JsonRpcProvider(rpc);
			const signer = new ethers.Wallet(privateKey, provider);

			const transactionData = {
				to: factory,
				data: contractBytecodes[contract]
			};

			const transactionResponse = signer.sendTransaction(transactionData);
			console.log({ transactionResponse });
			chains.update(($chains) => {
				const chain = $chains[index];
				chain[contract] = transactionResponse.then(async (submittedTransaction) => {
					return submittedTransaction.wait().then(() => {
						return getCode(provider, contractsAddresses[contract]);
					});
				});
				return $chains;
			});
		}
		return deploy_factory;
	}

	function deployInterfaces(
		index: number,
		interfaceType: InterfaceTypes,
		version: keyof typeof garpBytescodes
	) {
		function deploy_factory() {
			const chain = $chains[index];
			const rpc = chain.rpc;
			const provider = new ethers.JsonRpcProvider(rpc);
			const signer = new ethers.Wallet(privateKey, provider);

			const transactionData = {
				to: factory,
				data:
					interfaceType === 'cci'
						? cciBytescodes[version] +
							chain.expectedGarpAddress.replace('0x', '').padStart(64, '0') +
							(keyAddress ?? '').replace('0x', '').padStart(64, '0')
						: interfaceType === 'garp'
							? garpBytescodes[version] +
								sendLostGasTo.replace('0x', '').padStart(64, '0') +
								chain.messagingProtocolAddress.replace('0x', '').padStart(64, '0')
							: undefined
			};
			if (transactionData.data === undefined)
				throw Error(`interface type ${interfaceType} not found`);

			const transactionResponse = signer.sendTransaction(transactionData);
			console.log(transactionResponse);
			chains.update(($chains) => {
				const chain = $chains[index];
				chain.interfaces[interfaceType][version] = transactionResponse.then(
					async (submittedTransaction) => {
						return submittedTransaction.wait().then(() => {
							return getCode(
								provider,
								interfaceType === 'cci' ? chain.expectedCCIAddress : chain.expectedGarpAddress
							);
						});
					}
				);
				return $chains;
			});
		}
		return deploy_factory;
	}

	function connectInterfaces(fromChain: Chain, toChain: Chain) {
		async function connectInterfaces_factory() {
			const provider = fromChain.provider;
			const signer = new ethers.Wallet(privateKey, provider);

			const contractToCall = fromChain.expectedCCIAddress;

			const cciContract = new ethers.Contract(contractToCall, cciABI, signer);

			const destinationIdentifier: string =
				chainChannels[messagingProtocol][fromChain.chainId.toString()][toChain.chainId.toString()];
			const remoteCCI: string =
				'0x14' + toChain.expectedCCIAddress.replace('0x', '').padStart(64 * 2, '0');
			const remoteGARP: string =
				'0x' + toChain.expectedGarpAddress.replace('0x', '').padStart(64, '0');
			console.log({
				destinationIdentifier,
				cci: remoteCCI,
				garp: remoteGARP
			});
			if (destinationIdentifier.length != 64 + 2) return;
			if ((await toChain.interfaces.cci[messagingProtocol]) <= 4) return;
			if ((await toChain.interfaces.garp[messagingProtocol]) <= 4) return;

			console.log(cciContract.connectNewChain(destinationIdentifier, remoteCCI, remoteGARP));
		}
		return connectInterfaces_factory;
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
				const changedChain = {
					...{
						rpc: '',
						chainId: 0n,
						factory: 1,
						amplified_mathlib: 1,
						amplified_template: 1,
						volatile_mathlib: 1,
						volatile_template: 1,
						messagingProtocolAddress: '',
						interfaces: {
							cci: {},
							garp: {},
							protocol: {}
						} as interfaces,
						expectedGarpAddress: '',
						expectedCCIAddress: ''
					},
					rpc: newRpc.replace(' ', '')
				};
				$chains = [...$chains.slice(0, index + i), changedChain, ...$chains.slice(index + i + 1)];
			}
			for (let i = 0; i < newRpcs.length; ++i) {
				const newRpc = newRpcs[i];
				const changedChain = {
					...{
						rpc: '',
						chainId: 0n,
						factory: 1,
						amplified_mathlib: 1,
						amplified_template: 1,
						volatile_mathlib: 1,
						volatile_template: 1,
						messagingProtocolAddress: '',
						interfaces: {
							cci: {},
							garp: {},
							protocol: {}
						} as interfaces,
						expectedGarpAddress: '',
						expectedCCIAddress: ''
					},
					rpc: newRpc.replace(' ', '')
				};
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
		const garpFullcode = garpBytescodes[messagingProtocol].replace('0x', '');
		const garpSalt = garpFullcode.slice(0, 64);
		const garpBytecode = '0x' + garpFullcode.slice(64);
		const cciFullcode = cciBytescodes[messagingProtocol].replace('0x', '');
		const cciSalt = cciFullcode.slice(0, 64);
		const cciBytecode = '0x' + cciFullcode.slice(64);
		chains.update(($chains) => {
			for (let i = 0; i < $chains.length; ++i) {
				const chain = $chains[i];
				chain.expectedGarpAddress =
					'0x' +
					ethers
						.keccak256(
							'0xFF' +
								factory.replace('0x', '') +
								garpSalt +
								ethers
									.keccak256(
										garpBytecode +
											sendLostGasTo.replace('0x', '').padStart(64, '0') +
											$chains[i].messagingProtocolAddress.replace('0x', '').padStart(64, '0')
									)
									.replace('0x', '')
						)
						.slice(2 + 12 * 2, 2 + 32 * 2);
				chain.expectedCCIAddress =
					'0x' +
					ethers
						.keccak256(
							'0xFF' +
								factory.replace('0x', '') +
								cciSalt +
								ethers
									.keccak256(
										cciBytecode +
											chain.expectedGarpAddress.replace('0x', '').padStart(64, '0') +
											(keyAddress ?? '').replace('0x', '').padStart(64, '0')
									)
									.replace('0x', '')
						)
						.slice(2 + 12 * 2, 2 + 32 * 2);
				if (chain.provider) {
					chain.interfaces['protocol'][messagingProtocol] = getCode(
						chain.provider,
						chain.messagingProtocolAddress
					);
					chain.interfaces['garp'][messagingProtocol] = getCode(
						chain.provider,
						chain.expectedGarpAddress
					);
					chain.interfaces['cci'][messagingProtocol] = getCode(
						chain.provider,
						chain.expectedCCIAddress
					);
				}
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
<input bind:value={privateKey} placeholder="key" on:input={computeExpectedGarp} />
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
						<button class="deployActivated" class:deployed on:click={deployCore(i, 'factory')}
							>Deploy</button
						>
					{/await}
				</td>

				<td>
					{#await chain.amplified_mathlib}
						<button class="deployActivated deployed">...</button>
					{:then deployed}
						<button
							class="deployActivated"
							class:deployed
							on:click={deployCore(i, 'amplified_mathlib')}>Deploy</button
						>
					{/await}
				</td>
				<td>
					{#await chain.amplified_template}
						<button class="deployActivated deployed">...</button>
					{:then deployed}
						<button
							class="deployActivated"
							class:deployed
							on:click={deployCore(i, 'amplified_template')}>Deploy</button
						>
					{/await}
				</td>
				<td>
					{#await chain.volatile_mathlib}
						<button class="deployActivated deployed">...</button>
					{:then deployed}
						<button
							class="deployActivated"
							class:deployed
							on:click={deployCore(i, 'volatile_mathlib')}>Deploy</button
						>
					{/await}
				</td>
				<td>
					{#await chain.volatile_template}
						<button class="deployActivated deployed">...</button>
					{:then deployed}
						<button
							class="deployActivated"
							class:deployed
							on:click={deployCore(i, 'volatile_template')}>Deploy</button
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
			{#each $chains.slice(0, $chains.length - 1) as chainx}
				<th>{chainx.chainId}</th>
			{/each}
		</tr>
		<tr>
			<th>{messagingProtocol}</th>
			{#each $chains.slice(0, $chains.length - 1) as chainx}
				<th>
					<div class="flex flex-row items-center w-40">
						<input
							class="bg-gray-100 h-5"
							bind:value={chainx.messagingProtocolAddress}
							on:input={computeExpectedGarp}
						/>
						{#await chainx.interfaces.protocol[messagingProtocol]}
							<div class="rounded-r-full w-4 h-5 bg-gray-100"></div>
						{:then deployed}
							{#if deployed == undefined}
								<div class="rounded-r-full w-4 h-5 bg-gray-100"></div>
							{:else}
								<div
									class="rounded-r-full w-4 h-5"
									class:bg-green-100={deployed > 4}
									class:bg-red-100={deployed <= 4}
								></div>
							{/if}
						{/await}
					</div>
				</th>
			{/each}
		</tr>
		<tr>
			<th>GARP</th>
			{#each $chains.slice(0, $chains.length - 1) as chainx, x}
				<th class="text-center">
					{#await chainx.interfaces.garp[messagingProtocol]}
						<button class="w-40 monkey-ellipsis deployActivated deployed">...</button>
					{:then deployed}
						<button
							class="w-40 monkey-ellipsis deployActivated"
							class:deployed
							on:click={deployInterfaces(x, 'garp', messagingProtocol)}
							>{chainx.expectedGarpAddress}</button
						>
					{/await}
				</th>
			{/each}
		</tr>
		<tr>
			<th>CCI</th>
			{#each $chains.slice(0, $chains.length - 1) as chainx, x}
				<th class="text-center">
					{#await chainx.interfaces.cci[messagingProtocol]}
						<button class="w-40 monkey-ellipsis deployActivated deployed">...</button>
					{:then deployed}
						<button
							class="w-40 monkey-ellipsis deployActivated"
							class:deployed
							on:click={deployInterfaces(x, 'cci', messagingProtocol)}
							>{chainx.expectedCCIAddress}</button
						>
					{/await}
				</th>
			{/each}
		</tr>
	</thead>
	<tbody>
		{#each $chains.slice(0, $chains.length - 1) as chainy}
			<tr>
				<td>
					{chainy.chainId}
				</td>
				{#each $chains.slice(0, $chains.length - 1) as chainx}
					<td>
						<button on:click={connectInterfaces(chainx, chainy)}
							>Connect:{chainx.chainId},{chainy.chainId}</button
						>
					</td>
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
