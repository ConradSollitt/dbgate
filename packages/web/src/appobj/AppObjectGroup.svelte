<script>
  import Link from '../elements/Link.svelte';

  import { plusExpandIcon } from '../icons/expandIcons';

  import FontIcon from '../icons/FontIcon.svelte';

  import AppObjectListItem from './AppObjectListItem.svelte';

  export let group;
  export let groupFunc;
  export let items;
  export let module;
  export let checkedObjectsStore = null;
  export let disableContextMenu = false;

  let isExpanded = true;

  $: filtered = items.filter(x => x.isMatched);
  $: countText = filtered.length < items.length ? `${filtered.length}/${items.length}` : `${items.length}`;

  function handleCheckAll(isChecked) {
    checkedObjectsStore.update(checkList => {
      const res = isChecked
        ? [
            ...checkList,
            ...filtered
              .filter(x => !checkList.find(y => module.extractKey(x.data) == module.extractKey(y)))
              .map(x => x.data),
          ]
        : checkList.filter(x => groupFunc(x) != group);
      return res;
    });
  }
</script>

<div class="group" on:click={() => (isExpanded = !isExpanded)}>
  <span class="expand-icon">
    <FontIcon icon={plusExpandIcon(isExpanded)} />
  </span>

  {group}
  {items && `(${countText})`}
</div>

{#if isExpanded}
  {#if checkedObjectsStore}
    <div class="ml-2">
      <Link onClick={() => handleCheckAll(true)}>Check all</Link>
      |
      <Link onClick={() => handleCheckAll(false)}>Uncheck all</Link>
    </div>
  {/if}

  {#each filtered as item (module.extractKey(item.data))}
    <AppObjectListItem
      {...$$restProps}
      {module}
      data={item.data}
      {checkedObjectsStore}
      on:objectClick
      {disableContextMenu}
    />
  {/each}
{/if}

<style>
  .group {
    user-select: none;
    padding: 5px;
    cursor: pointer;
    white-space: nowrap;
    font-weight: bold;
  }

  .group:hover {
    background-color: var(--theme-bg-hover);
  }
  .expand-icon {
    margin-right: 3px;
  }
</style>
