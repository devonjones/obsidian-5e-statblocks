<script lang="ts">
    import type { Monster } from "@types";
    import type { SavesItem } from "src/layouts/types";
    import { toTitleCase } from "src/util/util";
    import { getContext } from "svelte";
    import TextContentHolder from "./TextContentHolder.svelte";

    export let monster: Monster;
    export let item: SavesItem;

    function getMod(value: number) {
        return `${value > 0 ? "+" : ""}${value}`;
    }

    let arr: any[] = monster[item.properties[0]] as any[];
    if (!Array.isArray(arr)) {
        arr = [];
    }

    const saves = arr
        .map((ability) => {
            if (typeof ability != "object" || ability == null) return null;
            let key = Object.keys(ability)[0];
            if (!key) return null;
            const value = Object.values(ability)[0];
            if (!value || isNaN(Number(value))) return null;
            return `${toTitleCase(key)} ${getMod(value as number)}`;
        })
        .filter((m) => m)
        .join(", ");

    let canDice = getContext<boolean>("dice");
</script>

<div class="info">
    <div class="line">
        <span class="property-name"
            >{item.display ?? toTitleCase(item.properties[0])}</span
        >
        <div class="property-text">
            <TextContentHolder render={item.markdown} property={saves} />
        </div>
    </div>
</div>

<style>
    .line {
        line-height: var(--statblock-saves-line-height);
        display: block;
        color: var(--statblock-font-color);
    }
    .property-name {
        margin: 0;
        margin-right: 0.25em;
        display: inline;
        font-weight: bold;
    }
    .property-text {
        display: inline;
        margin: 0;
    }
</style>
