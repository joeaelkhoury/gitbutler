<script lang="ts">
	import Icon from '$lib/components/Icon.svelte';
	import type { User } from '$lib/backend/cloud';
	import { goto } from '$app/navigation';

	export let user: User | undefined;
	export let pop = false;
</script>

<button class="btn" class:pop on:click={() => goto('/settings/')}>
	<span class="name text-base-13 text-semibold">
		{#if user}
			{#if user.name}
				{user.name}
			{:else if user.given_name}
				{user.given_name}
			{:else if user.email}
				{user.email}
			{/if}
		{:else}
			Account
		{/if}
	</span>
	{#if user?.picture}
		<img class="profile-picture" src={user.picture} alt="Avatar" />
	{:else}
		<div class="anon-icon">
			<Icon name="profile" />
		</div>
	{/if}
</button>

<!-- REMOVE IF IT'S NOT NEEDED -->

<style lang="postcss">
	.btn {
		display: flex;
		align-items: center;
		overflow-x: hidden;
		gap: var(--space-8);

		height: var(--size-btn-l);
		padding: var(--space-6) var(--space-8);
		border-radius: var(--radius-m);

		color: var(--clr-theme-scale-ntrl-50);
		transition:
			background-color var(--transition-fast),
			color var(--transition-fast),
			filter var(--transition-fast);

		&.pop {
			color: var(--clr-theme-scale-pop-10);
			background: color-mix(
				in srgb,
				var(--clr-theme-scale-pop-80) 70%,
				var(--clr-theme-scale-ntrl-100)
			);

			&:hover {
				color: var(--clr-theme-scale-pop-10);
				background: color-mix(
					in srgb,
					var(--clr-theme-scale-pop-80) 40%,
					var(--clr-theme-scale-ntrl-100)
				);
			}
		}

		&:hover {
			background-color: color-mix(in srgb, transparent, var(--darken-light));
			color: var(--clr-theme-scale-ntrl-40);
		}
	}
	.name {
		white-space: nowrap;
		text-overflow: ellipsis;
		overflow-x: hidden;
	}
	.anon-icon,
	.profile-picture {
		border-radius: var(--radius-m);
		width: var(--space-20);
		height: var(--space-20);
	}
	.anon-icon {
		display: flex;
		align-items: center;
		justify-content: center;
		padding: var(--space-2);
		background: var(--clr-theme-pop-element);
		color: var(--clr-theme-pop-on-element);
	}
</style>
