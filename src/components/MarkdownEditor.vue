<template>
  <div class="markdown-editor" :class="{ 'dark-mode': isDarkMode }">
    <div class="editor-header">
      <h2>Markdown 编辑器</h2>
      <!-- 将所有按钮移到标题下方 -->
      <div class="header-actions">
        <button @click="toggleDarkMode" class="dark-mode-button">
            切换模式
          </button>
        <button @click="togglePreview" class="toggle-preview-button">
          {{ showPreview ? '隐藏预览' : '显示预览' }}
        </button>
        <button @click="saveFile" class="save-button">保存</button>
        <input type="file" ref="fileInput" style="display: none" @change="loadFile" accept=".md,.markdown">
        <button @click="$refs.fileInput.click()" class="load-button">打开</button>
      </div>
    </div>
    
    <div class="editor-content" :class="{ 'preview-hidden': !showPreview }">
      <div class="editor-panel">
        <div class="panel-header">
          <h3>编辑区</h3>
          <span class="char-count">{{ markdownText.length }} 字符</span>
        </div>
        
        <!-- 快捷工具栏 -->
        <div class="toolbar">
          <button @click="insertMarkdown('# ')">一级标题</button>
          <button @click="insertMarkdown('## ')">二级标题</button>
          <button @click="insertMarkdown('### ')">三级标题</button>
          <button @click="insertMarkdown('**粗体**')">加粗</button>
          <button @click="insertMarkdown('*斜体*')">斜体</button>
          <button @click="insertMarkdown('- ')">列表</button>
          <button @click="insertMarkdown('```\n```\n')">代码块</button>
          <button @click="insertMarkdown('[链接文本](URL)')">链接</button>
          <button @click="insertMarkdown('![图片描述](URL)')">图片</button>
        </div>
        <div class="editor-container">
          <div class="line-numbers" :style="{ lineHeight: lineHeight + 'px' }" ref="lineNumbersRef">
            <div v-for="line in lineCount" :key="line" class="line-number">{{ line }}</div>
          </div>
          <textarea 
            v-model="markdownText"
            @input="handleInput"
            @scroll="syncScroll"
            placeholder="在此输入Markdown内容..."
            class="markdown-input"
            ref="textarea"
            spellcheck="false"
          ></textarea>
        </div>
      </div>
      
      <div class="preview-panel">
        <div class="panel-header">
          <h3>预览区</h3>
        </div>
        <div class="markdown-preview" v-html="renderedHtml"></div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, nextTick } from 'vue';
import MarkdownIt from 'markdown-it';

// 创建markdown-it实例 - 禁用语法检查相关功能
const md = new MarkdownIt({
  html: true,
  linkify: false,
  typographer: false
});

// 状态管理
const markdownText = ref('');
const renderedHtml = ref('');
const fileName = ref('document.md');
const textarea = ref(null);
const lineNumbersRef = ref(null);
const lineHeight = 24; // 行高，需要与CSS中设置的一致
const showPreview = ref(true); // 控制预览区显示/隐藏
const isDarkMode = ref(false); // 控制深色模式

// 初始化示例内容
const initialMarkdown = `# 欢迎使用 Markdown 编辑器

这是一个Markdown编辑器，支持实时预览、文件保存和打开。

## 功能特性
- 实时编辑 - 输入内容即时渲染
- 文件操作 - 支持保存和打开Markdown文件

## 基本语法示例

### 标题
# H1
## H2
### H3

### 强调
*斜体文本*
**粗体文本**
***粗斜体文本***

### 列表

#### 无序列表
- 项目1
- 项目2
- 项目3

#### 有序列表
1. 第一步
2. 第二步
3. 第三步

### 链接和图片
[GitHub](https://github.com)

![Vue Logo](https://vuejs.org/logo.svg)

### 代码
\`\`\`javascript
function greet(name) {
  return "Hello, " + name + "!";
}
\`\`\`

### 表格
| 名称 | 描述 |
|------|------|
| Markdown | 轻量级标记语言 |
| Vue.js | JavaScript框架 |
`;

