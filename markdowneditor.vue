<!-- eslint-disable vue/multi-word-component-names -->

<template>

    <div class="markdown-editor-component">
        <!-- Modal -->
        <div v-show="showToolboxMenu" class="modal custom-modal-menu" ref="toolboxMenu" :style="toolboxMenuStyle">
            <div class="list-group">
                <a class="list-group-item list-group-item-action text-muted" href="#"><i
                        class="fa-solid fa-hammer fa-fw"></i>Tool</a>
                <a class="list-group-item list-group-item-action text-muted" href="#"><i
                        class="fa-solid fa-hammer fa-fw"></i>Another Tool</a>
                <a class="list-group-item list-group-item-action text-muted" href="#"><i
                        class="fa-solid fa-hammer fa-fw"></i>Some Other Tool</a>
                <a class="list-group-item list-group-item-action text-muted" href="#"><i
                        class="fa-solid fa-hammer fa-fw"></i>That Tool</a>
                <a class="list-group-item list-group-item-action text-muted" href="#"><i
                        class="fa-solid fa-hammer fa-fw"></i>This Tool</a>
            </div>
        </div>

        <!-- Editor -->
        <div class="markdown-editor" ref="markdownEditor" :style="editorStyle">
            <!-- Nav tabs -->
            <ul class="nav nav-tabs">
                <li class="nav-item">
                    <a :class="['nav-link', { active: activeTab === 'write' }]" @click.prevent="switchTab('write')"
                        href="#" role="tab" aria-selected="true">Write</a>
                </li>
                <li class="nav-item">
                    <a :class="['nav-link', { active: activeTab === 'preview' }]" @click.prevent="switchTab('preview')"
                        href="#" role="tab" aria-selected="false">Preview</a>
                </li>
                <li class="nav-item flex-grow-1">
                    <div class="icon-buttons float-end" v-show="activeTab === 'write'">
                        <!-- Icon-only buttons -->
                        <template v-for="(toolbarItemGroup, groupIndex) in toolbarConfiguration" :key="groupIndex">
                            <template v-for="toolbarItem in toolbarItemGroup.ToolbarItems" :key="toolbarItem.command">
                                <a class="btn btn-light btn-sm" @click="toolbarItemCommand(toolbarItem.Command, $event)"
                                    :title="toolbarItem.Name">
                                    <i :class="toolbarItem.Icon"></i>
                                </a>
                            </template>
                            <div class="separator" v-if="groupIndex < toolbarConfiguration.length - 1"></div>
                        </template>
                    </div>
                </li>
            </ul>

            <!-- Tab panes -->
            <div class="editor-content">
                <div class="tab-content">
                    <!-- Write tab -->
                    <div :class="{ 'show active': activeTab === 'write', 'tab-pane': true }" role="tabpanel">
                        <textarea class="editor-textarea" v-model="textareaContent" ref="markdownTextArea"
                            placeholder="Use Markdown to format your comment"></textarea>
                    </div>
                    <!-- Preview tab -->
                    <div :class="{ 'show active': activeTab === 'preview', 'tab-pane': true }" role="tabpanel">
                        <div class="editor-preview" v-html="previewContent"></div>
                    </div>
                </div>
            </div>
        </div>

    </div>
</template>

<script lang="ts" setup>

import { ref, watch, onBeforeUnmount, onMounted, nextTick } from 'vue';

const markdownEditor = ref<HTMLElement | null>(null);
const markdownTextArea = ref<HTMLTextAreaElement | null>(null);
const initialMinimumHeight = 150;

const toolboxMenu = ref<HTMLElement | null>(null);
const showToolboxMenu = ref<boolean>(false);
const toolboxMenuStyle = ref({});

const spinnerHtml = `<div class="spinner">Loading...</div>`;

const activeTab = ref<string>("write");
const editorStyle = ref({});
const editorHeight = ref<number>(initialMinimumHeight);

const textareaContent = ref<string>("");
const previewContent = ref<string>("");

const props = defineProps({
    initialContent: {
        type: String,
        required: true
    }
});

/*
* Handle the toolbox menu
*/

const toggleShowToolboxMenu = async (toolboxButton: HTMLElement, event: MouseEvent) => {

    event.stopPropagation();
    showToolboxMenu.value = !showToolboxMenu.value;

    if (showToolboxMenu.value && toolboxMenu.value) {

        document.addEventListener('click', hideToolboxMenuClickHandler);

        toolboxMenuStyle.value = {
            display: 'block'
        };

        await nextTick();

        const buttonRect = toolboxButton.getBoundingClientRect();
        const modalWidth = toolboxMenu.value.offsetWidth;

        toolboxMenuStyle.value = {
            display: 'block',
            top: `${buttonRect.bottom}px`,
            left: `${buttonRect.right - modalWidth}px`,
        };

    }

};

