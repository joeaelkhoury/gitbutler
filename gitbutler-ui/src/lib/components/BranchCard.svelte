<script lang="ts">
	import BranchCommits from './BranchCommits.svelte';
	import BranchFiles from './BranchFiles.svelte';
	import BranchHeader from './BranchHeader.svelte';
	import CommitDialog from './CommitDialog.svelte';
	import DropzoneOverlay from './DropzoneOverlay.svelte';
	import ImgThemed from '$lib/components/ImgThemed.svelte';
	import Resizer from '$lib/components/Resizer.svelte';
	import { projectAiGenEnabled } from '$lib/config/config';
	import { projectAiGenAutoBranchNamingEnabled } from '$lib/config/config';
	import {
		isDraggableFile,
		isDraggableHunk,
		isDraggableRemoteCommit,
		type DraggableFile,
		type DraggableHunk,
		type DraggableRemoteCommit
	} from '$lib/dragging/draggables';
	import { dropzone } from '$lib/dragging/dropzone';
	import { persisted } from '$lib/persisted/persisted';
	import { SETTINGS_CONTEXT, type SettingsStore } from '$lib/settings/userSettings';
	import { computeAddedRemovedByFiles } from '$lib/utils/metrics';
	import { filesToOwnership, type Ownership } from '$lib/vbranches/ownership';
	import lscache from 'lscache';
	import { getContext, onMount } from 'svelte';
	import { get, type Writable } from 'svelte/store';
	import type { User, getCloudApiClient } from '$lib/backend/cloud';
	import type { Project } from '$lib/backend/projects';
	import type { BranchService } from '$lib/branches/service';
	import type { GitHubService } from '$lib/github/service';
	import type { Persisted } from '$lib/persisted/persisted';
	import type { BranchController } from '$lib/vbranches/branchController';
	import type { BaseBranch, Branch, LocalFile } from '$lib/vbranches/types';

	export let branch: Branch;
	export let isUnapplied = false;
	export let project: Project;
	export let base: BaseBranch | undefined | null;
	export let cloud: ReturnType<typeof getCloudApiClient>;
	export let branchService: BranchService;
	export let branchController: BranchController;
	export let branchCount = 1;
	export let user: User | undefined;
	export let selectedFiles: Writable<LocalFile[]>;
	export let githubService: GitHubService;
	export let selectedOwnership: Writable<Ownership>;
	export let commitBoxOpen: Writable<boolean>;

	export let isLaneCollapsed: Persisted<boolean>;

	const aiGenEnabled = projectAiGenEnabled(project.id);
	const aiGenAutoBranchNamingEnabled = projectAiGenAutoBranchNamingEnabled(project.id);

	let rsViewport: HTMLElement;

	const userSettings = getContext<SettingsStore>(SETTINGS_CONTEXT);
	const defaultBranchWidthRem = persisted<number | undefined>(24, 'defaulBranchWidth' + project.id);
	const laneWidthKey = 'laneWidth_';

	let laneWidth: number;

	$: if ($commitBoxOpen && branch.files.length === 0) {
		$commitBoxOpen = false;
	}

	function generateBranchName() {
		const diff = branch.files
			.map((f) => f.hunks)
			.flat()
			.map((h) => h.diff)
			.flat()
			.join('\n')
			.slice(0, 5000);

		if (user && aiGenEnabled) {
			cloud.summarize.branch(user.access_token, { diff }).then((result) => {
				if (result.message && result.message !== branch.name) {
					branch.name = result.message;
					branchController.updateBranchName(branch.id, branch.name);
				}
			});
		}
	}

	$: linesTouched = computeAddedRemovedByFiles(...branch.files);
	$: if (
		$aiGenAutoBranchNamingEnabled &&
		branch.name.toLowerCase().includes('virtual branch') &&
		linesTouched.added + linesTouched.removed > 4
	) {
		generateBranchName();
	}

	onMount(() => {
		laneWidth = lscache.get(laneWidthKey + branch.id);
	});

	function acceptCherrypick(data: any) {
		return isDraggableRemoteCommit(data) && data.branchId == branch.id;
	}

	function onCherrypicked(data: DraggableRemoteCommit) {
		branchController.cherryPick(branch.id, data.remoteCommit.id);
	}

	function acceptBranchDrop(data: any) {
		if (isDraggableHunk(data) && data.branchId != branch.id) {
			return true;
		} else if (isDraggableFile(data) && data.branchId != branch.id) {
			return true;
		} else {
			return false;
		}
	}

	function onBranchDrop(data: DraggableHunk | DraggableFile) {
		if (isDraggableHunk(data)) {
			const newOwnership = `${data.hunk.filePath}:${data.hunk.id}`;
			branchController.updateBranchOwnership(
				branch.id,
				(newOwnership + '\n' + branch.ownership).trim()
			);
		} else if (isDraggableFile(data)) {
			let files = get(data.files);
			if (files.length == 0) {
				files = [data.current];
			}
			const newOwnership = filesToOwnership(files);
			branchController.updateBranchOwnership(
				branch.id,
				(newOwnership + '\n' + branch.ownership).trim()
			);
		}
	}
