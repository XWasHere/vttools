<script>
    import { setContext } from "svelte";
    import { writable } from "svelte/store";
    import { C_RIBBON } from "./ribbon.mjs";

    let tabs    = [];
    let openTab = writable(0);

    setContext(C_RIBBON, {
        addTab(name) {
            return tabs.push(name) - 1;
        },

        get openTab() { return openTab; }
    });
</script>
<div class="root">
    <div>
        <slot></slot>
    </div>
    <div class="ribbon-tabs">
        {#each tabs as t, idx}
            <button class:ribbon-tab-active = {$openTab == idx}
                    on:click={function() {
                        $openTab = idx;
                    }}>
                {t}
            </button>
        {/each}
        <div class="ribbon-tab-spacer"></div>
    </div>
</div>
<style>
    .root {
        display: flex;

        flex-direction: column-reverse;

        --ribbon-tabbtn-active:   #ffffff;
        --ribbon-tabbtn-inactive: #ffffff;
    }

    .ribbon-tabs {
        display: flex;
    }

    .ribbon-tabs>button {
        border:        1px solid var(--ribbon-tabbtn-inactive);
        border-bottom: 1px solid;
        border-radius: 0;

        padding: 2px 5px;

        margin: 0;

        background-color: var(--ribbon-tabbtn-inactive);
    }

    .ribbon-tabs>button.ribbon-tab-active {
        border:        1px solid;
        border-bottom: 1px solid var(--ribbon-tabbtn-active);

        background-color: var(--ribbon-tabbtn-active);
    }

    .ribbon-tab-spacer {
        flex-grow: 1;

        border-bottom: 1px solid;
    }
</style>