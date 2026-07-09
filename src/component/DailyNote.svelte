<script lang="ts">
    import type DailyNoteViewPlugin from "../dailyNoteViewIndex";
    import { MarkdownView, TAbstractFile, TFile, WorkspaceLeaf, moment, App } from "obsidian";
    import { spawnLeafView } from "../leafView";
    import { getDailyNoteSettings, DEFAULT_DAILY_NOTE_FORMAT } from "obsidian-daily-notes-interface";
    import { onDestroy, onMount } from "svelte";

    /**
     * Check whether any other file in the vault links to the given file.
     * Uses the metadata cache's resolvedLinks map for an O(n) scan.
     */
    function fileHasBacklinks(app: App, file: TFile): boolean {
        const resolvedLinks = app.metadataCache.resolvedLinks;
        for (const sourcePath in resolvedLinks) {
            if (sourcePath === file.path) continue;
            if (file.path in resolvedLinks[sourcePath]) {
                return true;
            }
        }
        return false;
    }

    export let file: TAbstractFile;
    export let plugin: DailyNoteViewPlugin;
    export let leaf: WorkspaceLeaf;
    export let shouldRender: boolean = true;
    export let autoFocus: boolean = false;

    let editorEl: HTMLElement;
    let containerEl: HTMLElement;
    let title: string;
    let rawTitle: string;

    let rendered: boolean = false;

    let createdLeaf: WorkspaceLeaf;

    // Track if this component is being destroyed
    let isDestroying = false;

    // Track if the note is collapsed
    let isCollapsed: boolean = false;

    onMount(() => {
        if (file instanceof TFile) {
            const basename = file.basename;
            rawTitle = basename;
            const dateFormat = getDailyNoteSettings().format || DEFAULT_DAILY_NOTE_FORMAT;
            const parsed = moment(basename, dateFormat, true);
            title = parsed.isValid() ? parsed.format("dddd, MMMM Do YYYY") : basename;
        }
    });

    // Load editor when shouldRender becomes true; once loaded, never unload
    $: if (editorEl && shouldRender && !rendered) {
        showEditor();
    }

    onDestroy(() => {
        isDestroying = true;
        if (rendered && createdLeaf) {
            try {
                if (createdLeaf.detach) createdLeaf.detach();
                if (editorEl) editorEl.empty();
            } catch (e) {
                console.error("Error cleaning up editor:", e);
            }
        }
    });

    function showEditor() {
        if (!(file instanceof TFile)) return;
        if (rendered) return;
        if (isDestroying) return;

        try {
            [createdLeaf] = spawnLeafView(plugin, editorEl, leaf);
            createdLeaf.setPinned(true);

            // Only show backlinks if the setting is enabled AND the file actually has backlinks
            const showBacklinks = !plugin.settings.hideBacklinks && fileHasBacklinks(plugin.app, file);

            createdLeaf.setViewState({
                type: "markdown",
                state: {
                    file: file.path,
                    mode: "source",
                    source: false,
                    backlinks: showBacklinks,
                    backlinkOpts: {
                        collapseAll: false,
                        extraContext: false,
                        sortOrder: "alphabetical",
                        showSearch: false,
                        searchQuery: "",
                        backlinkCollapsed: false,
                        unlinkedCollapsed: true
                    }
                }
            });
            createdLeaf.parentLeaf = leaf;

            rendered = true;

            // Handle autoFocus after editor has had time to render
            if (autoFocus) {
                window.setTimeout(() => {
                    if (createdLeaf?.view instanceof MarkdownView) {
                        const editor = createdLeaf.view.editor;
                        editor.focus();
                        editor.setCursor(editor.lineCount(), 0);
                    }
                }, 400);
            }
        } catch (error) {
            console.error("Error creating leaf view:", error);
        }
    }

    function handleFileIconClick() {
        if (!(file instanceof TFile)) return;
        if (leaf && !(leaf as any)?.pinned) {
            leaf.openFile(file);
        } else plugin.app.workspace.getLeaf(false).openFile(file);
    }

    function handleEditorClick() {
        // @ts-ignore
        const editor = createdLeaf?.view?.editMode?.editor;
        if (editor && !editor.hasFocus()) {
            editor.focus();
        }
    }

    // Toggle collapse/expand state
    function toggleCollapse() {
        isCollapsed = !isCollapsed;
    }
