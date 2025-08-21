<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, ComputedRef } from 'vue';
import { marked } from 'marked';
import { markedHighlight } from 'marked-highlight';
import hljs from 'highlight.js';

import { open, save } from '@tauri-apps/plugin-dialog';
import { readTextFileLines, writeTextFile } from '@tauri-apps/plugin-fs';
import { listen } from "@tauri-apps/api/event";

import MarkdownOutput from './components/MarkdownOutput.vue';

marked.use(markedHighlight({
  langPrefix: 'hljs language-', // highlight.js css expects a language- prefix
  highlight(code: string, lang: string): string {
    const language = hljs.getLanguage(lang) ? lang : 'plaintext';
    return hljs.highlight(code, { language }).value;
  }
}));

const rawMarkdownInput = ref(`# Welcome to Tauri-Markdown-Editor\n\nThis is a simple Markdown editor built with Tauri and Vue.\n\n## Features\n\n- Real-time Markdown rendering\n- File reading and drag-and-drop\n- Basic styling\n\n## Start writing!\n\nGo ahead and edit this text to see the live preview on the right.\n\n## Build\n\`\`\`bash\npnpm install\npnpm run tauri build\n\`\`\``);

const inputFontSize = ref(1.2);
const outputFontSize = ref(1.0);
const isRightPanelVisible = ref(true);
const isLeftPanelVisible = ref(true);
const fileExtensions = ['md', 'txt'];
const errorMessage = ref('');
const isDarkTheme = ref(false);

const renderedMarkdown: ComputedRef<string> = computed(() => {
  return marked.parse(rawMarkdownInput.value, { gfm: true }) as string;
});

const markdownOutputRef = ref();

let defaultPath = 'untitled.md'

// MARK: - async functions
/** 
 * 打開文件選擇對話框並讀取 Markdown 文件
 */
async function readFile() {

  const filePath = await open({
    multiple: false,
    directory: false,
    filters: [{ name: 'Markdown Files', extensions: fileExtensions }],
  });

  displayMarkdown(filePath as string);
}

/**
 * 打開文件保存對話框並保存 Markdown 文件
 */
async function saveFile() {

  try {
    const filePath = await save({ 
      filters: [{ name: 'Markdown Files', extensions: fileExtensions }],
      defaultPath: defaultPath
    });

    if (!filePath) { errorMessage.value = 'No file path selected.'; return; }
    await writeTextFile(filePath, rawMarkdownInput.value);
    errorMessage.value = 'File saved successfully!';
  } catch (error) {
    errorMessage.value = `Error saving file: ${error}`;
    console.error('Error saving file:', error);
  }
}

/** 
 * 根據文件路徑讀取 Markdown 文件並顯示內容
 * @param filePath - 要讀取的文件路徑
 */
async function displayMarkdown(filePath?: string) {

  if (!filePath) { errorMessage.value = ''; return; }

  const fileExtension = filePath.split('.').pop();
  if (!fileExtensions.includes(fileExtension || '')) { errorMessage.value = `Unsupported file type: ${fileExtension}`; return; }

  const lines = await readTextFileLines(filePath);
  let parsed = '';

  for await (const line of lines) {
    
    let newLine = line

    if (line.endsWith('\0')) { newLine = line.slice(0, -1); }
    parsed += newLine + '\n';
  }

  rawMarkdownInput.value = parsed;
}

// MARK: - event handlers
/**
 * 處理鍵盤事件以增減字體大小 (Command / Ctrl + '+' / '=' / '-')
 * @param event - 鍵盤事件
 */
function handleKeyboardEvent(event: KeyboardEvent) {

  if (event.metaKey || event.ctrlKey) {
    switch (event.key) {
      case '=': // Typically the key for '+' without Shift
      case '+': event.preventDefault(); increaseInputFontSize(); increaseOutputFontSize(); break;
      case '-': event.preventDefault(); decreaseInputFontSize(); decreaseOutputFontSize() ;break;
      case 'r': event.preventDefault(); readFile(); break;
      case 's': event.preventDefault(); saveFile(); break;
    }
  }
}

/**
 * 處理文件拖放事件
 */
function handleFileDragDrop() {

  listen('tauri://drag-drop', (event: any) => {
    if (!event?.payload?.paths[0]) { errorMessage.value =''; return };
    defaultPath = event.payload.paths[0];
    displayMarkdown(defaultPath);
  });
}

/**
 * 匯出 HTML 文件
 * @returns 返回匯出結果的訊息
 */
async function handleExportHtml() {
  if (!markdownOutputRef.value) { errorMessage.value = 'Markdown output component is not ready.'; return; }
  errorMessage.value = await markdownOutputRef.value.exportHtmlFile(defaultPath);
}

// MARK: - Functions
/** 
 * 切換輸入面板的顯示狀態 (左) 
 */
function toggleLeftPanel() { 
  isLeftPanelVisible.value = !isLeftPanelVisible.value; 
}

/** 
 * 切換輸出面板的顯示狀態 (右) 
 */
function toggleRightPanel() { 
  isRightPanelVisible.value = !isRightPanelVisible.value; 
}

/** 
 * 切換主題 (暗色/亮色) 
 */
function toggleTheme() {
  isDarkTheme.value = !isDarkTheme.value;
}

/**
 * 增加輸入面板字體大小
 */
function increaseInputFontSize() {
  inputFontSize.value += 0.1;
}

/**
 * 減少輸入面板字體大小 - 最小限制為 0.5em
 */
function decreaseInputFontSize() {
  if (inputFontSize.value > 0.5) { inputFontSize.value -= 0.1; }
}

