<script lang="ts">
	import type { ModalSettings, TableSource } from '@skeletonlabs/skeleton';
	import { ProgressRadial, Table, tableMapperValues } from '@skeletonlabs/skeleton';
	import type { Search } from 'sailpoint-api-client';
	import { onMount } from 'svelte';
	import alasql from 'alasql';
	import { getModalStore } from '@skeletonlabs/skeleton';

	const modalStore = getModalStore();
	//export let data;
	let tableSimple: TableSource | undefined = undefined;
	let rawData: any;
	onMount(async () => {
		const search: Search = {
			indices: ['events'],
			query: {
				query: `name: "Create Account Failed" AND created: [now-90d TO now]`,
			},
			sort: ['created'],
		};

		const response = await fetch('/api/sailpoint/search', {
			method: 'POST',
			body: JSON.stringify(search),
			headers: {
				'content-type': 'application/json',
			},
		});

		rawData = await response.json();

		if (JSON.stringify(rawData) !== '{}') {
			let reportResult = [];

			for (let row of <any>rawData) {
				reportResult.push({
					name: row.target.name,
					source: row.attributes.sourceName,
					failure: row.name,
				});
			}

			let res = alasql(
				'SELECT failure, source, name, count(*) as failures FROM ? GROUP BY failure, source, name',
				[reportResult],
			);
			console.log(res);
			tableSimple = {
				// A list of heading labels.
				head: ['Name', 'Source', 'Failure', 'Number Failures'],
				// The data visibly shown in your table body UI.
				body: tableMapperValues(res, ['name', 'source', 'failure', 'failures']),
				// Optional: The data returned when interactive is enabled and a row is clicked.
				meta: tableMapperValues(res, ['name', 'source', 'failure', 'failures']),
			};
		}
	});

	function onTableclick(event: any) {
		console.log(event);
		let exceptions = '';
		for (let row of rawData) {
			if (
				row.target.name == event.detail[0] &&
				row.attributes.sourceName == event.detail[1] &&
				row.name == event.detail[2]
			) {
				console.log(row.attributes.errors);
				exceptions = JSON.stringify(JSON.parse(row.attributes.errors), null, '  ');
			}
		}
		const modal: ModalSettings = {
			type: 'alert',
			// Data
			title: 'Exception Details',
			body: `${exceptions}`,
		};
		modalStore.trigger(modal);
	}
</script>

<main>
	<div class="p-4">
		<img src="/SailPoint-Developer-Community-Lockup.png" alt="sailPoint Logo" />
		<a href="/home" class="btn variant-filled-primary w-full mt-2 text-slate-50 text-lg">
			Go back report screen
		</a>
		<div class="flex justify-center mt-4 flex-col align-middle">
			<div class="text-2xl text-slate-500 divide-dashed divide-y-2 mt-4 mb-2">
				Listing of Source Account Create Errors
			</div>
			{#if tableSimple}
				<Table
					class="w-full"
					source={tableSimple}
					interactive={true}
					on:selected={onTableclick}
				/>
			{:else}
				<div class="progress-bar">
					<ProgressRadial
						stroke={100}
						meter="stroke-primary-500"
						track="stroke-primary-500/30"
						class="progress-bar"
					/>
				</div>
			{/if}
		</div>
	</div>
</main>

<style>
	.progress-bar {
		padding-top: calc(50vh - 4.5rem - 200px);
		padding-left: calc(50% - 4.5rem);
	}
</style>
