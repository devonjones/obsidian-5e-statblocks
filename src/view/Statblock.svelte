<script lang="ts">
    import type { Monster } from "@types";
    import {
        debounce,
        ExtraButtonComponent,
        Menu,
        Notice,
        stringifyYaml
    } from "obsidian";
    import { stringify } from "querystring";
    import { EXPORT_SYMBOL, SAVE_SYMBOL } from "src/data/constants";
    import type { StatblockItem } from "src/layouts/types";
    import type StatBlockPlugin from "src/main";
    import {
        createEventDispatcher,
        onDestroy,
        onMount,
        setContext
    } from "svelte";
    import { Writable, writable } from "svelte/store";
    import type StatBlockRenderer from "./statblock";

    import Bar from "./ui/Bar.svelte";
    import Content from "./ui/Content.svelte";

    const dispatch = createEventDispatcher();

    export let monster: Monster;
    export let context: string;
    export let plugin: StatBlockPlugin;
    export let statblock: StatblockItem[];
    export let renderer: StatBlockRenderer;
    export let layout: string;
    export let canSave: boolean = true;

    let maxColumns =
        !isNaN(Number(monster.columns)) && Number(monster.columns) > 0
            ? Number(monster.columns)
            : 2;

    $: monsterColumnWidth = Number(`${monster.columnWidth}`.replace(/\D/g, ""));
    $: columnWidth =
        !isNaN(monsterColumnWidth) && monsterColumnWidth > 0
            ? monsterColumnWidth
            : 400;

    let canExport = monster.export ?? plugin.settings.export;
    let canDice =
        plugin.canUseDiceRoller && (monster.dice ?? plugin.settings.useDice);
    let canRender = monster.render ?? plugin.settings.renderDice;

    setContext<StatBlockPlugin>("plugin", plugin);
    setContext<boolean>("tryToRenderLinks", plugin.settings.tryToRenderLinks);
    setContext<string>("context", context);
    setContext<Monster>("monster", monster);
    setContext<boolean>("dice", canDice);
    setContext<boolean>("render", canRender);
    setContext<StatBlockRenderer>("renderer", renderer);

    const reset = writable<boolean>(false);
    setContext<Writable<boolean>>("reset", reset);

    let container: HTMLElement;
    let columns: number = maxColumns;
    let ready = false;

    const setColumns = () => {
        if (monster.forceColumns) {
            columns = maxColumns;
            observer.disconnect();
            return;
        }
        const width = container.clientWidth;
        columns = Math.min(
            Math.max(Math.floor(width / columnWidth), 1),
            maxColumns
        );
    };
    const onResize = debounce(
        () => {
            setColumns();
            if (!ready) ready = true;
        },
        100,
        false
    );
    const observer = new ResizeObserver(onResize);

    onMount(() => {
        setColumns();
        observer.observe(container);
    });

    onDestroy(() => {
        observer.disconnect();
    });

    const iconsEl = (node: HTMLElement) => {
        new ExtraButtonComponent(node).setIcon("vertical-three-dots");
    };
    const menu = new Menu();
    menu.addItem((item) =>
        item
            .setIcon(SAVE_SYMBOL)
            .setTitle("Save to Bestiary")
            .setDisabled(!canSave)
            .onClick(() => dispatch("save"))
    );
    menu.addItem((item) => {
        item.setTitle("Copy YAML")
            .setIcon("code")
            .onClick(async () => {
                try {
                    await navigator.clipboard.writeText(stringifyYaml(monster));
                    new Notice("Creature YAML copied to clipboard");
                } catch (e) {
                    new Notice(
                        `There was an issue copying the yaml:\n\n${e.message}`
                    );
                }
            });
    });
    if (canExport)
        menu.addItem((item) =>
            item
                .setIcon(EXPORT_SYMBOL)
                .setTitle("Export as PNG")
                .onClick(() => dispatch("export"))
        );
    if (canDice)
        menu.addItem((item) =>
            item
                .setIcon("reset")
                .setTitle("Reset Dice")
                .onClick(() => {
                    reset.set(true);
                    reset.set(false);
                })
        );
    const showMenu = (evt: MouseEvent) => {
        menu.showAtMouseEvent(evt);
    };

    const name =
        monster?.name
            ?.toLowerCase()
            .replace(/[^A-Za-z0-9\s]/g, "")
            .replace(/\s+/g, "-") ?? "no-name";
    const layoutName =
        layout
            .toLowerCase()
            .replace(/[^A-Za-z0-9\s]/g, "")
            .replace(/\s+/g, "-") ?? "no-layout";
    const classes = [name, layoutName].filter((n) => n?.length);
</script>

<div class="container" bind:this={container}>
    {#if ready}
        <div
            class:obsidian-statblock-plugin={true}
            class:statblock={true}
            class={classes.join(" ")}
        >
            {#if monster}
                <Bar />
                {#key columns}
                    <Content
                        {columns}
                        {maxColumns}
                        {statblock}
                        {ready}
                        on:save
                        on:export
                    />
                {/key}
                <Bar />
            {:else}
                <span>Invalid monster.</span>
            {/if}
        </div>
        <!-- {#if icons} -->
        <div class="icons" use:iconsEl on:click={showMenu} />
        <!-- {/if} -->
    {/if}
</div>

<style>
    .container {
        display: flex;
        width: 100%;
        margin: 0.25rem 0;
    }
    .statblock {
        margin: 0 auto;
        position: relative;
    }

    .icons {
        position: absolute;
        right: 0;
    }
</style>
