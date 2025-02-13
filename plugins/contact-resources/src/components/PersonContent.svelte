<!--
// Copyright © 2022 Hardcore Engineering Inc.
// 
// Licensed under the Eclipse Public License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License. You may
// obtain a copy of the License at https://www.eclipse.org/legal/epl-2.0
// 
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// 
// See the License for the specific language governing permissions and
// limitations under the License.
-->
<script lang="ts">
  import { Employee, getName, Person } from '@hcengineering/contact'
  import { IntlString } from '@hcengineering/platform'
  import {
    getPlatformAvatarColorDef,
    getPlatformAvatarColorForTextDef,
    IconSize,
    Label,
    LabelAndProps,
    themeStore,
    tooltip
  } from '@hcengineering/ui'
  import { DocNavLink } from '@hcengineering/view-resources'
  import { createEventDispatcher, onMount } from 'svelte'
  import Avatar from './Avatar.svelte'
  import { getClient } from '@hcengineering/presentation'

  export let value: Person | Employee | undefined | null
  export let inline: boolean = false
  export let disabled = false
  export let shouldShowAvatar: boolean = true
  export let shouldShowName = true
  export let element: HTMLElement | undefined = undefined
  export let shouldShowPlaceholder = false
  export let noUnderline: boolean = false
  export let defaultName: IntlString | undefined = undefined
  export let statusLabel: IntlString | undefined = undefined
  export let avatarSize: IconSize = 'x-small'
  export let onEdit: ((event: MouseEvent) => void) | undefined = undefined
  export let showTooltip: LabelAndProps | undefined = undefined
  export let enlargedText = false
  export let colorInherit: boolean = false
  export let accent: boolean = false
  export let maxWidth = ''

  const client = getClient()

  const onEditClick = (evt: MouseEvent) => {
    if (!disabled) {
      onEdit?.(evt)
    }
  }
  const dispatch = createEventDispatcher()

  $: accentColor =
    value?.name !== undefined
      ? getPlatformAvatarColorForTextDef(value?.name ?? '', $themeStore.dark)
      : getPlatformAvatarColorDef(0, $themeStore.dark)

  $: dispatch('accent-color', accentColor)

  onMount(() => {
    dispatch('accent-color', accentColor)
  })
</script>

{#if value}
  <DocNavLink object={value} onClick={onEdit} {disabled} {noUnderline} {inline} {colorInherit} {accent} noOverflow>
    {#if inline}
      <span class="antiMention" use:tooltip={disabled ? undefined : showTooltip}>
        @{getName(client.getHierarchy(), value)}
      </span>
    {:else}
      <span
        use:tooltip={disabled ? undefined : showTooltip}
        class="contentPresenter"
        class:text-base={enlargedText}
        style:max-width={maxWidth}
      >
        {#if shouldShowAvatar}
          <span
            class="eContentPresenterIcon"
            class:mr-2={shouldShowName && !enlargedText}
            class:mr-3={shouldShowName && enlargedText}
          >
            <Avatar size={avatarSize} avatar={value.avatar} />
          </span>
        {/if}
        {#if shouldShowName}
          <span class="eContentPresenterLabel overflow-label" class:colorInherit class:fs-bold={accent}
            >{getName(client.getHierarchy(), value)}</span
          >
        {/if}
      </span>
    {/if}
  </DocNavLink>
  {#if statusLabel}
    <span class="status">
      <Label label={statusLabel} />
    </span>
  {/if}
{:else if shouldShowPlaceholder}
  <!-- svelte-ignore a11y-click-events-have-key-events -->
  <span
    class="contentPresenter"
    class:text-base={enlargedText}
    use:tooltip={disabled ? undefined : showTooltip}
    on:click={onEditClick}
  >
    {#if !inline && shouldShowAvatar}
      <span
        class="eContentPresenterIcon"
        class:mr-2={shouldShowName && !enlargedText}
        class:mr-3={shouldShowName && enlargedText}
      >
        <Avatar size={avatarSize} on:accent-color />
      </span>
    {/if}
    {#if shouldShowName && defaultName}
      <span class="eContentPresenterLabel" class:colorInherit class:fs-bold={accent}>
        <Label label={defaultName} />
      </span>
    {/if}
  </span>
{/if}

<style lang="scss">
  .contentPresenter {
    display: flex;
    align-items: center;
    flex-wrap: nowrap;
    min-width: 0;

    .eContentPresenterIcon {
      color: var(--theme-dark-color);
    }
    .eContentPresenterLabel {
      min-width: 0;
      text-align: left;
      color: var(--theme-caption-color);

      // overflow: hidden;
      // display: -webkit-box;
      // /* autoprefixer: ignore next */
      // -webkit-box-orient: vertical;
      // -webkit-line-clamp: 2;
      // line-clamp: 2;
      // user-select: none;

      &.colorInherit {
        color: inherit;
      }
    }
    &:hover {
      .eContentPresenterIcon {
        color: var(--theme-caption-color);
      }
      .eContentPresenterLabel {
        color: var(--theme-caption-color);
      }
    }
  }
  .status {
    margin-left: 0.25rem;
    padding: 0.125rem 0.25rem;
    font-size: 0.75rem;
    color: var(--theme-content-color);
    background-color: var(--theme-button-default);
    border-radius: 0.25rem;
  }
</style>
