<script lang="ts">
	import { enhance } from '$app/forms';
	import { invalidate } from '$app/navigation';
	import { page } from '$app/stores';
	import { save_repl } from '$lib/api/client/repls';
	import { commands, on_command } from '$lib/command_runner/commands';
	import AsyncButton from '$lib/components/AsyncButton.svelte';
	import Avatar from '$lib/components/Avatar.svelte';
	import DropdownMenu from '$lib/components/DropdownMenu.svelte';
	import Logo from '$lib/components/Logo.svelte';
	import MenuItem from '$lib/components/MenuItem.svelte';
	import { stringify } from '$lib/components/parsers';
	import { PUBLIC_SAVE_IN_LOCAL_STORAGE_NAME } from '$lib/constants';
	import { ICON } from '$lib/icons';
	import { share_with_hash, share_with_id } from '$lib/share';
	import { command_runner } from '$lib/stores/command_runner_store';
	import { layout_store } from '$lib/stores/layout_store';
	import { is_repl_saving, is_repl_to_save, repl_id, repl_name } from '$lib/stores/repl_id_store';
	import { get_theme } from '$lib/theme';
	import { webcontainer } from '$lib/webcontainer';
	import { onMount } from 'svelte';
	import SearchDocsIcon from '~icons/sveltelab/svelte-search';
	import MenuBar from './MenuBar.svelte';
	// TODO: dedupe header and profile header (use slots for specific buttons?)

	const theme = get_theme();
	$: ({ user, github_login, owner_id, REDIRECT_URI } = $page.data ?? {});
	export let mobile = false;
	let forking = false;

	onMount(() => {
		return on_command('fork', async () => {
			forking = true;
			await save_repl(true);
			forking = false;
		});
	});

	let a_hover = false;

	const save_repl_then_navigate = async ({ detail: e }: { detail: MouseEvent }) => {
		e.preventDefault();
		try {
			// save current project to local storage (we have to use local instead
			// of session because Firefox Mobile it's being weird)
			window.localStorage.setItem(
				PUBLIC_SAVE_IN_LOCAL_STORAGE_NAME,
				stringify(await webcontainer.get_tree_from_container(true))
			);
		} catch (e) {
			if (!window.confirm('You will lose progress on this project...do you want to continue?')) {
				// this will prevent the actual navigation
				throw new Error('');
			}
		}
		window.location.assign((e.target as HTMLAnchorElement).href);
	};
</script>

