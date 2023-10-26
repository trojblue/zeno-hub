<script lang="ts">
	import { instanceOfFilterPredicate } from '$lib/api/slice';
	import { getFilteredTable } from '$lib/api/table';
	import {
		columns,
		model,
		project,
		rowsPerPage,
		selectionIds,
		selectionPredicates,
		selections,
		sort,
		tagIds
	} from '$lib/stores';
	import { Join, ZenoColumnType, ZenoService } from '$lib/zenoapi';
	import { Label } from '@smui/button';
	import { Pagination } from '@smui/data-table';
	import IconButton from '@smui/icon-button';
	import InstanceView from '@zeno-ml/zeno-instance-views';
	import { getContext } from 'svelte';

	export let numberOfInstances = 0;

	const zenoClient = getContext('zenoClient') as ZenoService;

	let tablePromise: Promise<Record<string, string | number | boolean>[]>;
	let currentPage = 0;
	let lastPage = 0;
	let sampleOptions = [
		...new Set(
			$project.samplesPerPage !== undefined
				? [5, 15, 30, 60, 100, $project.samplesPerPage]
				: [5, 15, 30, 60, 100]
		)
	].sort((a, b) => a - b);

	$: idColumn = $columns.find((col) => col.columnType === ZenoColumnType.ID)?.id;
	$: dataColumn = $columns.find((col) => col.columnType === ZenoColumnType.DATA)?.id;
	$: labelColumn = $columns.find((col) => col.columnType === ZenoColumnType.LABEL)?.id;

	$: start = currentPage * $rowsPerPage;
	$: end = start + $rowsPerPage;
	$: lastPage = Math.max(Math.ceil(numberOfInstances / $rowsPerPage) - 1, 0);

	$: if (currentPage > lastPage) {
		currentPage = lastPage;
	}

	// when state changes update current table view
	$: {
		currentPage;
		$rowsPerPage;
		$columns;
		$selections.tags;
		$model;
		$sort;
		$tagIds;
		$selectionIds;
		updateTable();
	}

	$: modelColumn = $columns.find(
		(col) => col.columnType === ZenoColumnType.OUTPUT && col.model === $model
	);

	// reset page on selection change
	selectionPredicates.subscribe(() => {
		if (currentPage !== 0) {
			currentPage = 0;
		}
	});

	function updateTable() {
		if (isNaN(start) || isNaN(end) || end <= start) return;
		let predicates = $selectionPredicates;
		if (predicates !== undefined && instanceOfFilterPredicate(predicates)) {
			predicates = {
				join: Join._,
				predicates: [predicates]
			};
		}
		const secureTagIds = $tagIds === undefined ? [] : $tagIds;
		const secureSelectionIds = $selectionIds === undefined ? [] : $selectionIds;
		const dataIds = [...new Set([...secureTagIds, ...secureSelectionIds])];
		tablePromise = getFilteredTable(
			$project.uuid,
			$columns,
			$model ? [$model] : [],
			undefined,
			start,
			end - start,
			$sort,
			dataIds,
			zenoClient,
			predicates
		);
	}
</script>

<div class="overflow-y-auto flex flex-wrap content-start w-full h-full">
	{#await tablePromise then table}
		{#if idColumn !== undefined}
			{#each table as inst (inst[idColumn])}
				<div class="mr-2 mt-2">
					<InstanceView
						view={$project.view ?? ''}
						{dataColumn}
						{labelColumn}
						modelColumn={modelColumn?.id}
						entry={inst}
					/>
				</div>
			{/each}
		{/if}
	{:catch e}
		<p>error loading data: {e}</p>
	{/await}
</div>
<Pagination slot="paginate" class="pagination">
	<svelte:fragment slot="rowsPerPage">
		<Label>Instances Per Page</Label>
		<select class="ml-2" bind:value={$rowsPerPage}>
			{#each sampleOptions as option}
				<option value={option}>{option}</option>
			{/each}
		</select>
	</svelte:fragment>
	<svelte:fragment slot="total">
		{start + 1} -
		{Math.min(end, numberOfInstances)} of
		{numberOfInstances}
	</svelte:fragment>

	<IconButton
		class="material-icons"
		action="first-page"
		title="First page"
		on:click={() => (currentPage = 0)}
		disabled={currentPage === 0}>first_page</IconButton
	>
	<IconButton
		class="material-icons"
		action="prev-page"
		title="Prev page"
		on:click={() => currentPage--}
		disabled={currentPage === 0}>chevron_left</IconButton
	>
	<IconButton
		class="material-icons"
		action="next-page"
		title="Next page"
		on:click={() => currentPage++}
		disabled={currentPage >= lastPage}>chevron_right</IconButton
	>
	<IconButton
		class="material-icons"
		action="last-page"
		title="Last page"
		on:click={() => (currentPage = lastPage)}
		disabled={currentPage >= lastPage}>last_page</IconButton
	>
</Pagination>