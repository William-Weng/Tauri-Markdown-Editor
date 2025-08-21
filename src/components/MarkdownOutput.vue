<script setup lang="ts">
import { ref, watch, onMounted } from 'vue';

const props = defineProps({
  renderedMarkdown: { type: String, required: true },
  isDarkTheme: { type: Boolean, required: true },
  fontSize: { type: Number, default: 1.0 },
});

const outputIframe = ref<HTMLIFrameElement | null>(null);

// MARK: - Functions
/**
 * 更新 iframe 內容
 * @description 將渲染後的 Markdown 內容寫入 iframe 中，並根據主題和字體大小設置樣式。
 */
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

/**
 * [選擇主題 CSS 連結](https://cdn.jsdelivr.net/npm/github-markdown-css@5.8.1/github-markdown-dark.css)
 * @param isDarkTheme [是否為深色主題](https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github.min.css)
 * @returns 主題 CSS 連結
 */
function selectThemeCssLink(isDarkTheme: boolean): string {
  return isDarkTheme ? '/assets/github-markdown-dark.css' : '/assets/github-markdown-light.css';
}

/**
 * [選擇 Highlight.js 主題 CSS 連結](https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github.min.css)
 * @param isDarkTheme [是否為深色主題](https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github-dark.min.css)
 * @returns Highlight.js 主題 CSS 連結
 */
function selectHighlightJsThemeLink(isDarkTheme: boolean): string {
  return isDarkTheme ? '/assets/github-dark.min.css' : '/assets/github.min.css';
}

// MARK: - 生命週期
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
