<script lang="ts">
  import Header from '$lib/components/ui/layout/pages/Header.svelte'
  import {
    DEFAULT_INSTANCE_URL,
    LINKED_INSTANCE_URL,
    instance as currentInstance,
  } from '$lib/instance.js'
  import { getClient, validateInstance } from '$lib/lemmy.js'
  import { t } from '$lib/translations'
  import { Button, TextInput, toast } from 'mono-svelte'

  let instance = LINKED_INSTANCE_URL || $currentInstance || ''
  let email = ''
  let loading = false

  async function submit() {
    loading = true
    try {
      if (!(await validateInstance(instance)))
        throw new Error($t('toast.failInstanceURL'))

      await getClient(instance).passwordReset({
        email: email,
      })

      toast({
        content: $t('toast.resetLink'),
        type: 'success',
      })
    } catch (err) {
      toast({
        content: err as any,
        type: 'error',
      })
    }
    loading = false
  }
</script>

<div class="my-auto max-w-xl mx-auto flex flex-col gap-2">
  <Header>{$t('routes.resetLogin.title')}</Header>
  <p>
    {$t('routes.resetLogin.description')}
  </p>
  <form class="mt-2 flex flex-col gap-4" on:submit|preventDefault={submit}>
    {#if !LINKED_INSTANCE_URL}
      <TextInput
        bind:value={instance}
        label={$t('form.instance')}
        placeholder={DEFAULT_INSTANCE_URL}
        required
      />
    {/if}
    <TextInput
      bind:value={email}
      label={$t('form.email')}
      type="email"
      required
      placeholder="example@example.com"
    />
    <Button color="primary" size="lg" {loading} disabled={loading} submit>
      {$t('form.submit')}
    </Button>
  </form>
</div>