const hideToolboxMenuClickHandler = (event: MouseEvent) => {

    const toolbox = toolboxMenu.value;

    if (toolbox && !toolbox.contains(event.target as Node)) {

        showToolboxMenu.value = false;
        toolboxMenuStyle.value = {};

        console.log('hideToolboxMenuClickHandler');

        document.removeEventListener('click', hideToolboxMenuClickHandler);
    }
};

/*
* Handle switching between tabs and the events it triggers
*/

const switchTab = (tabName: string) => {
    activeTab.value = tabName;
};

watch(activeTab, async (newTab) => {

    if (newTab === 'preview') {
        // Call to update the preview HTML
        await getMarkdownPreview();

        // Update the editor style
        editorStyle.value = {
            resize: 'unset'
        };
    }
    else {

        // Update the editor style
        editorStyle.value = {
            height: editorHeight.value + 'px',
            resize: 'vertical'
        };

    }

});

/*
* Handle running the markdown preview
*/

const getMarkdownPreview = async () => {

    // Short-circuit the preview handler if there's nothing to preview
    if (!textareaContent.value) {
        previewContent.value = "Nothing to preview";
        return Promise.resolve("guard-stop");
    }

    // Before fetching the html, set a loading spinner
    previewContent.value = spinnerHtml;

    // Get the html from the markdown-to-html handler
    const response = await fetch('/MarkdownPreviewHandler/BuildPreview', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({ markdown: textareaContent.value }),
    });

    // 
    const data = await response.json();
    previewContent.value = data;

    return Promise.resolve("markdown-handler-response");
};


/*
* Handle all the mouse events and movements
*/

onBeforeUnmount(() => {
    if (markdownEditor.value) {
        markdownEditor.value.removeEventListener('mousedown', handleMouseDown);
    }
    window.removeEventListener('mousemove', handleMouseMove);
    window.removeEventListener('mouseup', handleMouseUp);
});

onMounted(() => {
    if (markdownEditor.value) {
        markdownEditor.value?.addEventListener('mousedown', handleMouseDown);
    }

    textareaContent.value = props.initialContent;
});

const handleMouseMove = (e: MouseEvent) => {
    if (markdownEditor.value) {
        const newHeight = e.clientY - markdownEditor.value.getBoundingClientRect().top;
        editorHeight.value = newHeight >= initialMinimumHeight ? newHeight : initialMinimumHeight;
        editorStyle.value = { height: `${editorHeight.value}px` };
    }
};

const handleMouseUp = () => {
    window.removeEventListener('mousemove', handleMouseMove);
    window.removeEventListener('mouseup', handleMouseUp);
};

const handleMouseDown = (e: MouseEvent) => {
    if (markdownEditor.value) {
        const rect = markdownEditor.value.getBoundingClientRect();
        const distanceFromBottom = rect.bottom - e.clientY;

        if (distanceFromBottom <= 10) {
            window.addEventListener('mousemove', handleMouseMove);
            window.addEventListener('mouseup', handleMouseUp);
        }
    }
};

/*
* Setup the toolbar items
*/

class toolbarItem {
    "Id": number;
    "Name": string;
    "Icon": string;
    "Command": string;

    constructor(
        id: number,
        name: string,
        icon: string,
        command: string,
    ) {
        this.Id = id;
        this.Name = name;
        this.Icon = icon;
        this.Command = command;
    }
}

class toolbarItemGroup {
    "ToolbarItems": toolbarItem[]

    constructor(toolbarItems: toolbarItem[]) {
        this.ToolbarItems = toolbarItems;
    }
}

const toolbarConfiguration = ref<toolbarItemGroup[]>([
    new toolbarItemGroup([
        new toolbarItem(1, 'Heading', 'fa-solid fa-heading', 'heading'),
        new toolbarItem(2, 'Bold', 'fa-solid fa-bold', 'bold'),
        new toolbarItem(3, 'Italic', 'fa-solid fa-italic', 'italic'),
    ]),
    new toolbarItemGroup([
        new toolbarItem(1, 'Indent', 'fa-solid fa-indent', 'indent'),
        new toolbarItem(1, 'Code', 'fa-solid fa-code', 'code'),
        new toolbarItem(1, 'Link', 'fa-solid fa-link', 'link'),
    ]),
    new toolbarItemGroup([
        new toolbarItem(1, 'Unordered List', 'fa-solid fa-list-ul', 'ul'),
        new toolbarItem(1, 'Ordered List', 'fa-solid fa-list-ol', 'ol'),
        new toolbarItem(1, 'Tasks', 'fa-solid fa-list-check', 'tasks'),
    ]),
    new toolbarItemGroup([
        new toolbarItem(1, 'Tag User', 'fa-solid fa-at', 'tag'),
        new toolbarItem(1, 'Mention', 'fa-regular fa-message', 'mention'),
        new toolbarItem(1, 'Back', 'fa-solid fa-arrow-left-long', 'unk'),
    ]),
    new toolbarItemGroup([
        new toolbarItem(1, 'Toolbox', 'fa-solid fa-screwdriver-wrench', 'tool'),
    ])
]);

