<script lang="ts">
    import type { EventSummaryModel, SummaryTemplateKeys } from '$features/events/components/summary/index';

    import * as DataTable from '$comp/data-table';
    import DelayedRender from '$comp/delayed-render.svelte';
    import ErrorMessage from '$comp/error-message.svelte';
    import * as FacetedFilter from '$comp/faceted-filter';
    import { Button } from '$comp/ui/button';
    import * as Card from '$comp/ui/card';
    import * as Sheet from '$comp/ui/sheet';
    import EventsDrawer from '$features/events/components/events-drawer.svelte';
    import { shouldRefreshPersistentEventChanged } from '$features/events/components/filters/helpers';
    import { filterChanged, filterRemoved, toFilter } from '$features/events/components/filters/models.svelte';
    import OrganizationDefaultsFacetedFilterBuilder from '$features/events/components/filters/organization-defaults-faceted-filter-builder.svelte';
    import { getTableContext } from '$features/events/components/table/options.svelte';
    import { ChangeType, type WebSocketMessageValue } from '$features/websockets/models';
    import { useFetchClientStatus } from '$shared/api/api.svelte';
    import { isTableEmpty, removeTableData } from '$shared/table';
    import { type FetchClientResponse, useFetchClient } from '@exceptionless/fetchclient';
    import { createTable } from '@tanstack/svelte-table';
    import ExternalLink from 'lucide-svelte/icons/external-link';
    import { useEventListener } from 'runed';
    import { queryParameters, ssp } from 'sveltekit-search-params';
    import { debounce } from 'throttle-debounce';

    let selectedEventId: null | string = $state(null);
    function rowclick(row: EventSummaryModel<SummaryTemplateKeys>) {
        selectedEventId = row.id;
    }

    // TODO: Default filters
    const params = queryParameters({ filter: ssp.string(), limit: ssp.number(10) });
    let filters = $state<FacetedFilter.IFilter[]>([]);
    const filter = $derived(toFilter(filters));

    function onDrawerFilterChanged(added: FacetedFilter.IFilter): void {
        filters = filterChanged(filters, added);
        params.filter = filter;
        selectedEventId = null;
    }

    function onFilterChanged(addedOrUpdated: FacetedFilter.IFilter): void {
        if (addedOrUpdated.type !== 'date') {
            filters = filterChanged(filters, addedOrUpdated);
            params.filter = filter;
        }
    }

    function onFilterRemoved(removed?: FacetedFilter.IFilter): void {
        filters = filterRemoved(filters, removed);
        params.filter = filter;
    }

    const context = getTableContext<EventSummaryModel<SummaryTemplateKeys>>({ limit: params.limit, mode: 'summary' }, function (options) {
        options.columns = options.columns.filter((c) => c.id !== 'select').map((c) => ({ ...c, enableSorting: false }));
        options.enableMultiRowSelection = false;
        options.enableRowSelection = false;
        options.manualSorting = false;
        return options;
    });

    const table = createTable(context.options);

    const client = useFetchClient();
    const clientStatus = useFetchClientStatus(client);

    let response = $state<FetchClientResponse<EventSummaryModel<SummaryTemplateKeys>[]>>();
    let before: string | undefined;

    async function loadData(filterChanged: boolean = false) {
        if (client.isLoading && filterChanged && !before) {
            return;
        }

        if (filterChanged) {
            before = undefined;
        }

        response = await client.getJSON<EventSummaryModel<SummaryTemplateKeys>[]>('events', {
            params: {
                ...context.parameters,
                before,
                filter
            }
        });

        if (response.ok) {
            if (response.meta.links.previous?.before) {
                before = response.meta.links.previous?.before;
            }

            const data = filterChanged ? [] : context.data;
            for (const summary of response.data?.reverse() || []) {
                data.push(summary);
            }

            context.data = data.slice(-params.limit);
            context.meta = response.meta;
        }
    }

    const debouncedLoadData = debounce(5000, loadData);
    async function onPersistentEventChanged(message: WebSocketMessageValue<'PersistentEventChanged'>) {
        if (message.id && message.change_type === ChangeType.Removed) {
            if (removeTableData(table, (doc) => doc.id === message.id)) {
                // If the grid data is empty from all events being removed, we should refresh the data.
                if (isTableEmpty(table)) {
                    await debouncedLoadData();
                    return;
                }
            }
        }

        // Do not refresh if the filter criteria doesn't match the web socket message.
        if (!shouldRefreshPersistentEventChanged(filters, filter, message.organization_id, message.project_id, message.stack_id, message.id)) {
            return;
        }

        await debouncedLoadData();
    }

    useEventListener(document, 'refresh', async () => await loadData());
    useEventListener(document, 'PersistentEventChanged', (event) => onPersistentEventChanged((event as CustomEvent).detail));

    $effect(() => {
        loadData();
    });
</script>

<Card.Root>
    <Card.Title class="p-6 pb-0 text-2xl" level={2}>Event Stream</Card.Title>
    <Card.Content>
        <DataTable.Root>
            <DataTable.Toolbar {table}>
                <FacetedFilter.Root changed={onFilterChanged} {filters} remove={onFilterRemoved}>
                    <OrganizationDefaultsFacetedFilterBuilder includeDateFacets={false} />
                </FacetedFilter.Root>
            </DataTable.Toolbar>
            <DataTable.Body rowClick={rowclick} {table}>
                {#if clientStatus.isLoading}
                    <DelayedRender>
                        <DataTable.Loading {table} />
                    </DelayedRender>
                {:else}
                    <DataTable.Empty {table} />
                {/if}
            </DataTable.Body>
            <DataTable.Footer {table}>
                <div class="flex w-full items-center justify-center space-x-4">
                    <DataTable.PageSize bind:value={params.limit} {table} />
                    <div class="text-center">
                        <ErrorMessage message={response?.problem?.errors.general} />
                    </div>
                </div>
            </DataTable.Footer>
        </DataTable.Root>
    </Card.Content></Card.Root
>

<Sheet.Root onOpenChange={() => (selectedEventId = null)} open={!!selectedEventId}>
    <Sheet.Content class="w-full overflow-y-auto sm:max-w-full md:w-5/6">
        <Sheet.Header>
            <Sheet.Title
                >Event Details <Button href="/event/{selectedEventId}" size="sm" title="Open in new window" variant="ghost"><ExternalLink /></Button
                ></Sheet.Title
            >
        </Sheet.Header>
        <EventsDrawer changed={onDrawerFilterChanged} id={selectedEventId || ''} close={() => (selectedEventId = null)}></EventsDrawer>
    </Sheet.Content>
</Sheet.Root>
