<script>
	import { onMount } from 'svelte'
	import MD5 from "crypto-js/md5";

	//FORM Binds 
	let methodName = 'set_value';
	let networkType = 'testnet'
	let stampLimit = 50000;
	let argName, argType, argValue
	let approveNetwork = 'testnet'
	let approveContract = 'con_wallet_testing'

	//DOM Binds
	let selectedArgType, selectedVk, argNameInput

	$: walletInstalled = false;
	$: lamdenInfo = {wallets: []};
	$: errors = [];
	let txStatus = '';
	let approvalHash = '';
	let reapprove = false;
	let newKeypair = false;

	$: kwargs = {
		key_name: {type: 'text', value:''},
		key_value: {type: 'text', value: new Date().toLocaleTimeString()}
	}

	$: contractMethods = null;

	let ArgTypes = ['text', 'address', 'data', 'fixedPoint', 'bool']

	onMount(() => {
		makeArgs()
		document.addEventListener('lamdenWalletInfo', (event) => {
			walletInstalled = true;
			console.log(event)
			if (event.detail.errors) {
				errors = event.detail.errors; 
				console.log(event.detail.errors)
			}
			else{
				console.log(event.detail)
				errors= [];
				lamdenInfo = {...event.detail};
				if (lamdenInfo.wallets.length > 0 ) {
					kwargs['key_name'].value = lamdenInfo.wallets[0] + ':time'
					kwargs = {...kwargs}
				}

			}

		});
		document.addEventListener('lamdenWalletTxStatus', (event) => {
			console.log(event)
			let detail = event.detail
			if (detail.status === "error"){
				errors = detail.data.errors || detail.errors || detail.data.resultInfo.errorInfo
			}
			txStatus = {...event.detail};
		});

		checkForWallet()

		return () => {
			document.removeEventListener('lamdenWalletInfo')
			document.removeEventListener('lamdenWalletTxStatus')
		}
	})

	const checkForWallet = () => {	
		document.dispatchEvent(new CustomEvent('lamdenWalletGetInfo'));
		setTimeout(() => {
			if (!walletInstalled) checkForWallet()
		}, 1000)
	}

	function connectToWallet(){
		const charms = [
			{
				formatAs: "string",
				iconPath: "/images/thumbsup.png",
				variableName: "yourState",
				key: "<wallet vk>:time",
				name: "Last Updated"
			}
		]
		let detail = {
			appName: 'Integration Testing',
			description: 'Congrats! Your Dapp sent a successful approval request! Click APPROVE to create a dApp connection.',
			contractName: approveContract,
			networkType: approveNetwork,
			preApproval: {
				stampsToPreApprove: 1000000, 
				message: 'Please pre-approve some stamps that we can use for testing.'
			},
			charms,
			logo: "/images/star.png",
			background: "/images/background.png"
		}
		approvalHash = MD5(JSON.stringify(detail)).toString();
		errors = [];
		if (reapprove) {
			detail.reapprove = true
			if (newKeypair) detail.newKeypair = true
		}
		detail = JSON.stringify(detail)
		document.dispatchEvent(new CustomEvent('lamdenWalletConnect', {detail}));
	}

	function sendTx(){
		errors = [];
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
		errors = [];
		document.dispatchEvent(new CustomEvent('lamdenWalletGetInfo'));
	}

	function addArg(){
		if (argName === '') return;
		if (typeof argValue === 'undefined') return
		kwargs[argName] = {
			value: selectedArgType === 'fixedPoint' ? parseFloat(argValue) : argValue, 
			type: selectedArgType
		}
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
	const toggleNewKeypair = () => reapprove ? newKeypair = false : null

	const openLink = (url) => {
        window.open(url, '_blank');
	}
</script>


<main class="flex-col">
	<h1>Lamden Wallet Integration Demo</h1>
	<button 
		class="install-box flex-row" 
		class:installed={walletInstalled} 
		on:click={() => openLink('https://chrome.google.com/webstore/detail/lamden-wallet-browser-ext/fhfffofbcgbjjojdnpcfompojdjjhdim')}>
		{#if walletInstalled}
			Wallet Installed
		{:else}
			Install Lamden Wallet
		{/if}
	</button>
	{#if walletInstalled}
		{#if errors.length > 0}
			<div class="errors-box flex-col">
				{#each errors as error}
					<p>{error}</p>
				{/each}
			</div>
		{/if}
		<div class="approval flex-col">
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
			<button on:click={connectToWallet}>Send Approval</button>
			<div class="flex-row">
				<lable>reappove conenction</lable>
				<input type="checkbox" bind:checked={reapprove} on:click={toggleNewKeypair}/>
				<lable>create new KeyPair</lable>
				<input type="checkbox" bind:checked={newKeypair} disabled={!reapprove}/>
			</div>
		</div>

		<div>
			<p>{`Hash of sent approval: ${approvalHash}`}</p>
		</div>


		<div class="tests">
			<h2>{`Wallet Status?`}</h2>
			<button on:click={getStatus}>Get Wallet Info</button>
			{#if lamdenInfo}
				<p>{`Wallet Installed: ${isUndefiend(lamdenInfo.installed) ? '' : lamdenInfo.installed}`}</p>
				<p>{`Wallet Setup: ${isUndefiend(lamdenInfo.setup) ? '' : lamdenInfo.setup}`}</p>
				<p>{`Wallet Locked: ${isUndefiend(lamdenInfo.locked) ? '' : lamdenInfo.locked}`}</p>
				<p>{`Wallet Vk: ${isUndefiend(lamdenInfo.wallets[0]) ? 'None' : lamdenInfo.wallets[0]}`}</p>
				{#if !isUndefiend(lamdenInfo.approvals)}
				<h2>{`Contract Authorizations`}</h2>
					{#each Object.keys(lamdenInfo.approvals) as approval}
					<p>{`${approval.toUpperCase()}: Approved Contract is ${lamdenInfo.approvals[approval].contractName} and Approval Hash is ${lamdenInfo.approvals[approval].approvalHash}`}</p>
					{/each}
				{/if}
			{/if}
			
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

	{/if}
</main>

<style>
	main {
		padding: 1em;
		margin: 0 auto;
		max-width: 1020px;
	}
	.install-box{
		background: #461BC2;
		color: #eae3ff;
		width: 260px;
		margin: 0 auto 1rem auto;
		padding: 15px;
		justify-content: center;
		border-radius: 10px;
		box-shadow: 0px 0px 0px 0px rgba(0, 0, 0, 0);
		font-size: 1.3em;
	}
	.install-box:hover{
		background: rgb(86, 41, 221);
		box-shadow: 3px 3px 24px -10px rgba(0, 0, 0, 0.904);
	}
	.install-box > a{
		color: #d0d5ff;
	}
	.install-box.installed{
		background: #0fb40f;
		color: #e0ffe0;
	}
	.approval{
		max-width: 550px;
		margin: 0 auto;
		align-items: center;
	}
	h1 {
		color: #461BC2;
		text-transform: uppercase;
		font-size: 3em;
		font-weight: 100;
		text-align: center;
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
	.errors-box{
		border: 1px solid #5f0000;
		text-align: center;
		margin: 1rem 0;
	}
	.errors-box > p{
		color: red;
	}
	input[type=checkbox]{
		margin: 0 1rem 0 0.5rem; 
	}
</style>