const toolbarItemCommand = (command: string, event: MouseEvent) => {

    const buttonElement = event.currentTarget as HTMLElement;

    const start = markdownTextArea.value?.selectionStart ?? 0;
    const end = markdownTextArea.value?.selectionEnd ?? 0;
    const selectedText = markdownTextArea.value?.value.substring(start, end);

    let replacement = selectedText ?? '';

    switch (command) {
        case 'bold':
            replacement = `**${selectedText}**`;
            break;
        case 'italic':
            replacement = `*${selectedText}*`;
            break;
        case 'heading':
            replacement = `### ${selectedText}`;
            break;
        case 'tool':
            toggleShowToolboxMenu(buttonElement, event);
            break;
        default:
            console.log(`Unknown command: ${command}`);
            return;
    }

    // Replace the selected text with the formatted text
    textareaContent.value = replacement ?? "";
    markdownTextArea.value?.setRangeText(replacement, start, end, 'end');
    markdownTextArea.value?.focus();
};


</script>

<style scoped lang="scss">
$background-light: #f6f8fa;
$color-light-gray: #9e9e9e;
$color-gray: #757575;
$color-border: #ced4da;
$background-white: #fff;
$font-size-small: 0.8rem;
$color-dark: #24292e;
$color-header-gray: #eaecef;
$color-muted-gray: #6a737d;
$color-link: #0366d6;
$color-blockquote-border: #dfe2e5;
$color-code-background: rgba(27, 31, 35, 0.05);
$color-pre-background: #f6f8fa;
$color-table-border: #dfe2e5;
$color-hr-background: #9e9e9e;
$color-alert-border-note: #0969da;
$color-alert-border-tip: #1a7f37;
$color-alert-border-important: #8250df;
$color-alert-border-warning: #9a6700;
$color-alert-border-caution: #d1242f;

.nav.nav-tabs {
    position: relative;
    left: -1px;
    top: -1px;
    width: calc(100% + 1px);
    padding-right: 2px;
    background-color: $background-light;
}

a.nav-link {
    font-size: $font-size-small;

    &:not(.active) {
        color: $color-light-gray;

        &:hover {
            border-color: transparent !important;
        }
    }
}

li.nav-item.ml-auto {
    padding: 2px;
}

.icon-buttons {
    float: right !important;
    position: relative;
    top: 3px;

    a {
        color: $color-gray;

        &:focus {
            outline: none;
            box-shadow: none;
        }
    }

    .separator {
        display: inline-block;
        width: 1px;
        background-color: $color-border;
        height: 1rem;
        position: relative;
        top: 4px;
    }
}

.markdown-editor {
    border-radius: 5px;
    border: 1px solid $color-border;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Noto Sans',
        Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji';
    background-color: $background-white;
    resize: vertical;
    overflow: hidden;
    display: flex;
    flex-direction: column;
    min-height: 200px;
}

.editor-content,
.tab-content,
.tab-pane {
    flex: 1;
    display: flex;
    flex-direction: column;
    margin: 0;
    padding: 0;
    overflow: hidden;
}

.tab-content>.active {
    display: contents !important;
}

.editor-preview {
    all: unset;
}

.editor-textarea,
.editor-preview {
    background-color: $background-white;
    flex: 1;
    width: 100%;
    margin: 0;
    padding: 8px;
    border: none;
    outline: none;
    resize: none;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Noto Sans',
        Helvetica, Arial, sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji';
    font-size: 14px;
    overflow: auto;
}

.editor-textarea {
    padding: 20px;

    & img {
        max-width: 100%;
    }
}

.editor-preview {
    padding: 20px;
    width: unset;
}

/* Custom Modal As Menu */
.custom-modal-menu {
    position: absolute;
    display: none;
    z-index: 1050;
    min-width: 200px;
    width: unset;
    height: unset;
    font-size: $font-size-small;

    .fa-fw {
        position: relative;
        left: -5px;
    }
}

