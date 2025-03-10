<script lang="ts">
	import HunkContextMenu from './HunkContextMenu.svelte';
	import HunkLine from './HunkLine.svelte';
	import { draggable } from '$lib/dragging/draggable';
	import { draggableHunk } from '$lib/dragging/draggables';
	import { onDestroy } from 'svelte';
	import type { HunkSection } from '$lib/utils/fileSections';
	import type { BranchController } from '$lib/vbranches/branchController';
	import type { Ownership } from '$lib/vbranches/ownership';
	import type { Hunk } from '$lib/vbranches/types';
	import type { Writable } from 'svelte/store';

	export let filePath: string;
	export let section: HunkSection;
	export let branchId: string | undefined;
	export let projectPath: string | undefined;

	export let minWidth: number;
	export let selectable = false;
	export let isUnapplied: boolean;
	export let isFileLocked: boolean;
	export let readonly: boolean = false;

	export let branchController: BranchController;
	export let selectedOwnership: Writable<Ownership> | undefined = undefined;

	function onHunkSelected(hunk: Hunk, isSelected: boolean) {
		if (!selectedOwnership) return;
		if (isSelected) {
			selectedOwnership.update((ownership) => ownership.addHunk(hunk.filePath, hunk.id));
		} else {
			selectedOwnership.update((ownership) => ownership.removeHunk(hunk.filePath, hunk.id));
		}
	}

	function updateContextMenu(filePath: string) {
		if (popupMenu) popupMenu.$destroy();
		return new HunkContextMenu({
			target: document.body,
			props: { projectPath, filePath, branchController }
		});
	}

	$: popupMenu = updateContextMenu(filePath);

	$: draggingDisabled = readonly || isUnapplied || section.hunk.locked || !branchId;

	onDestroy(() => {
		if (popupMenu) {
			popupMenu.$destroy();
		}
	});
</script>

<div
	tabindex="0"
	role="cell"
	use:draggable={{
		...draggableHunk(branchId, section.hunk),
		disabled: draggingDisabled
	}}
	on:contextmenu|preventDefault
	class="hunk"
	class:readonly
	class:opacity-60={section.hunk.locked && !isFileLocked}
>
	<div class="hunk__bg-stretch">
		{#each section.subSections as subsection}
			{@const hunk = section.hunk}
			{#each subsection.lines.slice(0, subsection.expanded ? subsection.lines.length : 0) as line}
				<HunkLine
					{line}
					{filePath}
					{readonly}
					{minWidth}
					{selectable}
					{draggingDisabled}
					selected={$selectedOwnership?.containsHunk(hunk.filePath, hunk.id)}
					on:selected={(e) => onHunkSelected(hunk, e.detail)}
					sectionType={subsection.sectionType}
					on:contextmenu={(e) =>
						popupMenu.openByMouse(e, {
							hunk,
							section: subsection,
							lineNumber: line.afterLineNumber ? line.afterLineNumber : line.beforeLineNumber
						})}
				/>
			{/each}
		{/each}
	</div>
</div>

<style lang="postcss">
	.hunk {
		display: flex;
		flex-direction: column;
		overflow-x: auto;

		background: var(--clr-theme-container-light);
		border-radius: var(--radius-s);
		border: 1px solid var(--clr-theme-container-outline-light);
		transition: border-color var(--transition-fast);
	}

	.hunk__bg-stretch {
		width: 100%;
		min-width: max-content;
	}
</style>
