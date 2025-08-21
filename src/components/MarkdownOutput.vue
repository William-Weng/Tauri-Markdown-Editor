<script setup lang="ts">
import { ref, watch, onMounted } from 'vue';
import { save } from '@tauri-apps/plugin-dialog';
import { writeTextFile } from '@tauri-apps/plugin-fs';

defineExpose({ exportHtmlFile });

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

  const doc = outputIframe.value.contentDocument;
  const content = htmlContent(true, props.isDarkTheme);

  doc.open();
  doc.writeln(content);
  doc.close();
}

/**
 * 根據主題和字體大小生成完整的 HTML 字符串，包括必要的 CSS 樣式和渲染後的 Markdown 內容
 * @param isExport 是否為匯出模式
 * @param isDarkTheme 是否為暗黑主題
 * @returns 返回完整的 HTML 字符串
 */
function htmlContent(isExport: boolean, isDarkTheme: boolean): string {

  const mainTheme = selectMainTheme(isExport, isDarkTheme);
  const heightTheme = selectHeightTheme(isExport, isDarkTheme);

return `
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>Exported Markdown</title>
    <link rel="stylesheet" href="${mainTheme}">
    <link rel="stylesheet" href="${heightTheme}">
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
`.trim();
}

/**
 * 根據主題選擇 CSS 樣式表
 * @param isExport 是否為匯出模式
 * @param isDarkTheme 是否為暗黑主題
 * @returns 返回 CSS 樣式表的路徑
 */
function selectMainTheme(isExport: boolean, isDarkTheme: boolean): string {
  if (!isExport) { return isDarkTheme ? '/assets/github-markdown-dark.css' : '/assets/github-markdown-light.css'; }
  return isDarkTheme ? 'https://cdn.jsdelivr.net/npm/github-markdown-css@5.8.1/github-markdown-dark.css' : 'https://cdn.jsdelivr.net/npm/github-markdown-css@5.8.1/github-markdown-light.css';
}

/**
 * 根據主題選擇高亮樣式表
 * @param isExport 是否為匯出模式
 * @param isDarkTheme 是否為暗黑主題
 * @returns 返回高亮樣式表的路徑
 */
function selectHeightTheme(isExport: boolean, isDarkTheme: boolean): string {
  if (!isExport) { return isDarkTheme ? '/assets/github-dark.min.css' : '/assets/github.min.css'; }
  return isDarkTheme ? 'https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github-dark.min.css' : 'https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github.min.css';
}

/**
 * 匯出渲染後的 HTML 檔案
 */
async function exportHtmlFile() {

  try {
    const filePath = await save({ filters: [{ name: 'HTML Files', extensions: ['html'] }] });
    if (!filePath) return;
    await writeTextFile(filePath, htmlContent(true, props.isDarkTheme));
    return 'HTML exported successfully!';
  } catch (error) {
    return error instanceof Error ? error.message : 'Error exporting HTML!';
  }
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