</script>

<div class="daily-note-container" data-id='dn-editor-{file.path}' bind:this={containerEl}>
    <div class="daily-note">
        {#if title}
            <div class="daily-note-title inline-title">
                <!-- Collapse/Expand button -->
                <!-- svelte-ignore a11y-interactive-supports-focus -->
                <!-- svelte-ignore a11y-click-events-have-key-events -->
                <span role="button" data-collapsed={isCollapsed} class="collapse-button" on:click={toggleCollapse} title={isCollapsed ? "Expand" : "Collapse"}>
                    <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-chevron-down"><path d="m6 9 6 6 6-6"/></svg>
                </span>
                <!-- svelte-ignore a11y-interactive-supports-focus -->
                <!-- svelte-ignore a11y-click-events-have-key-events -->
                <span role="link" class="clickable-link" on:click={handleFileIconClick} data-title={title}>{title}</span>



            </div>
        {/if}
        <div class="daily-note-editor" data-collapsed={isCollapsed} aria-hidden="true" bind:this={editorEl} data-title={rawTitle} on:click={handleEditorClick}>
            {#if !rendered && shouldRender}
                <div class="editor-placeholder">Loading...</div>
            {/if}
            {#if !shouldRender && !rendered}
                <div class="editor-placeholder">Scroll to view content</div>
            {/if}
        </div>
    </div>
</div>

<style>
    .daily-note {
        margin-bottom: var(--size-4-5);
        padding-bottom: var(--size-4-8);
    }

    .daily-note:has(.daily-note-editor[data-collapsed="true"]) {
        margin-bottom: 0;
        padding-bottom: 0;
    }

    .daily-note-editor {
        min-height: 100px;
    }

    .daily-note-editor[data-collapsed="true"] {
        display: none;
    }

    .daily-note .collapse-button {
        display: none;
    }

    .daily-note:hover .collapse-button {
        display: block;
    }

    .daily-note .collapse-button {
        color: var(--text-muted);
    }

    .daily-note .collapse-button:hover  {
        color: var(--text-normal);
    }

    .daily-note:has(.is-readable-line-width) .daily-note-title {
        max-width: calc(var(--file-line-width) + var(--size-4-4));
        width: calc(var(--file-line-width) + var(--size-4-4));
        margin-left: auto;
        margin-right: auto;
        margin-bottom: var(--size-4-8);
        display: flex;
        align-items: center;
        justify-content: start;

        gap: var(--size-4-2);
    }

    .collapse-button {
        margin-left: calc(var(--size-4-8) * -1);
    }

    .collapse-button[data-collapsed="true"] {
        transform: rotate(-90deg);

        transition: transform 0.2s ease;
    }

    .daily-note:not(:has(.is-readable-line-width)) .daily-note-title {
        display: flex;
        justify-content: start;
        align-items: center;
        width: 100%;
        padding-left: calc(calc(100% - var(--file-line-width)) / 2 - var(--size-4-2));
        padding-right: calc(calc(100% - var(--file-line-width)) / 2 - var(--size-4-2));
        margin-top: var(--size-4-8);

        gap: var(--size-4-2);
    }

    .clickable-link {
        cursor: pointer;
        text-decoration: none;
    }

    .clickable-link:hover {
        color: var(--color-accent);
        text-decoration: underline;
    }

    .editor-placeholder {
        display: flex;
        justify-content: center;
        align-items: center;
        height: 100px;
        color: var(--text-muted);
        font-style: italic;
    }

    .collapse-button {
        cursor: pointer;
        display: flex;
        align-items: center;
        justify-content: center;
        width: 24px;
        height: 24px;
        border-radius: 4px;
        color: var(--text-muted);
        transition: background-color 0.2s ease;
    }

    .collapse-button:hover {
        /* background-color: var(--background-modifier-hover); */
        color: var(--text-normal);
    }
</style>
