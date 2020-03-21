<script>
	import { onMount } from 'svelte'

	//FORM Binds 
	let methodName = 'transfer';
	let networkType = 'testnet'
	let stampLimit = 50000;
	let argName, argType, argValue
	let approveNetwork = 'testnet'
	let approveContract = 'currency'

	//DOM Binds
	let selectedArgType, selectedVk, argNameInput

	$: lamdenInfo = {wallets: []};
	let txStatus = '';

	$: kwargs = {
		to: {type: 'address', value:'57be23d7af186ef946408dbbfb7407b5df4faac4abb856a6e0daf186080fc69d'},
		amount: {type: 'fixedPoint', value:2}
	}

	let ArgTypes = ['text', 'address', 'data', 'fixedPoint', 'bool']

	onMount(() => {
		makeArgs()
		document.addEventListener('lamdenWalletInfo', (event) => {
			console.log(event)
			if (event.detail.errors) console.log(event.detail.errors)
			else{
				console.log(event.detail)
				lamdenInfo = {...event.detail};
			}

		});
		document.addEventListener('lamdenWalletTxStatus', (event) => {
			console.log(event.detail)
			//txStatus = {...event.detail};
		});

		return () => {
			document.removeEventListener('lamdenWalletInfo')
			document.removeEventListener('lamdenWalletTxStatus')
		}
	})

	function connectToWallet(){
		const charms = [
			{
				formatAs: "number",
				iconPath: "/images/refinery.png",
				variableName: "balances",
				key: "<wallet vk>",
				name: "Current Energy"
			},
			{
				formatAs: "number",
				iconPath: "/images/factory.png",
				variableName: "balances",
				key: "<wallet vk>",
				name: "Ground Units"
			}
		]
		const detail = JSON.stringify({
			appName: 'Lamden World',
			description: 'Welcome to Lamden World!  Please click "approve" to connect your Lamden Wallet to the game.',
			//contractName: approveContract,
			contractName: 'currency',
			//contractName: 'submission',
			networkType: approveNetwork,
			preApproval: {
				stampsToPreApprove: 1000000, 
				message: 'This game requires a lot of transactions. To streamline the experience we recommed setting a pre-approve amount.'
			},
			charms,
			logo: "/images/flag.png",
			background: "/images/background.png",
			reapprove: true,
			//newKeypair: true
		})
		console.log(detail)
		document.dispatchEvent(new CustomEvent('lamdenWalletConnect', {detail}));
	}

	function sendTx(){
		const detail = JSON.stringify({networkType, methodName, kwargs: makeArgs(), stampLimit});
		document.dispatchEvent(new CustomEvent('lamdenWalletSendTx', {detail}));
	}

	function makeArgs(){
		let kwargsObj = {}
		Object.keys(kwargs).forEach(kwarg => {
			kwargsObj[kwarg] = kwargs[kwarg].value
		})
		return kwargsObj
	}

	function getStatus(){
		document.dispatchEvent(new CustomEvent('lamdenWalletGetInfo'));
	}

	function addArg(){
		if (argName === '') return;
		kwargs[argName] = selectedArgType === 'fixedPoint' ? parseFloat(argValue) : argValue
		kwargs = {...kwargs}
		argName = argValue = '';
	}

	function delArg(name){
		delete kwargs[name]
		kwargs = {...kwargs}
	}

	function checkSelectedVk(){
		console.log(selectedVk)
	}

	function setValidation(node, message){
        node.setCustomValidity(message);
        node.reportValidity();
	}

    function refreshValidity(){ 
        setValidation(argNameInput, '')
	}

	function isUndefiend(value){
		return typeof value === 'undefined';
	}

</script>


