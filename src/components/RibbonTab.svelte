<script>
    import { getContext } from "svelte";
    import { C_RIBBON } from "./ribbon.mjs";

    export let label;

    let ribbon  = getContext(C_RIBBON);
    let tabId   = ribbon.addTab(label);
    let openTab = ribbon.openTab;
</script>
<div style:display={$openTab == tabId ? "" : "none"}>
    <slot></slot>
</div>