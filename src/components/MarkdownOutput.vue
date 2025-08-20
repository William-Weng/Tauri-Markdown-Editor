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

  if (!outputIframe.value) return;
  if (!outputIframe.value.contentWindow) { console.warn('Output iframe is not ready yet.'); return; }
  if (!outputIframe.value.contentDocument) { return; }

  const themeCssLink = selectThemeCssLink(props.isDarkTheme);  
  const highlightJsThemeLink = selectHighlightJsThemeLink(props.isDarkTheme);
  const doc = outputIframe.value.contentDocument;

  doc.open();
  doc.writeln(`
    <!DOCTYPE html>
    <html>
      <head>
        <link rel="stylesheet" href="${themeCssLink}">
        <link rel="stylesheet" href="${highlightJsThemeLink}">
        <style>
          .markdown-body { padding: 1rem; font-size: ${props.fontSize}em; }
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

function selectThemeCssLink(isDarkTheme: boolean): string {
  return isDarkTheme
    ? 'https://cdn.jsdelivr.net/npm/github-markdown-css@5.8.1/github-markdown-dark.css'
    : 'https://cdn.jsdelivr.net/npm/github-markdown-css@5.8.1/github-markdown-light.css';
}
function selectHighlightJsThemeLink(isDarkTheme: boolean): string {
  return isDarkTheme
    ? 'https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github-dark.min.css'
    : 'https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github.min.css'; // Light theme
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