</script>

{#if $isLaneCollapsed}
	<div class="collapsed-lane-wrapper">
		<BranchHeader
			{isUnapplied}
			{branchController}
			{branch}
			{base}
			{githubService}
			{branchService}
			bind:isLaneCollapsed
			projectId={project.id}
			on:action={(e) => {
				if (e.detail == 'generate-branch-name') {
					generateBranchName();
				}
			}}
		/>
	</div>
{:else}
	<div
		class="branch-card"
		data-tauri-drag-region
		class:target-branch={branch.active && branch.selectedForChanges}
	>
		<div
			bind:this={rsViewport}
			style:width={`${laneWidth || $defaultBranchWidthRem}rem`}
			class="branch-card__contents"
		>
			<BranchHeader
				{isUnapplied}
				{branchController}
				{branch}
				{base}
				{githubService}
				{branchService}
				bind:isLaneCollapsed
				projectId={project.id}
				on:action={(e) => {
					if (e.detail == 'generate-branch-name') {
						generateBranchName();
					}
				}}
			/>
			<!-- DROPZONES -->
			<DropzoneOverlay class="cherrypick-dz-marker" label="Apply here" />
			<DropzoneOverlay class="lane-dz-marker" label="Move here" />

			<div
				class="branch-card__dropzone-wrapper"
				use:dropzone={{
					hover: 'cherrypick-dz-hover',
					active: 'cherrypick-dz-active',
					accepts: acceptCherrypick,
					onDrop: onCherrypicked,
					disabled: isUnapplied
				}}
				use:dropzone={{
					hover: 'lane-dz-hover',
					active: 'lane-dz-active',
					accepts: acceptBranchDrop,
					onDrop: onBranchDrop,
					disabled: isUnapplied
				}}
			>
				<DropzoneOverlay class="cherrypick-dz-marker" label="Apply here" />
				<DropzoneOverlay class="lane-dz-marker" label="Move here" />
				{#if branch.files?.length > 0}
					<div class="card">
						<BranchFiles
							{branch}
							{isUnapplied}
							{selectedOwnership}
							{selectedFiles}
							showCheckboxes={$commitBoxOpen}
						/>
						{#if branch.active}
							<CommitDialog
								projectId={project.id}
								{branchController}
								{branch}
								{cloud}
								{selectedOwnership}
								{user}
								bind:expanded={commitBoxOpen}
								on:action={(e) => {
									if (e.detail == 'generate-branch-name') {
										generateBranchName();
									}
								}}
							/>
						{/if}
					</div>
				{:else if branch.commits.length == 0}
					<div class="new-branch card" data-dnd-ignore>
						<div class="new-branch__content">
							<div class="new-branch__image">
								<ImgThemed
									imgSet={{
										light: '/images/lane-new-light.webp',
										dark: '/images/lane-new-dark.webp'
									}}
								/>
							</div>
							<h2 class="new-branch__title text-base-body-15 text-semibold">
								This is a new branch.
							</h2>
							<p class="new-branch__caption text-base-body-13">
								You can drag and drop files or parts of files here.
							</p>
						</div>
					</div>
				{:else}
					<!-- attention: these markers have custom css at the bottom of thise file -->
					<div class="no-changes card" data-dnd-ignore>
						<div class="new-branch__content">
							<div class="new-branch__image">
								<ImgThemed
									imgSet={{
										light: '/images/lane-no-changes-light.webp',
										dark: '/images/lane-no-changes-dark.webp'
									}}
								/>
							</div>
							<h2 class="new-branch__caption text-base-body-13">
								No uncommitted changes<br />on this branch
							</h2>
						</div>
					</div>
				{/if}
			</div>

			<BranchCommits
				{base}
				{branch}
				{project}
				{githubService}
				{branchController}
				{branchService}
				{branchCount}
				{isUnapplied}
				{selectedFiles}
			/>
		</div>

		<div class="divider-line">
			<Resizer
				viewport={rsViewport}
				direction="right"
				inside={$selectedFiles.length > 0}
				minWidth={320}
				sticky
				on:width={(e) => {
					laneWidth = e.detail / (16 * $userSettings.zoom);
					lscache.set(laneWidthKey + branch.id, laneWidth, 7 * 1440); // 7 day ttl
					$defaultBranchWidthRem = laneWidth;
				}}
			/>
		</div>
	</div>
{/if}

<style lang="postcss">
	/* .branch-card-wrapper {
		position: relative;
		display: flex;
		height: 100%;
	} */
	.branch-card {
		height: 100%;
		position: relative;
		user-select: none;
		overflow-x: hidden;
		overflow-y: scroll;

		&::-webkit-scrollbar {
			width: 0px;
			background: transparent; /* Chrome/Safari/Webkit */
		}
	}

	.divider-line {
		position: absolute;
		top: 0;
		right: 0;
		width: var(--space-4);
		height: 100%;
		transform: translateX(var(--selected-resize-shift));

		&:after {
			pointer-events: none;
			content: '';
			position: absolute;
			top: 0;
			right: 0;
			width: 1px;
			height: 100%;
			opacity: var(--selected-opacity);
			background-color: var(--clr-theme-container-outline-light);
		}
	}

	.branch-card__dropzone-wrapper {
		display: flex;
		flex-direction: column;
		flex: 1;
		position: relative;
	}

	.branch-card__contents {
		position: relative;
		display: flex;
		flex-direction: column;
		flex: 1;
		min-height: 100%;
		gap: var(--space-8);
		padding: var(--space-12);
	}

	.card {
		flex: 1;
	}

	.new-branch__content {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: var(--space-8);
		max-width: 14rem;
	}

	.new-branch,
	.no-changes {
		user-select: none;
		display: flex;
		flex-grow: 1;
		flex-direction: column;
		align-items: center;
		color: var(--clr-theme-scale-ntrl-60);
		background: var(--clr-theme-container-light);
		justify-content: center;
		padding: var(--space-48) 0;
		border-radius: var(--radius-m);
	}

	.no-changes {
		color: var(--clr-theme-scale-ntrl-40);
		text-align: center;
	}

	.new-branch__title {
		color: var(--clr-theme-scale-ntrl-40);
	}

	.new-branch__caption {
		color: var(--clr-theme-scale-ntrl-50);
		opacity: 0.6;
	}

	.new-branch__caption,
	.new-branch__title {
		text-align: center;
	}

	.new-branch__image {
		width: 7.5rem;
		margin-bottom: var(--space-10);
	}

	/* hunks drop zone */
	:global(.lane-dz-active .lane-dz-marker) {
		display: flex;
	}

	/* cherry pick drop zone */
	:global(.cherrypick-dz-active .cherrypick-dz-marker) {
		@apply flex;
	}

	/* squash drop zone */
	:global(.squash-dz-active .squash-dz-marker) {
		@apply flex;
	}

	.branch-card :global(.contents) {
		display: flex;
		flex-direction: column;
		min-height: 100%;
	}

	/* COLLAPSED LANE */
	.collapsed-lane-wrapper {
		display: flex;
		flex-direction: column;
		padding: var(--space-12);
		height: 100%;
		border-right: 1px solid var(--clr-theme-container-outline-light);
	}
</style>
