<script lang="ts">
    import { page } from '$app/state';
    import type { Writable } from 'svelte/store';
    import { preferences } from '$lib/stores/preferences';
    import { onMount, type Snippet } from 'svelte';
    import type { Column } from '$lib/helpers/types';
    import { ActionMenu, Layout, Popover, Selector } from '@appwrite.io/pink-svelte';

    let {
        columns,
        isCustomCollection = false,
        allowNoColumns = false,
        children
    }: {
        columns: Writable<Column[]>;
        isCustomCollection?: boolean;
        allowNoColumns?: boolean;
        children: Snippet<[toggle: () => void, selectedColumnsNumber: number]>;
    } = $props();

    let maxHeight = $state('none');
    let containerRef;

    const calcMaxHeight = () => {
        if (containerRef) {
            // get parent row element for correct top position
            const parent = containerRef.parentElement.parentElement;
            const { top } = parent.getBoundingClientRect();

            maxHeight = `${window.innerHeight - top - 48}px`;
        }
    };

    onMount(async () => {
        if (isCustomCollection) {
            const prefs = preferences.getCustomCollectionColumns(page.params.collection);
            columns.set(
                $columns.map((column) => {
                    column.hide = prefs?.includes(column.id) ?? false;
                    return column;
                })
            );
        } else {
            const prefs = preferences.get(page.route);

            // Override the shown columns only if a preference was set
            if (prefs?.columns) {
                columns.set(
                    $columns.map((column) => {
                        column.hide = prefs.columns?.includes(column.id) ?? false;
                        return column;
                    })
                );
            }
        }

        columns.subscribe((ctx) => {
            const columns = ctx.filter((n) => n.hide === true).map((n) => n.id);

            if (isCustomCollection) {
                preferences.setCustomCollectionColumns(columns);
            } else {
                preferences.setColumns(columns);
            }
        });

        calcMaxHeight();
    });

    let selectedColumnsNumber = $derived(
        $columns.reduce((acc, column) => {
            if (column.hide === true) return acc;

            return ++acc;
        }, 0)
    );
</script>

<svelte:window on:resize={calcMaxHeight} />
{#if $columns?.length}
    <Popover let:toggle placement="bottom-end" padding="none">
        {@render children(toggle, selectedColumnsNumber)}
        <svelte:fragment slot="tooltip">
            <div style:max-height={maxHeight} style:overflow="scroll" bind:this={containerRef}>
                <ActionMenu.Root>
                    {#each $columns as column}
                        {#if !column?.exclude}
                            <ActionMenu.Item.Button
                                on:click={() => (column.hide = !column.hide)}
                                disabled={allowNoColumns
                                    ? false
                                    : selectedColumnsNumber <= 1 && column.hide !== true}>
                                <Layout.Stack direction="row" gap="s">
                                    <Selector.Checkbox
                                        checked={!column.hide}
                                        size="s"
                                        on:click={() => (column.hide = !column.hide)} />
                                    {column.title}
                                </Layout.Stack>
                            </ActionMenu.Item.Button>
                        {/if}
                    {/each}
                </ActionMenu.Root>
            </div>
        </svelte:fragment>
    </Popover>
{/if}