/**
 * 增加輸出面板字體大小
 */
function increaseOutputFontSize() {
  outputFontSize.value += 0.1;
}

/**
 * 減少輸出面板字體大小 - 最小限制為 0.5em
 */
function decreaseOutputFontSize() {
  if (outputFontSize.value > 0.5) { outputFontSize.value -= 0.1; }
}

// MARK: - 生命週期
onMounted(() => {
  window.addEventListener('keydown', handleKeyboardEvent);
  handleFileDragDrop();
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyboardEvent);
});

</script>

<template>
  <main class="container">
    <div class="panels">
      <div class="panel" :class="{ 'panel-hidden': !isLeftPanelVisible }">
        <div class="panel-header">
          <h2>Input Markdown</h2>
          <div class="button-group">
            <button @click="readFile" title="Read file" class="save-button">Read</button>
            <button @click="saveFile" title="Save file" class="read-button">Save</button>
            <button @click="toggleRightPanel" :title="isRightPanelVisible ? 'Hide Output Panel' : 'Show Output Panel'" class="toggle-panel-button hide-show-button">{{ isRightPanelVisible ? 'Hide' : 'Show' }}</button>
            <button @click="decreaseInputFontSize" title="Decrease font size" class="font-size-button">-</button>
            <button @click="increaseInputFontSize" title="Increase font size" class="font-size-button">+</button>
          </div>
        </div>
        <textarea v-model="rawMarkdownInput" :style="{ fontSize: inputFontSize + 'em' }"></textarea>
        <div v-if="errorMessage" class="error-display">{{ errorMessage }}</div>
      </div>
      <div class="panel" :class="{ 'panel-hidden': !isRightPanelVisible }">
        <div class="panel-header">
          <h2>Rendered Output</h2>
          <div class="button-group">
            <button @click="handleExportHtml" title="Export as HTML" class="save-button">Export HTML</button>
            <button @click="toggleLeftPanel" :title="isLeftPanelVisible ? 'Hide Input Panel' : 'Show Input Panel'" class="toggle-panel-button hide-show-button">{{ isLeftPanelVisible ? 'Hide' : 'Show' }}</button>
            <button @click="decreaseOutputFontSize" title="Decrease output font size" class="font-size-button">-</button>
            <button @click="increaseOutputFontSize" title="Increase output font size" class="font-size-button">+</button>
            <button @click="toggleTheme" title="Toggle Theme" class="toggle-panel-button">{{ isDarkTheme ? 'Light Theme' : 'Dark Theme' }}</button>
          </div>
        </div>
        <MarkdownOutput ref="markdownOutputRef" :renderedMarkdown="renderedMarkdown" :isDarkTheme="isDarkTheme" :fontSize="outputFontSize" />
      </div>
    </div>
  </main>
</template>

<style scoped>
:global(body) {
  margin: 0;
  background-color: #1a1a1a;
  color: #e0e0e0;
}

.container {
  padding: 2rem;
  height: 100vh;
  box-sizing: border-box;
}

h1, h2 {
  color: #f5f5f5;
}

h1 {
  margin-top: 0;
  text-align: center;
}

.panels {
  display: flex;
  height: 100%;
  margin: 0 -1rem; /* Absorb child margins */
}

.panel {
  flex: 1 1 50%;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  transition: all 0.4s ease-in-out;
  margin: 0 1rem; /* Create gap */
}

.panel-hidden {
  flex: 0 0 0;
  min-width: 0;
  opacity: 0;
  padding: 0;
  border: 0;
  margin: 0;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

h2 {
  margin-top: 0;
  font-size: 1.2rem;
  margin-bottom: 0;
}

.button-group {
  display: flex;
  gap: 0.5rem;
}

button {
  padding: 0.5em 1em;
  border: 1px solid #444;
  background-color: #3a3a3a;
  color: #e0e0e0;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: background-color 0.2s, border-color 0.2s;
}

button:hover {
  background-color: #4a4a4a;
  border-color: #666;
}

button:active {
  background-color: #5a5a5a;
}

.read-button {
  background-color: #FF00FF; /* A standard brown */
  color: white;
}

.read-button:hover {
  background-color: #e9068f; /* A darker brown for hover */
}

.save-button {
  background-color: #007bff; /* A standard blue */
  color: white;
}

.save-button:hover {
  background-color: #0056b3; /* A darker blue for hover */
}

.font-size-button {
  background-color: #d9534f;
  color: white;
}

.font-size-button:hover {
  background-color: #c9302c;
}

.toggle-panel-button {
  background-color: #007bff; /* A standard blue */
  color: white;
}

.toggle-panel-button:hover {
  background-color: #0056b3; /* A darker blue for hover */
}

.hide-show-button {
  background-color: #28a745; /* A standard green */
  color: white;
}

.hide-show-button:hover {
  background-color: #218838; /* A darker green for hover */
}

textarea {
  flex-grow: 1;
  border-radius: 8px;
  border: 1px solid #444;
  padding: 0.8em;
  font-family: Menlo, Monaco, 'Courier New', monospace;
  resize: none;
  line-height: 1.5;
  background-color: #2a2a2a;
  color: #e0e0e0;
}

textarea:focus {
  outline: none;
  border-color: #5c7e10;
}

.error-display {
  color: #ff8a80;
  margin-top: 1rem;
  font-family: monospace;
  background-color: #4d2a2a;
  padding: 0.5rem;
  border-radius: 4px;
  border: 1px solid #ff5252;
}

.markdown-output {
  height: 100%;
  width: 100%;
  border: 1px solid #444;
  border-radius: 8px;
}
</style>