// 计算行数
const lineCount = computed(() => {
  if (!markdownText.value) return 1;
  return markdownText.value.split('\n').length;
});

// 渲染Markdown为HTML
function renderMarkdown() {
  renderedHtml.value = md.render(markdownText.value || '');
}

// 处理输入事件
function handleInput() {
  renderMarkdown();
  syncScroll();
}

// 同步滚动
function syncScroll() {
  if (textarea.value && lineNumbersRef.value) {
    lineNumbersRef.value.scrollTop = textarea.value.scrollTop;
  }
}

// 在光标位置插入Markdown语法
function insertMarkdown(syntax) {
  if (!textarea.value) return;
  
  const textareaElem = textarea.value;
  const start = textareaElem.selectionStart;
  const end = textareaElem.selectionEnd;
  const currentValue = markdownText.value;
  
  // 插入语法
  markdownText.value = currentValue.substring(0, start) + syntax + currentValue.substring(end);
  
  // 设置光标位置
  nextTick(() => {
    textareaElem.focus();
    // 如果是标题或列表，光标放在内容后面
    if (syntax.endsWith(' ') || syntax.endsWith('\n')) {
      const newPosition = start + syntax.length;
      textareaElem.setSelectionRange(newPosition, newPosition);
    }
    // 如果是粗体/斜体，光标放在中间
    else if (syntax.includes('粗体') || syntax.includes('斜体') || syntax.includes('链接文本') || syntax.includes('URL') || syntax.includes('图片描述')) {
      const cursorPosition = syntax.includes('粗体') ? syntax.indexOf('粗体') : 
                           (syntax.includes('斜体') ? syntax.indexOf('斜体') : 
                           (syntax.includes('链接文本') ? syntax.indexOf('链接文本') : 
                           (syntax.includes('URL') ? syntax.indexOf('URL') : syntax.indexOf('图片描述'))));
      const newPosition = start + cursorPosition;
      textareaElem.setSelectionRange(newPosition, newPosition + (syntax.includes('粗体') ? 2 : (syntax.includes('斜体') ? 2 : (syntax.includes('链接文本') ? 4 : (syntax.includes('URL') ? 3 : 4)))));
    }
    // 如果是代码块，光标放在中间
    else if (syntax.includes('```')) {
      const newPosition = start + 4; // 放在第一对```后面的换行符后
      textareaElem.setSelectionRange(newPosition, newPosition);
    }
  });
  
  // 重新渲染和同步滚动
  renderMarkdown();
  syncScroll();
}

// 切换预览区显示/隐藏
function togglePreview() {
  showPreview.value = !showPreview.value;
}

// 切换深色模式
function toggleDarkMode() {
  isDarkMode.value = !isDarkMode.value;
}

