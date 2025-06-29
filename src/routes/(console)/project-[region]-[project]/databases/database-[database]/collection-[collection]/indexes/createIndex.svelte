<script lang="ts">
    import { goto, invalidate } from '$app/navigation';
    import { base } from '$app/paths';
    import { page } from '$app/state';
    import { Submit, trackError, trackEvent } from '$lib/actions/analytics';
    import { Modal } from '$lib/components';
    import { Dependencies } from '$lib/constants';
    import { Button, InputNumber, InputSelect, InputText } from '$lib/elements/forms';
    import { remove } from '$lib/helpers/array';
    import { addNotification } from '$lib/stores/notifications';
    import { sdk } from '$lib/stores/sdk';
    import { IndexType } from '@appwrite.io/console';
    import { isRelationship } from '../document-[document]/attributes/store';
    import { type Attributes, collection, indexes } from '../store';
    import { Icon, Layout } from '@appwrite.io/pink-svelte';
    import { IconPlus } from '@appwrite.io/pink-icons-svelte';
    import { flags } from '$lib/flags';

    export let showCreateIndex = false;
    export let externalAttribute: Attributes = null;

    const databaseId = page.params.database;

    let key = '';
    let error: string;
    let types = [
        { value: IndexType.Key, label: 'Key' },
        { value: IndexType.Unique, label: 'Unique' },
        { value: IndexType.Fulltext, label: 'FullText' }
    ];
    let selectedType = IndexType.Key;

    let attributeOptions = $collection.attributes
        .filter((attribute) => !isRelationship(attribute))
        .map((attribute) => ({
            value: attribute.key,
            label: attribute.key
        }));

    let attributeList = [{ value: '', order: '', length: null }];

    const showLengths = flags.showIndexLengths(page.data);

    function generateIndexKey() {
        let indexKeys = $indexes.map((index) => index.key);

        let highestIndex = indexKeys.reduce((max, key) => {
            const match = key.match(/^index_(\d+)$/);
            return match ? Math.max(max, parseInt(match[1], 10)) : max;
        }, indexKeys.length);

        return `index_${highestIndex + 1}`;
    }

    function initialize() {
        attributeList = externalAttribute
            ? [{ value: externalAttribute.key, order: 'ASC', length: null }]
            : [{ value: '', order: 'ASC', length: null }];
        selectedType = IndexType.Key;
        key = `index_${$indexes.length + 1}`;
    }

    $: if (showCreateIndex) {
        error = null;
        initialize();
        key = generateIndexKey();
    }

    $: addAttributeDisabled = !attributeList.at(-1)?.value || !attributeList.at(-1)?.order;

    async function create() {
        if (!(key && selectedType && !addAttributeDisabled)) {
            error = 'All fields are required';
            return;
        }

        try {
            await sdk.forProject(page.params.region, page.params.project).databases.createIndex(
                databaseId,
                $collection.$id,
                key,
                selectedType,
                attributeList.map((a) => a.value),
                attributeList.map((a) => a.order),
                attributeList.map((a) => (a.length ? Number(a.length) : null))
            );

            await Promise.allSettled([
                invalidate(Dependencies.COLLECTION),
                invalidate(Dependencies.DATABASE)
            ]);

            goto(
                `${base}/project-${page.params.region}-${page.params.project}/databases/database-${databaseId}/collection-${$collection.$id}/indexes`
            );

            addNotification({
                message: 'Creating index',
                type: 'success'
            });
            trackEvent(Submit.IndexCreate);
            showCreateIndex = false;
        } catch (err) {
            error = err.message;
            trackError(err, Submit.IndexCreate);
        }
    }

    function addAttribute() {
        if (addAttributeDisabled) return;

        // We assign instead of pushing to trigger Svelte's reactivity
        attributeList = [...attributeList, { value: '', order: '', length: null }];
    }
</script>

<Modal title="Create index" bind:error onSubmit={create} bind:show={showCreateIndex}>
    <InputText
        required
        id="key"
        label="Index Key"
        placeholder="Enter Key"
        bind:value={key}
        autofocus />
    <InputSelect required options={types} id="type" label="Index type" bind:value={selectedType} />

    <Layout.Stack gap="s">
        {#each attributeList as attribute, i}
            <Layout.Stack direction="row">
                <InputSelect
                    required
                    options={[
                        { value: '$id', label: '$id' },
                        { value: '$createdAt', label: '$createdAt' },
                        { value: '$updatedAt', label: '$updatedAt' },
                        ...attributeOptions
                    ]}
                    id={`attribute-${i}`}
                    label={i === 0 ? 'Attribute' : undefined}
                    placeholder="Select Attribute"
                    bind:value={attribute.value} />

                <InputSelect
                    options={[
                        { value: 'ASC', label: 'ASC' },
                        { value: 'DESC', label: 'DESC' }
                    ]}
                    required
                    id={`order-${i}`}
                    label={i === 0 ? 'Order' : undefined}
                    bind:value={attribute.order}
                    placeholder="Select Order" />

                <Layout.Stack direction="row" alignItems="flex-end" gap="xs">
                    {#if selectedType === IndexType.Key && showLengths}
                        <InputNumber
                            id={`length-${i}`}
                            label={i === 0 ? 'Length' : undefined}
                            placeholder="Enter Length"
                            bind:value={attribute.length} />
                    {/if}
                    <Button
                        icon
                        compact
                        disabled={attributeList.length <= 1}
                        on:click={() => {
                            attributeList = remove(attributeList, i);
                        }}>
                        <span class="icon-x" aria-hidden="true"></span>
                    </Button>
                </Layout.Stack>
            </Layout.Stack>
        {/each}
        <div>
            <Button compact on:click={addAttribute} disabled={addAttributeDisabled}>
                <Icon icon={IconPlus} slot="start" size="s" />
                Add attribute
            </Button>
        </div>
    </Layout.Stack>
    <svelte:fragment slot="footer">
        <Button secondary on:click={() => (showCreateIndex = false)}>Cancel</Button>
        <Button submit>Create</Button>
    </svelte:fragment>
</Modal>
