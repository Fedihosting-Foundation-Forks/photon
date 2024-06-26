<script lang="ts">
  import { getClient } from '$lib/lemmy'
  import type { CommentView, PostView } from 'lemmy-js-client'
  import { amMod, ban, isAdmin, remove } from './moderation'
  import {
    Fire,
    Icon,
    LockClosed,
    LockOpen,
    Megaphone,
    ShieldExclamation,
    Trash,
  } from 'svelte-hero-icons'
  import { isCommentView, isPostView } from '$lib/lemmy/item.js'
  import { Menu, MenuButton, MenuDivider, toast } from 'mono-svelte'
  import { profile } from '$lib/auth.js'
  import { Button } from 'mono-svelte'
  import { t } from '$lib/translations'
  import ShieldIcon from './ShieldIcon.svelte'

  export let item: PostView | CommentView

  let locking = false
  let pinning = false

  $: acting = locking || pinning

  async function lock(lock: boolean) {
    if (!$profile?.jwt || !isPostView(item)) return
    locking = true

    try {
      await getClient().lockPost({
        locked: lock,
        post_id: item.post.id,
      })

      item.post.locked = lock
    } catch (err) {
      toast({
        content: err as any,
        type: 'error',
      })
    }

    locking = false
  }

  async function pin(pinned: boolean, toInstance: boolean = false) {
    if (!$profile?.jwt || !isPostView(item)) return

    pinning = true

    try {
      await getClient().featurePost({
        feature_type: toInstance ? 'Local' : 'Community',
        featured: pinned,
        post_id: item.post.id,
      })

      item.post.featured_community = pinned
    } catch (err) {
      toast({
        content: err as any,
        type: 'error',
      })
    }

    pinning = false
  }
</script>

<Menu placement="bottom-end">
  <Button
    class="hover:!text-green-500"
    slot="target"
    size="square-md"
    color="ghost"
    loading={acting}
    {...$$restProps}
  >
    <ShieldIcon filled width={16} />
  </Button>
  {#if ($profile?.user && amMod($profile.user, item.community)) || ($profile?.user && isAdmin($profile.user))}
    <MenuDivider>
      {#if !item.community.local && !amMod($profile.user, item.community)}
        {$t('moderation.labelInstanceOnly')}
      {:else}
        {$t('moderation.label')}
      {/if}
    </MenuDivider>
    <MenuButton
      color="warning-subtle"
      on:click={() => lock(!item.post.locked)}
      loading={locking}
      disabled={locking}
    >
      <Icon
        src={item.post.locked ? LockOpen : LockClosed}
        size="16"
        mini
        slot="prefix"
      />
      {item.post.locked ? $t('moderation.unlock') : $t('moderation.lock')}
    </MenuButton>

    <MenuButton
      color="success-subtle"
      on:click={() =>
        pin(isPostView(item) ? !item.post.featured_community : false)}
      loading={pinning}
      disabled={pinning}
    >
      <Icon src={Megaphone} size="16" mini />
      <div
        class="flex flex-row gap-2 text-left items-center justify-between w-full"
      >
        <span>
          {item.post.featured_community
            ? $t('moderation.unfeature')
            : $t('moderation.feature')}
        </span>
        {#if isAdmin($profile.user)}
          <span class="text-xs opacity-80">{$t('form.post.community')}</span>
        {/if}
      </div>
    </MenuButton>
    <MenuButton color="danger-subtle" on:click={() => remove(item)}>
      <Icon src={Trash} size="16" mini />
      {#if isCommentView(item)}
        {item.comment.removed
          ? $t('moderation.restore')
          : $t('moderation.remove')}
      {:else}
        {item.post.removed ? $t('moderation.restore') : $t('moderation.remove')}
      {/if}
    </MenuButton>
    {#if $profile?.user && $profile.user.local_user_view.person.id != item.creator.id}
      <MenuButton
        color="danger-subtle"
        on:click={() =>
          ban(item.creator_banned_from_community, item.creator, item.community)}
      >
        <Icon src={ShieldExclamation} size="16" mini />
        {item.creator_banned_from_community
          ? $t('moderation.ban.unbanFromCommunity')
          : $t('moderation.ban.banFromCommunity')}
      </MenuButton>
    {/if}
  {/if}
  {#if $profile?.user && isAdmin($profile.user)}
    <MenuDivider>{$t('admin.label')}</MenuDivider>
    <MenuButton
      color="success-subtle"
      on:click={() =>
        pin(isPostView(item) ? !item.post.featured_local : false, true)}
    >
      <Icon src={Megaphone} size="16" mini />
      <div
        class="flex flex-row gap-2 text-left items-center justify-between w-full"
      >
        <span>
          {item.post.featured_local
            ? $t('moderation.unfeature')
            : $t('moderation.feature')}
        </span>
        <span class="text-xs opacity-80">{$t('admin.instance')}</span>
      </div>
    </MenuButton>
    <MenuButton color="danger-subtle" on:click={() => remove(item, true)}>
      <Icon src={Fire} size="16" mini />
      {$t('admin.purge')}
    </MenuButton>
  {/if}
</Menu>
