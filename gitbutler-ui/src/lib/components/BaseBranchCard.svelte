<script lang="ts">
	import SyncButton from './SyncButton.svelte';
	import Badge from '$lib/components/Badge.svelte';
	import Icon from '$lib/components/Icon.svelte';
	import type { Project } from '$lib/backend/projects';
	import type { GitHubService } from '$lib/github/service';
	import type { BaseBranchService } from '$lib/vbranches/branchStoresCache';
	import { page } from '$app/stores';

	export let project: Project;
	export let baseBranchService: BaseBranchService;
	export let githubService: GitHubService;

	$: base$ = baseBranchService.base$;
	$: selected = $page.url.href.endsWith('/base');

	let baseContents: HTMLElement;
</script>

<a href="/{project.id}/base" class="base-branch-card" class:selected bind:this={baseContents}>
	<div class="icon">
		<svg width="16" height="16" viewBox="0 0 16 16" fill="none" xmlns="http://www.w3.org/2000/svg">
			<rect width="16" height="16" rx="4" fill="#FB7D61" />
			<path d="M8 4L12 8L8 12L4 8L8 4Z" fill="white" />
		</svg>
	</div>

	<div class="content">
		<div class="row_1">
			<span class="text-base-14 text-semibold trunk-label">Trunk</span>
			{#if ($base$?.behind || 0) > 0}
				<Badge count={$base$?.behind || 0} help="Unmerged upstream commits" />
			{/if}
			<SyncButton
				projectId={project.id}
				{baseBranchService}
				{githubService}
				cloudEnabled={project?.api?.sync || false}
			/>
		</div>
		<div class="row_2 text-base-12">
			{#if $base$?.remoteUrl.includes('github.com')}
				<!-- GitHub logo -->
				<svg
					style="width:0.75rem; height: 0.75rem"
					viewBox="0 0 14 14"
					fill="none"
					xmlns="http://www.w3.org/2000/svg"
				>
					<path
						fill-rule="evenodd"
						clip-rule="evenodd"
						d="M6.98091 0.599976C3.45242 0.599976 0.599976 3.47344 0.599976 7.02832C0.599976 9.86992 2.42763 12.2753 4.96308 13.1266C5.28007 13.1906 5.39619 12.9883 5.39619 12.8181C5.39619 12.6691 5.38574 12.1582 5.38574 11.626C3.61072 12.0092 3.24109 10.8597 3.24109 10.8597C2.95583 10.1147 2.53317 9.92321 2.53317 9.92321C1.9522 9.52941 2.57549 9.52941 2.57549 9.52941C3.21993 9.57199 3.55808 10.1893 3.55808 10.1893C4.12847 11.1683 5.04758 10.8917 5.41735 10.7214C5.47011 10.3063 5.63926 10.0189 5.81885 9.85934C4.40314 9.71031 2.91364 9.15691 2.91364 6.68768C2.91364 5.98525 3.16703 5.41055 3.56853 4.9636C3.50518 4.80399 3.28327 4.14401 3.63201 3.26068C3.63201 3.26068 4.17078 3.09036 5.38561 3.92053C5.90572 3.77982 6.4421 3.70824 6.98091 3.70763C7.51968 3.70763 8.06891 3.78221 8.57607 3.92053C9.79103 3.09036 10.3298 3.26068 10.3298 3.26068C10.6785 4.14401 10.4565 4.80399 10.3932 4.9636C10.8052 5.41055 11.0482 5.98525 11.0482 6.68768C11.0482 9.15691 9.55867 9.6996 8.13238 9.85934C8.36487 10.0615 8.56549 10.4446 8.56549 11.0513C8.56549 11.9133 8.55504 12.6052 8.55504 12.818C8.55504 12.9883 8.67129 13.1906 8.98815 13.1267C11.5236 12.2751 13.3513 9.86992 13.3513 7.02832C13.3617 3.47344 10.4988 0.599976 6.98091 0.599976Z"
						fill="currentColor"
					/>
				</svg>
			{:else}
				<Icon name="branch" />
			{/if}
			{$base$?.branchName}
		</div>
	</div>
</a>

<style lang="postcss">
	.base-branch-card {
		display: flex;
		gap: var(--space-10);
		padding: var(--space-10) var(--space-8);
		border-radius: var(--radius-m);
		transition: background-color var(--transition-fast);
	}

	.base-branch-card:hover,
	.base-branch-card:focus,
	.selected {
		background-color: color-mix(in srgb, transparent, var(--darken-light));
	}

	.icon {
		flex-shrink: 0;
	}
	.content {
		display: flex;
		flex-direction: column;
		gap: var(--space-8);
	}
	.trunk-label {
		color: var(--clr-theme-scale-ntrl-0);
	}
	.row_1 {
		display: flex;
		gap: var(--space-6);
		align-items: center;
		color: var(--clr-theme-scale-ntrl-10);
	}
	.row_2 {
		display: flex;
		align-items: center;
		gap: var(--space-4);
		color: var(--clr-theme-scale-ntrl-40);
	}
</style>