<header>
	<a
		href="/"
		title="New REPL"
		class="logo"
		on:mouseenter={() => (a_hover = true)}
		on:mouseleave={() => (a_hover = false)}
	>
		{#if !a_hover}
			<Logo />
		{:else}
			<ICON.AddNew />
		{/if}
	</a>
	<div class="no-mobile menu-bar">
		<MenuBar commands={$commands} />
	</div>
	<div class="grow">
		<button
			class="search-docs"
			on:click={() => {
				command_runner.open('search-kit-docs');
			}}
			title="Search sveltekit documentation"
			><SearchDocsIcon /> Search SvelteKit Docs...
		</button>
	</div>
	{#if !mobile}
		<button
			class="supplemental"
			title="Toggle File Browser"
			on:click={layout_store.toggle_file_tree}
			aria-pressed={$layout_store.file_tree !== 0}
		>
			<ICON.FileBrowser />
		</button>

		<button
			class="supplemental"
			title="Toggle Terminal"
			on:click={layout_store.toggle_terminal}
			aria-pressed={$layout_store.terminal !== 0}
		>
			<ICON.Terminal />
		</button>
	{/if}
	<button
		class="supplemental"
		on:click={(e) => {
			if (e.shiftKey) {
				theme.remove_preference();
			} else {
				theme.change_preference();
			}
		}}
		title="Change theme. Shift+Click to delete preference"
	>
		{#if $theme.next === 'light'}
			<ICON.LightMode />
		{:else}
			<ICON.DarkMode />
		{/if}
	</button>
	<a class="no-mobile" href="http://docs.sveltelab.dev/" target="_blank" title="SvelteLab Docs">
		<ICON.Docs />
	</a>
	<button
		class="supplemental"
		on:click={() => {
			command_runner.open('> ');
		}}
		title="Open Command Runner (CTRL+K)"
	>
		<ICON.Cmd />
	</button>

	{#if user}
		<!-- Fork button -->
		{#if $repl_id}
			<AsyncButton
				click={async () => {
					if (window.confirm(`Are you sure you want to fork "${$repl_name}"`)) {
						await save_repl(true);
					}
				}}
				title="Fork Project"
				loading={forking}
			>
				<ICON.Fork />
			</AsyncButton>
		{/if}
		<!-- Save button -->
		{#if !owner_id || user.id === owner_id}
			<AsyncButton
				badged={$is_repl_to_save}
				click={async () => {
					await save_repl();
				}}
				title="Save Changes"
				loading={$is_repl_saving}
			>
				<ICON.Save />
			</AsyncButton>
		{/if}
	{/if}
	<!-- Share button or dropdown -->
	{#if !$repl_id}
		<button
			on:click={async () => {
				share_with_hash();
			}}
			title="Share Files Snapshot"
		>
			<ICON.Share />
		</button>
	{:else}
		<DropdownMenu indicator>
			<svelte:fragment slot="trigger">
				<ICON.Share />
			</svelte:fragment>
			<MenuItem
				on:click={() => {
					share_with_id();
				}}
			>
				<ICON.Url /> Share Project
			</MenuItem>
			<MenuItem
				on:click={() => {
					share_with_hash();
				}}
				><ICON.Hash /> Share Code Snapshot
			</MenuItem>
		</DropdownMenu>
	{/if}
	{#if user}
		<!-- Profile or login -->
		<DropdownMenu indicator>
			<svelte:fragment slot="trigger">
				{#if user.avatarUrl}
					<Avatar alt={`${user.name} profile`} src={`./proxy/?url=${user.avatarUrl}`} />
				{:else}
					<ICON.Profile />
				{/if}
			</svelte:fragment>
			<MenuItem href="/profile"><ICON.Profile /> Your profile</MenuItem>
			<form
				use:enhance={() => {
					return () => {
						//on logout we invalidate authed:user which reload the page
						invalidate('authed:user');
					};
				}}
				method="POST"
				action="?/logout"
			>
				<MenuItem>
					<ICON.SignOut /> Log out
				</MenuItem>
			</form>
		</DropdownMenu>
	{:else}
		<DropdownMenu indicator>
			<svelte:fragment slot="trigger">
				<ICON.Login />
			</svelte:fragment>
			{#if github_login}
				<MenuItem
					on:click={save_repl_then_navigate}
					href={`${github_login?.authUrl}${REDIRECT_URI}${$page.url.pathname}`}
				>
					<ICON.Github /> Login with GitHub
				</MenuItem>
			{/if}
			<MenuItem href="/login" on:click={save_repl_then_navigate}>
				<ICON.Login />
				Login with Password
			</MenuItem>
		</DropdownMenu>
	{/if}
</header>

<style>
	header {
		--padding-y: 0.25em;
		padding: var(--padding-y) 1em;
		display: flex;
		gap: 1em;
		background-color: var(--sk-back-2);
		position: relative;
		z-index: 2;
		align-items: center;
		--shadow-height: 0.5rem;
		--shadow-gradient: linear-gradient(
			to bottom,
			rgba(0, 0, 0, 0.1) 0%,
			rgba(0, 0, 0, 0.05) 30%,
			transparent 100%
		);
	}

	header:after {
		content: '';
		position: absolute;
		width: 100%;
		height: var(--shadow-height);
		left: 0;
		bottom: calc(-1 * var(--shadow-height));
		background: var(--shadow-gradient);
	}

	header > :global(:not(.grow)) {
		flex-shrink: 0;
	}

	.grow {
		width: 100%;
	}

	.search-docs {
		border-radius: 99999rem;
		margin: auto;
		max-width: 24rem;
		font-size: 1.1rem;
		color: var(--sk-text-2);
		border: 1px solid var(--sk-back-5);
		width: 100%;
		justify-content: center;
		opacity: 0.8;
		padding-block: 0.3rem;
	}

	.search-docs:hover {
		border-color: var(--sk-theme-1);
		color: var(--sk-text-1);
	}

	header :global(a),
	header :global(button) {
		gap: 0.5rem;
		display: flex;
		align-items: center;
		position: relative;
		padding: 0.5rem;
		color: var(--sk-text-1);
	}

	header > :global(a):hover,
	header > :global(button):hover {
		color: var(--sk-theme-1);
	}

	header :global(a) :global(svg),
	header :global(button) :global(svg) {
		font-size: 1.25em;
	}

	button[aria-pressed='true']::after {
		content: '';
		position: absolute;
		background-color: var(--sk-theme-1);
		right: 1px;
		left: 1px;
		bottom: 0;
		top: calc(100% - 3px);
	}

	.logo {
		scale: 1.5;
	}

	.menu-bar {
		display: flex;
		gap: 0.5rem;
	}

	.menu-bar > :global(button) {
		border-radius: 0.5rem;
		padding: 0.5rem 1rem;
	}

	.menu-bar > :global(button:hover) {
		background-color: var(--sk-theme-1);
		color: white;
	}

	@media only screen and (max-width: 940px) {
		.search-docs {
			display: none;
		}
	}

	@media only screen and (max-width: 800px) and (min-width: 500px) {
		.supplemental {
			display: none;
		}
	}

	@media only screen and (max-width: 500px) {
		.no-mobile {
			display: none;
		}
		header {
			gap: 0.75rem;
		}
	}
</style>