.custom-modal-menu[style*='display: block'] {
    display: block;
}

/* Preview Styles */
::v-deep .editor-preview {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial,
        sans-serif, 'Apple Color Emoji', 'Segoe UI Emoji';
    color: $color-dark;
    line-height: 1.5;
    background-color: $background-white;
    padding: 20px;

    h1,
    h2,
    h3,
    h4,
    h5,
    h6 {
        font-weight: 600;
        line-height: 1.25;
    }

    h1 {
        font-size: 2em;
        padding-bottom: 0.3em;
        border-bottom: 1px solid $color-header-gray;
    }

    h2 {
        font-size: 1.5em;
        padding-bottom: 0.3em;
        border-bottom: 1px solid $color-header-gray;
    }

    h3 {
        font-size: 1.25em;
    }

    h4 {
        font-size: 1em;
    }

    h5 {
        font-size: 0.875em;
    }

    h6 {
        font-size: 0.85em;
        color: $color-muted-gray;
    }

    p {
        margin-top: 0;
        margin-bottom: 1em;
    }

    a {
        color: $color-link;
        text-decoration: none;

        &:hover {
            text-decoration: underline;
        }
    }

    ul,
    ol {
        padding-left: 2em;
        margin-top: 0;
        margin-bottom: 1em;

        li+li {
            margin-top: 0.25em;
        }
    }

    blockquote {
        padding: 0.5em 1em;
        margin-bottom: 1em;
        color: $color-muted-gray;
        border-left: 0.25em solid $color-blockquote-border;
    }

    code {
        font-family: Consolas, 'Liberation Mono', Menlo, Courier, monospace;
        font-size: 85%;
        background-color: $color-code-background;
        padding: 0.2em 0.4em;
        border-radius: 3px;
    }

    pre {
        font-family: Consolas, 'Liberation Mono', Menlo, Courier, monospace;
        font-size: 85%;
        background-color: $color-pre-background;
        padding: 1em;
        overflow: auto;
        border-radius: 10px;
        line-height: 1.45;

        code {
            background-color: transparent;
            padding: 0;
        }
    }

    table {
        display: table;
        border-spacing: 0;
        border-collapse: collapse;
        margin-bottom: 1em;
        width: 100%;

        th {
            padding: 0.6em;
            background-color: $background-light;
            border: 1px solid $color-table-border;
        }

        td {
            padding: 0.6em;
            border: 1px solid $color-table-border;
        }
    }

    img {
        max-width: 100%;
    }

    hr {
        height: 0.25em;
        padding: 0;
        margin: 1.5em 0;
        background-color: $color-hr-background;
        border: 0;
    }

    strong {
        font-weight: 600;
    }

    em {
        font-style: italic;
    }

    .task-list-item {
        list-style-type: none;

        input[type='checkbox'] {
            margin: 0 0.2em 0.25em -1.6em;
            vertical-align: middle;
        }
    }

    .footnotes {
        font-size: 0.85em;
        padding-top: 1em;

        ol {
            padding-left: 1.4em;
        }

        li {
            margin-bottom: 0.5em;
        }
    }

    details {
        margin-bottom: 1em;

        summary {
            font-weight: 600;
            cursor: pointer;
        }
    }

    .markdown-alert {
        padding: 8px 16px;
        margin-bottom: 16px;
        color: inherit;
        border-left: 0.25em solid $color-alert-border-note;

        p {
            margin-bottom: 0px;
        }

        &-title {
            display: flex;
            font-weight: 500;
            align-items: center;
            line-height: 1;
            margin-bottom: 1em;
        }

        &.markdown-alert-note {
            border-left-color: $color-alert-border-note;

            &-title {
                color: $color-alert-border-note;
            }
        }

        &.markdown-alert-tip {
            border-left-color: $color-alert-border-tip;

            &-title {
                color: $color-alert-border-tip;
            }
        }

        &.markdown-alert-important {
            border-left-color: $color-alert-border-important;

            &-title {
                color: $color-alert-border-important;
            }
        }

        &.markdown-alert-warning {
            border-left-color: $color-alert-border-warning;

            &-title {
                color: $color-alert-border-warning;
            }
        }

        &.markdown-alert-caution {
            border-left-color: $color-alert-border-caution;

            &-title {
                color: $color-alert-border-caution;
            }
        }

        svg {
            display: inline-block;
            overflow: visible !important;
            vertical-align: text-bottom;
            fill: currentColor;
            margin-right: 10px;
        }
    }
}
</style>
