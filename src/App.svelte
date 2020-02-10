<script>
	import { onMount } from 'svelte'

	//Form Binds
	let contractName = 'currency'; 
	let methodName = 'transfer';
	let senderVk, argName, argType, argValue

	//DOM Binds
	let selectedArgType, selectedVk, argNameInput

	$: lamdenInfo = {wallets: []};
	let txStatus = '';
	$: kwargs = {
		to: {
			type: "address", 
			value: '384138d6965b32c49e3bec94d3f239f0994ee75dbf9ca59c8698617843e9ef7e'
		},
		amount: {
			type: "fixedPoint", 
			value: 1000
		}
	}

	let ArgTypes = ['text', 'address', 'data', 'fixedPoint', 'bool']

	onMount(() => {
		document.addEventListener('lamdenWalletInfo', (event) => {
			console.log(`Webpage Wallet Status? ${JSON.stringify(event.detail)}`)
			lamdenInfo = {...event.detail};
		});
		document.addEventListener('txStatus', (event) => {
			console.log(`${JSON.stringify(event.detail)}`)
			txStatus = {...event.detail};
		});

		return () => {
			document.removeEventListener('lamdenWalletStatus')
			document.removeEventListener('txStatus')
		}
	})

	function sendTx(){
		let detail = {senderVk: selectedVk, contractName, methodName, kwargs};

		document.dispatchEvent(new CustomEvent('signTx', {detail}));
		clearAllValues();
	}

	function getStatus(){
		document.dispatchEvent(new CustomEvent('getLamdenWalletInfo'));
	}

	function addArg(){
		if (argName === '') return;
		kwargs[argName] = {
			type: selectedArgType, 
			value: argValue
		}
		kwargs = {...kwargs}
		clearArgValues();
	}

	function delArg(name){
		delete kwargs[name]
		kwargs = {...kwargs}
	}

	function clearArgValues() {
		argName = argValue = '';
	}

	function clearFromValues() {
		contractName = methodName = senderVk = '';
	}

	function clearAllValues(){
		clearArgValues();
		clearFromValues();
		kwargs = {};
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
	<div class="tests">
		<h2>{`Wallet Status?`}</h2>
		{#if lamdenInfo}
			<p>{`Wallet Installed: ${isUndefiend(lamdenInfo.installed) ? '' : lamdenInfo.installed}`}</p>
			<p>{`Wallet Setup: ${isUndefiend(lamdenInfo.setup) ? '' : lamdenInfo.setup}`}</p>
			<p>{`Wallet Locked: ${isUndefiend(lamdenInfo.locked) ? '' : lamdenInfo.locked}`}</p>
			<p>{`Number of Wallets: ${lamdenInfo.wallets.length}`}</p>
			{#if !isUndefiend(lamdenInfo.currentNetwork)}
				<p>{`Current Network: ${lamdenInfo.currentNetwork.name}: ${lamdenInfo.currentNetwork.ip}:${lamdenInfo.currentNetwork.port}`}</p>
			{/if}
		{/if}
		<button on:click={getStatus}>Check Status</button>
	</div>

	<h2>{`Create Transaction`}</h2>
    <form on:submit|preventDefault={sendTx} target="_self">
		<div class="flex-row">
		    <div class="input-item">
				<label>{'Contract Name:'}</label>
				<input bind:value={contractName} required/>
        	</div>
			<div class="input-item">
				<label>{'Method Name:'}</label>
				<input bind:value={methodName} required/>
			</div>
		</div>

		<div class="input-item">
			<label>{'Sender VK (public key):'}</label>
			<select class="vkSelect" bind:value={selectedVk}>
				<option value={''} selected={true}>{`Select a VK (public key)`}</option>
				{#each lamdenInfo.wallets as wallet, index}
					<option value={wallet.vk}>{`${index + 1}: ${wallet.vk}`}</option>
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