<script lang="ts">
    import { A, type AProps } from '$comp/typography';
    import { cn } from '$lib/utils';
    import IconFilter from '~icons/mdi/filter';

    import { StringFilter } from './filters.svelte';

    type Props = {
        changed: (filter: StringFilter) => void;
        term: string;
        value?: null | string;
    } & AProps;
    let { changed, children, class: className, term, value, ...props }: Props = $props();

    const title = `Search ${term}:${value}`;
</script>

<A class={cn('cursor-pointer', className)} onclick={() => changed(new StringFilter(term, value ?? undefined))} {title} {...props}>
    {#if children}
        {@render children()}
    {:else}
        <IconFilter class="text-muted-foreground text-opacity-50 hover:text-primary" />
    {/if}
</A>
