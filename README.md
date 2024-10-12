# vue-github-markdown-editor
Github Markdown Editor Clone as a Vue Component.

> [!IMPORTANT]  
> You will need to implement your own server-side Markdown to HTML render process to hook `getMarkdownPreview` into.

## Usage:
```vue
<template>

    <markdowneditor :initialContent="initialContent" />

</template>

<script setup lang="ts">

import markdowneditor from '@/components/markdowneditor.vue';

const initialContent = 'Some content updated';

</script>
```

![image](https://github.com/user-attachments/assets/476043e4-1a6f-41ad-8fa4-6cfa7b15e2fd)