<main class="flex-col">
	<h1>Lamden Wallet Integration Example</h1>
	<div class="flex-row">
		<div class="input-item">
			<label>{'Network Name:'}</label>
			<input bind:value={approveNetwork} required/>
		</div>
		<div class="input-item">
			<label>{'Contract Name:'}</label>
			<input bind:value={approveContract} required/>
		</div>
	</div>
	<button on:click={connectToWallet}>Request Connection Approval</button>

	<div class="tests">
		<h2>{`Wallet Status?`}</h2>
		{#if lamdenInfo}
			<p>{`Wallet Installed: ${isUndefiend(lamdenInfo.installed) ? '' : lamdenInfo.installed}`}</p>
			<p>{`Wallet Setup: ${isUndefiend(lamdenInfo.setup) ? '' : lamdenInfo.setup}`}</p>
			<p>{`Wallet Locked: ${isUndefiend(lamdenInfo.locked) ? '' : lamdenInfo.locked}`}</p>
			<p>{`Wallet Vk: ${isUndefiend(lamdenInfo.wallets[0]) ? 'None' : lamdenInfo.wallets[0]}`}</p>
			{#if !isUndefiend(lamdenInfo.approvals)}
			<h2>{`Contract Authorizations`}</h2>
				<p>{`Mainnet: ${isUndefiend(lamdenInfo.approvals['mainnet']) ? 'None' : lamdenInfo.approvals['mainnet']}`}</p>
				<p>{`Testnet: ${isUndefiend(lamdenInfo.approvals['testnet']) ? 'None' : lamdenInfo.approvals['testnet']}`}</p>
				<p>{`mockchain: ${isUndefiend(lamdenInfo.approvals['mockchain']) ? 'None' : lamdenInfo.approvals['mockchain']}`}</p>
			{/if}
		{/if}
		<button on:click={getStatus}>Check Status</button>
	</div>

	<h2>{`Create Transaction`}</h2>
    <form on:submit|preventDefault={sendTx} target="_self">
		<div class="flex-row">
			<div class="input-item">
				<label>{'Method Name:'}</label>
				<input bind:value={methodName} required/>
			</div>
			<div class="input-item">
				<label>{'Stamp Limit:'}</label>
				<input bind:value={stampLimit} required/>
			</div>
		</div>

		<div class="input-item">
			<label>{'Wallet VK (created by the wallet and is specific to this url):'}</label>
			<select class="vkSelect" bind:value={selectedVk}>
				<option value={''} selected={true}>{`Select a VK (public key)`}</option>
				{#each lamdenInfo.wallets as vk, index}
					<option value={vk}>{`${index + 1}: ${vk}`}</option>
				{/each}
			</select>
			<button on:click={checkSelectedVk}>Check Vk</button>
		</div>

		<div>
			<label>{'Add Kwarg:'}</label>
			<div class="add-arg">
				<input bind:this={argNameInput} bind:value={argName} placeholder="Arg Name" />
				<select bind:value={selectedArgType} >
					{#each ArgTypes as type}
						<option value={type} selected={type === 'text'}>{type}</option>
					{/each}
				</select>
				<input bind:value={argValue} placeholder="Arg Value" />
				<button type="button" on:click={addArg}>Add Kwarg</button>
			</div>
		</div>

		<div>
            <label>{'Kwarg List:'}</label>
			{#if Object.keys(kwargs).length === 0}
				<p>No Kwargs Added</p>
			{:else}
				{#each Object.keys(kwargs) as kwarg}
					<div class="arg">
						<div class="arg-heading flex-row">
							<div class="name flex-col">
								<label>{kwarg.toUpperCase()}</label>
							</div>
							<button class="del-btn"  type="button" on:click={() => delArg(kwarg)}>X</button>	
						</div>
						<div>
							<div class="flex-row">
								<label>{'TYPE:'}</label>{kwargs[kwarg].type}
							</div>
							<div class="flex-row">
								<label>{'VALUE:'}</label>{kwargs[kwarg].value}
							</div>	
						</div>
					</div>
				{/each}
			{/if}
        </div>

		<input type="submit" value="Send Transaction">
    </form>

	<h2>{`Transaction Status`}</h2>
	{#if txStatus !== ''}
		<div class="flex-row">
			<label>{`Transaction Status: `}</label>{txStatus.status}
		</div>
		
		<label>{`Transaction Data: `}</label>
		<div class="tx-data">
			{#if !isUndefiend(txStatus.data)}
				{#each Object.keys(txStatus.data) as item}
					<label>{`${item}:`}</label>
					{#if item === 'state_changes' && Object.keys(txStatus.data[item]).length > 0}
						{#each Object.keys(txStatus.data[item]) as stateChange}
							<div class="data-value">{`${stateChange}: ${JSON.stringify(txStatus.data[item][stateChange])}`}</div>
						{/each}
					{:else}
						<div class="data-value">{JSON.stringify(txStatus.data[item])}</div>
					{/if}
				{/each}
			{/if}
		</div>

	{/if}

</main>

<style>
	main {
		padding: 1em;
		margin: 0 auto;
	}

	h1 {
		color: #ff3e00;
		text-transform: uppercase;
		font-size: 3em;
		font-weight: 100;
	}
	label{
		font-weight: 600;
		margin-right: 5px;
	}
	.tests{
		margin-bottom: 2rem;
		text-align: left;
	}
	.flex-row{
		display: flex;
		flex-direction: row;
	}
	.flex-col{
		display: flex;
		flex-direction: column;
	}
	.arg{
		padding: 10px;
		border-top: 1px solid rgb(100, 100, 100);
		margin: 10px 0px;
		align-items: center;
		min-width: 600px;
		width: 25%;
	}
	.arg:last-child{
		border-bottom: 1px solid rgb(100, 100, 100);
	}
	.arg, .arg-heading{
		align-items: flex-start;
		justify-content: space-between;
	}
	.name{
		color: rgb(74, 0, 160);
	}
	.input-item{
		padding: 5px;
	}
	input[type="submit"]{
		background-color: rgb(160, 255, 160);
	}
	button.del-btn{
		padding: 0px 6px 1px 7px;
		background-color: #ff2828;
		font-weight: 700;
		border-radius: 5px;
	}
	.vkSelect{
		min-width: 600px;	
	}
	.tx-data{
		margin-left: 20px;
	}
	.data-value{
		margin-bottom: 1rem;
	}
	@media (min-width: 640px) {
		main {
			max-width: none;
		}
	}
</style>