// 保存文件
function saveFile() {
  const blob = new Blob([markdownText.value], { type: 'text/markdown' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = fileName.value;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
}

// 加载文件
function loadFile(event) {
  const file = event.target.files[0];
  if (file) {
    fileName.value = file.name;
    const reader = new FileReader();
    reader.onload = (e) => {
      markdownText.value = e.target.result;
      renderMarkdown();
    };
    reader.readAsText(file);
  }
}

// 初始化
onMounted(() => {
  markdownText.value = initialMarkdown;
  renderMarkdown();
});
</script>

<style scoped>
/* 隐藏所有滚动条 */
* {
  scrollbar-width: none; /* Firefox */
  -ms-overflow-style: none; /* IE 和 Edge */
}

*::-webkit-scrollbar {
  display: none; /* Chrome, Safari 和 Opera */
}

.markdown-editor {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: #ffffff;
  overflow: hidden;
  user-select: none; /* 禁止默认选择 */
}

/* 深色模式样式 */
.markdown-editor.dark-mode {
  background: #1e1e1e;
}

.markdown-editor.dark-mode .editor-header {
  background: #2d2d2d;
  border-bottom-color: #444;
}

.markdown-editor.dark-mode .editor-header h2 {
  color: #ffffff;
}

.markdown-editor.dark-mode .header-actions button {
  background: #3c3c3c;
  color: #ffffff;
  border-color: #555;
}

.markdown-editor.dark-mode .header-actions button:hover {
  background: #4a4a4a;
}

.editor-header {
  background: #f0f0f0;
  padding: 1rem 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  align-items: flex-start;
  border-bottom: 1px solid #ddd;
}

.editor-header h2 {
  color: #333;
  margin: 0;
  font-size: 1.2rem;
  font-weight: 600;
}

.header-actions {
  display: flex;
  gap: 0.5rem;
}

.save-button, .load-button, .toggle-preview-button, .dark-mode-button {
  background: #ffffff;
  color: #333;
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 0.5rem 1rem;
  font-size: 0.9rem;
  font-weight: normal;
  cursor: pointer;
  transition: none;
  box-shadow: none;
  user-select: none; /* 禁止选择 */
}

.save-button:hover, .load-button:hover, .toggle-preview-button:hover, .dark-mode-button:hover {
    background: #f5f5f5;
    transform: none;
    box-shadow: none;
  }

/* 深色模式完整样式 */
.markdown-editor.dark-mode .editor-panel,
.markdown-editor.dark-mode .preview-panel,
.markdown-editor.dark-mode .editor-container {
  background: #1e1e1e;
}

.markdown-editor.dark-mode .editor-panel {
  border-right-color: #444;
}

.markdown-editor.dark-mode .panel-header {
  background: #2d2d2d;
  border-bottom-color: #444;
}

.markdown-editor.dark-mode .panel-header h3,
.markdown-editor.dark-mode .char-count {
  color: #ffffff;
}

.markdown-editor.dark-mode .toolbar {
  background: #2d2d2d;
  border-bottom-color: #444;
}

.markdown-editor.dark-mode .toolbar button {
  background: #3c3c3c;
  color: #ffffff;
  border-color: #555;
}

.markdown-editor.dark-mode .toolbar button:hover {
  background: #4a4a4a;
}

.markdown-editor.dark-mode .line-numbers {
  background: #2d2d2d;
  border-right-color: #444;
  color: #888;
}

.markdown-editor.dark-mode .markdown-input {
  color: #ffffff;
  background: transparent;
}

.markdown-editor.dark-mode .markdown-input::placeholder {
  color: #888;
}

.markdown-editor.dark-mode .markdown-preview {
  background: #1e1e1e;
  color: #ffffff;
}

.markdown-editor.dark-mode .markdown-preview h1,
.markdown-editor.dark-mode .markdown-preview h2,
.markdown-editor.dark-mode .markdown-preview h3,
.markdown-editor.dark-mode .markdown-preview h4,
.markdown-editor.dark-mode .markdown-preview h5,
.markdown-editor.dark-mode .markdown-preview h6 {
  color: #ffffff;
}

.markdown-editor.dark-mode .markdown-preview a {
  color: #4da6ff;
}

.markdown-editor.dark-mode .markdown-preview code,
.markdown-editor.dark-mode .markdown-preview pre {
  background: #2d2d2d;
  border-color: #444;
}

.markdown-editor.dark-mode .markdown-preview blockquote {
  border-left-color: #555;
  color: #cccccc;
}

.markdown-editor.dark-mode .markdown-preview th {
  background: #2d2d2d;
  border-color: #444;
}

.markdown-editor.dark-mode .markdown-preview td {
  border-color: #444;
}

.editor-content {
  flex: 1;
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0;
  overflow: hidden;
}

/* 隐藏预览区时，编辑区占据整个宽度 */
.editor-content.preview-hidden {
  grid-template-columns: 1fr;
}

.editor-content.preview-hidden .preview-panel {
  display: none;
}

.editor-panel, .preview-panel {
  display: flex;
  flex-direction: column;
  background: #ffffff;
  overflow: hidden;
}

.editor-panel {
  border-right: 1px solid #ddd;
}

.editor-container {
  flex: 1;
  display: flex;
  background: #ffffff;
  position: relative;
  overflow: hidden;
}

.line-numbers {
  background: #f5f5f5;
  border-right: 1px solid #ddd;
  padding: 1rem 0.5rem;
  width: 30px;
  text-align: right;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
  font-size: 0.8rem;
  color: #666;
  line-height: 24px;
  overflow-y: hidden;
  user-select: none;
}

.line-number {
  height: 24px;
}

.markdown-input {
  flex: 1;
  width: 100%;
  padding: 1rem;
  border: none;
  resize: none;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
  font-size: 0.9rem;
  line-height: 24px; /* 与行高保持一致 */
  color: #333;
  background: transparent;
  outline: none;
  overflow-y: auto;
  user-select: text; /* 允许选择和复制 */
}

.panel-header {
  background: #f5f5f5;
  padding: 0.75rem 1rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #ddd;
}

/* 快捷工具栏样式 */
.toolbar {
  background: #fafafa;
  padding: 0.5rem 1rem;
  display: flex;
  gap: 0.25rem;
  border-bottom: 1px solid #ddd;
  flex-wrap: wrap;
}

.toolbar button {
  background: #ffffff;
  color: #333;
  border: 1px solid #ddd;
  border-radius: 3px;
  padding: 0.25rem 0.5rem;
  font-size: 0.8rem;
  font-weight: normal;
  cursor: pointer;
  transition: none;
  box-shadow: none;
  min-width: 30px;
  user-select: none;
}

.toolbar button:hover {
  background: #f5f5f5;
  transform: none;
  box-shadow: none;
}

.toolbar button:active {
  background: #e9e9e9;
}

.panel-header h3 {
  margin: 0;
  font-size: 0.9rem;
  font-weight: 600;
  color: #333;
}

.char-count {
  font-size: 0.8rem;
  color: #666;
}

.markdown-preview {
  flex: 1;
  padding: 1rem;
  overflow-y: auto;
  background: #ffffff;
  color: #333;
  user-select: text; /* 允许选择和复制 */
}

.markdown-preview h1, 
.markdown-preview h2, 
.markdown-preview h3, 
.markdown-preview h4, 
.markdown-preview h5, 
.markdown-preview h6 {
  color: #333;
  margin-top: 1.2em;
  margin-bottom: 0.5em;
}

.markdown-preview a {
  color: #0066cc;
  text-decoration: none;
}

.markdown-preview a:hover {
  text-decoration: underline;
}

.markdown-preview code {
  background: #f5f5f5;
  padding: 0.2em 0.4em;
  border-radius: 3px;
  font-family: monospace;
  font-size: 0.85rem;
}

.markdown-preview pre {
  background: #f5f5f5;
  padding: 0.75rem;
  border-radius: 4px;
  overflow-x: auto;
  border: 1px solid #ddd;
}

.markdown-preview pre code {
  background: transparent;
  padding: 0;
}

.markdown-preview blockquote {
  border-left: 3px solid #ccc;
  padding-left: 0.75rem;
  margin-left: 0;
  color: #555;
  font-style: italic;
}

.markdown-preview img {
  max-width: 100%;
  height: auto;
}

.markdown-preview table {
  width: 100%;
  border-collapse: collapse;
  margin: 0.75rem 0;
  font-size: 0.9rem;
}

.markdown-preview th, 
.markdown-preview td {
  padding: 0.5rem;
  border: 1px solid #ddd;
  text-align: left;
}

.markdown-preview th {
  background: #f5f5f5;
  font-weight: 600;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .editor-content {
    grid-template-columns: 1fr;
    grid-template-rows: 1fr 1fr;
  }
  
  .editor-panel {
    border-right: none;
    border-bottom: 1px solid #ddd;
  }
}
</style>