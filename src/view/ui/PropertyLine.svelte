<script lang="ts">
    import type { Monster } from "@types";
    import { Notice } from "obsidian";
    import type { PropertyItem } from "src/layouts/types";
    import { stringify } from "src/util/util";
    import TextContentHolder from "./TextContentHolder.svelte";

    export let monster: Monster;
    export let item: PropertyItem;

    let property = stringify(monster[item.properties[0]]);
    let display = item.display ?? item.properties[0];

    if (item.callback) {
        try {
            const frame = document.body.createEl("iframe");
            const funct = (frame.contentWindow as any).Function;
            const func = new funct("monster", item.callback);
            property = func.call(undefined, monster) ?? property;
            document.body.removeChild(frame);
        } catch (e) {
            new Notice(
                `There was an error executing the provided callback for [${item.properties.join(
                    ", "
                )}]\n\n${e.message}`
            );
            console.error(e);
        }
    }
    if (!item.conditioned && !`${property}`.length) {
        property = item.fallback ?? "-";
    }
</script>

{#if !item.conditioned || (item.conditioned && `${property}`.length)}
    <div class="line {item.properties[0]}">
        <span class="property-name">{display}</span>
        <TextContentHolder render={item.markdown} {property} />
    </div>
{/if}

<style>
    .line {
        line-height: var(--statblock-property-line-height);
        display: block;
        color: var(--statblock-property-font-color);
    }
    .property-name {
        margin: 0;
        margin-right: 0.25em;
        display: inline;
        color: var(--statblock-property-name-font-color);
        font-weight: var(--statblock-property-name-font-weight);
    }
</style>
