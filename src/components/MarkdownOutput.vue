<script setup lang="ts">
import { ref, watch, onMounted } from 'vue';

const props = defineProps({
  renderedMarkdown: {
    type: String,
    required: true,
  },
  isDarkTheme: {
    type: Boolean,
    required: true,
  },
  fontSize: {
    type: Number,
    default: 1.0,
  },
});

const outputIframe = ref<HTMLIFrameElement | null>(null);

function updateIframeContent() {
  if (outputIframe.value && outputIframe.value.contentDocument) {
    const doc = outputIframe.value.contentDocument;
    doc.open();
    const themeCssLink = props.isDarkTheme
      ? 'https://cdn.jsdelivr.net/npm/github-markdown-css@5.8.1/github-markdown-dark.css'
      : 'https://cdn.jsdelivr.net/npm/github-markdown-css@5.8.1/github-markdown-light.css';
    const highlightJsThemeLink = props.isDarkTheme
      ? 'https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github-dark.min.css'
      : 'https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github.min.css'; // Light theme

    doc.writeln(`
      <!DOCTYPE html>
      <html>
        <head>
          <link rel="stylesheet" href="${themeCssLink}">
          <link rel="stylesheet" href="${highlightJsThemeLink}">
          <style>
            .markdown-body {
              padding: 1rem;
              font-size: ${props.fontSize}em;
            }
          </style>
        </head>
        <body class="markdown-body">
          <main class="container">
            ${props.renderedMarkdown}
          </main>
        </body>
      </html>
    `);
    doc.close();
  }
}

watch(() => props.renderedMarkdown, () => {
  updateIframeContent();
}, { immediate: true });

watch(() => props.isDarkTheme, () => {
  updateIframeContent();
});

watch(() => props.fontSize, (_: number) => {
  updateIframeContent();
});

onMounted(() => {
  updateIframeContent();
});
</script>

<template>
  <iframe ref="outputIframe" class="markdown-output"></iframe>
</template>

<style scoped>
.markdown-output {
  height: 100%;
  width: 100%;
  border: 1px solid #444;
  border-radius: 8px;
}
</style>
