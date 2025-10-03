<template>
  <div class="app-shell">
    <!-- 左侧侧栏 -->
    <aside :class="['sidebar', { 'sidebar-hidden': !sidebarOpen }]">
      <div class="sidebar-header">
        <div class="brand">对话</div>
        <button class="new-chat-btn" @click="newConversation">＋ 新建对话</button>
      </div>
      <div class="sidebar-search">
        <input v-model="searchText" placeholder="搜索对话…" />
      </div>
      <div class="conversation-list">
        <div v-if="!conversations.length" class="empty-tip">
          暂无对话，点击上方创建
        </div>
        <div
            v-for="c in filteredConversations"
            :key="c.id"
            :class="['conversation-item', c.id === activeConversationId ? 'active' : '']"
            @click="selectConversation(c.id)"
        >
          <div class="conversation-title" @dblclick.stop="startRename(c)">
            <template v-if="renameId === c.id">
              <input
                  v-model="renameText"
                  class="rename-input"
                  @keyup.enter.stop="confirmRename(c)"
                  @blur="confirmRename(c)"
                  ref="renameInput"
              />
            </template>
            <template v-else>
              {{ c.title || '未命名对话' }}
            </template>
          </div>
          <div class="conversation-sub">
            {{ c.lastSummary || '（空）' }}
          </div>
          <button class="delete-btn" @click.stop="deleteConversation(c.id)">删除</button>
        </div>
      </div>
    </aside>

    <!-- 右侧聊天主区 -->
    <main class="main-pane">
      <header class="topbar">
        <button class="hamburger" @click="toggleSidebar">☰</button>
        <div class="title">{{ activeConversation?.title || '新的对话' }}</div>
        <div class="topbar-actions">
          <button class="clear-btn small" @click="clearMessages" :disabled="!messages.length">清屏</button>
        </div>
      </header>

      <!-- 聊天内容 -->
      <div class="chat-content">
        <div
            v-for="(msg, index) in messages"
            :key="index"
            :class="['message', msg.from === 'user' ? 'user-msg' : 'bot-msg']"
        >
          <template v-if="msg.from === 'user'">
            {{ msg.text }}
          </template>

          <template v-else>
            <!-- 思考内容 -->
            <details v-if="msg.think" class="think-block">
              <summary>思考过程</summary>
              <pre>{{ msg.think }}</pre>
            </details>

            <!-- 回复文本 -->
            <div v-if="msg.text" class="bot-response">
              <div class="bot-message-actions">
                <button class="copy-btn" @click="copyMessage(msg.text)" title="复制">📋</button>
              </div>

              <template v-if="shouldExpandAutomatically(msg.text)">
                <div class="final-answer">{{ msg.text }}</div>
              </template>
              <template v-else>
                <div v-if="!isMessageExpanded(index)" class="final-answer-container">
                  <div class="final-answer truncated">{{ msg.text }}</div>
                  <button class="expand-btn" @click="expandMessage(index)">展开</button>
                </div>
                <div v-else class="expanded-content">
                  <div class="final-answer">{{ msg.text }}</div>
                  <button class="collapse-btn" @click="collapseMessage(index)">收起</button>
                </div>
              </template>
            </div>
          </template>
        </div>

        <div v-if="loading" class="message bot-msg typing-indicator">
          <span class="dot"></span><span class="dot"></span><span class="dot"></span>
        </div>
      </div>

      <!-- 输入区域 -->
      <div class="input-area">
        <textarea
            v-model="inputText"
            placeholder="输入消息，Enter 发送，Shift/Ctrl 换行"
            :disabled="loading"
            rows="1"
            @keydown.enter.prevent="handleEnter"
            @input="autoResize"
            ref="inputBox"
        ></textarea>
        <button @click="sendMessage" :disabled="loading" class="send-btn">
          <span v-if="!loading">📤</span>
          <span v-else>⏳</span>
        </button>
      </div>
    </main>
  </div>
</template>

<script>
const LS_KEY = 'chat_conversations_v1';
function uuid() {
  return 'xxxxxx-xxxx-4xxx-yxxx-xxxxxxxx'.replace(/[xy]/g, c => {
    const r = (Math.random() * 16) | 0;
    const v = c === 'x' ? r : (r & 0x3) | 0x8;
    return v.toString(16);
  });
}

export default {
  data() {
    return {
      inputText: '',
      messages: [],
      conversationId: '',
      userId: 'abc-123',
      loading: false,
      conversations: [],
      activeConversationId: '',
      sidebarOpen: true,
      searchText: '',
      renameId: '',
      renameText: '',
      expandedMessages: new Set(),
    };
  },
  computed: {
    activeConversation() {
      return this.conversations.find(c => c.id === this.activeConversationId) || null;
    },
    filteredConversations() {
      const q = this.searchText.trim().toLowerCase();
      if (!q) return this.conversations;
      return this.conversations.filter(
          c =>
              (c.title && c.title.toLowerCase().includes(q)) ||
              (c.lastSummary && c.lastSummary.toLowerCase().includes(q))
      );
    },
  },
  mounted() {
    this.loadFromStorage();
    if (!this.conversations.length) this.newConversation();
    else if (!this.activeConversationId) {
      this.activeConversationId = this.conversations[0].id;
      this.syncFromActive();
    }
    this.$nextTick(() => {
      this.$refs.inputBox?.focus();
      this.scrollToBottom();
    });
    window.addEventListener('keydown', this.handleGlobalKeys);
    this.updateHeight();
    window.addEventListener('resize', this.updateHeight);
  },
  beforeUnmount() {
    window.removeEventListener('keydown', this.handleGlobalKeys);
    window.removeEventListener('resize', this.updateHeight);
  },
  methods: {
    toggleSidebar() { this.sidebarOpen = !this.sidebarOpen; },
    newConversation() {
      const id = uuid();
      const conv = { id, title: '新的对话', createdAt: Date.now(), lastSummary: '', backendId: '', messages: [] };
      this.conversations.unshift(conv);
      this.activeConversationId = id;
      this.messages = [];
      this.conversationId = '';
      this.persist();
      this.$nextTick(() => {
        this.scrollToBottom();
        this.$refs.inputBox?.focus();
      });
    },
    selectConversation(id) {
      if (id === this.activeConversationId) return;
      this.activeConversationId = id;
      this.syncFromActive();
      this.$nextTick(() => this.scrollToBottom());
    },
    deleteConversation(id) {
      const idx = this.conversations.findIndex(c => c.id === id);
      if (idx === -1) return;
      const isActive = id === this.activeConversationId;
      this.conversations.splice(idx, 1);
      if (isActive) {
        if (this.conversations.length) {
          this.activeConversationId = this.conversations[0].id;
          this.syncFromActive();
        } else this.newConversation();
      }
      this.persist();
    },
    startRename(c) { this.renameId = c.id; this.renameText = c.title || ''; },
    confirmRename(c) {
      const name = this.renameText.trim();
      if (name) c.title = name;
      this.renameId = ''; this.renameText = '';
      this.persist();
    },
    persist() {
      localStorage.setItem(LS_KEY, JSON.stringify({
        conversations: this.conversations,
        activeConversationId: this.activeConversationId
      }));
    },
    loadFromStorage() {
      try {
        const raw = localStorage.getItem(LS_KEY);
        if (!raw) return;
        const obj = JSON.parse(raw);
        this.conversations = Array.isArray(obj.conversations) ? obj.conversations : [];
        this.activeConversationId = obj.activeConversationId || (this.conversations[0]?.id || '');
        this.syncFromActive();
      } catch (e) { console.warn('loadFromStorage error', e); }
    },
    syncFromActive() {
      const c = this.activeConversation;
      if (!c) return;
      this.messages = Array.isArray(c.messages) ? [...c.messages] : [];
      this.conversationId = c.backendId || '';
      this.$nextTick(() => this.scrollToBottom());
    },
    saveActiveMessages() {
      const c = this.activeConversation;
      if (!c) return;
      c.messages = [...this.messages];
      c.backendId = this.conversationId || '';
      const last = c.messages[c.messages.length - 1];
      c.lastSummary = last ? last.text.slice(0, 28) : '';
      this.persist();
    },
    handleGlobalKeys(e) {
      const mod = e.ctrlKey || e.metaKey;
      if (mod && e.key.toLowerCase() === 'n') { e.preventDefault(); this.newConversation(); }
    },
    updateHeight() {
      document.documentElement.style.setProperty('--vh', `${window.innerHeight * 0.01}px`);
    },
    autoResize(e) {
      e.target.style.height = 'auto';
      e.target.style.height = `${e.target.scrollHeight}px`;
    },
    handleEnter(e) {
      if (e.shiftKey || e.ctrlKey) {
        const start = e.target.selectionStart;
        const end = e.target.selectionEnd;
        this.inputText = this.inputText.substring(0, start) + '\n' + this.inputText.substring(end);
        this.$nextTick(() => {
          e.target.selectionStart = e.target.selectionEnd = start + 1;
        });
      } else this.sendMessage();
    },
    isMessageExpanded(index) { return this.expandedMessages.has(index); },
    expandMessage(index) { this.expandedMessages.add(index); this.$nextTick(() => this.scrollToBottom()); },
    collapseMessage(index) { this.expandedMessages.delete(index); this.$nextTick(() => this.scrollToBottom()); },
    shouldExpandAutomatically(text) {
      const lines = text.split('\n');
      const lineCount = lines.reduce((count, line) => count + Math.max(1, Math.ceil(line.length / 40)), 0);
      return lineCount < 10;
    },
    async copyMessage(text) {
      try { await navigator.clipboard.writeText(text); alert('已复制到剪贴板'); }
      catch (err) { console.error('复制失败', err); alert('复制失败，请手动复制'); }
    },
    async sendMessage() {
      const userMessage = this.inputText.trim();
      if (!userMessage || this.loading) return;

      this.messages.push({ from: 'user', text: userMessage });
      this.inputText = '';
      this.$nextTick(() => { this.autoResize({ target: this.$refs.inputBox }); this.scrollToBottom(); });
      this.saveActiveMessages();

      this.loading = true;
      const botIndex = this.messages.push({ from: 'bot', text: '', think: '' }) - 1;
      this.$nextTick(() => this.scrollToBottom());

      try {
        const res = await fetch('http://127.0.0.1:8090/api/chat/stream', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ query: userMessage, conversation_id: this.conversationId, user: this.userId })
        });

        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        if (!res.body) throw new Error('ReadableStream not supported');

        const reader = res.body.getReader();
        const decoder = new TextDecoder('utf-8');
        let done = false, accumulated = '';
        let currentThinkContent = ''; // 用于累积当前思考内容
        let inThinkBlock = false;    // 标记是否在思考区块内

        while (!done) {
          const { value, done: readerDone } = await reader.read();
          done = readerDone;
          if (value) {
            accumulated += decoder.decode(value, { stream: true });
            const lines = accumulated.split('\n');
            accumulated = lines.pop() || '';

            for (const line of lines) {
              const trim = line.trim();
              if (!trim.startsWith('data:')) continue;

              const jsonStr = trim.substring(5).trim();
              if (jsonStr === '[DONE]') { done = true; break; }

              try {
                const dataObj = JSON.parse(jsonStr);
                if (dataObj.event === 'message' && dataObj.answer !== undefined) {
                  const answer = dataObj.answer;

                  // 处理思考区块
                  for (let i = 0; i < answer.length; i++) {
                    const char = answer[i];

                    // 检测思考区块开始
                    if (!inThinkBlock && answer.startsWith('<think>', i)) {
                      inThinkBlock = true;
                      i += 6; // 跳过 <think> 的7个字符中的6个（循环会再+1）
                      continue;
                    }

                    // 检测思考区块结束
                    if (inThinkBlock && answer.startsWith('</think>', i)) {
                      inThinkBlock = false;
                      i += 7; // 跳过 </think> 的8个字符中的7个

                      // 保存思考内容并重置
                      if (currentThinkContent) {
                        this.messages[botIndex].think += currentThinkContent.trim();
                        currentThinkContent = '';
                      }
                      continue;
                    }

                    // 根据状态将内容添加到相应区域
                    if (inThinkBlock) {
                      currentThinkContent += char;
                    } else {
                      this.messages[botIndex].text += char;
                    }
                  }

                  this.$nextTick(() => this.scrollToBottom());
                  this.saveActiveMessages();
                }
                if (dataObj.conversation_id) {
                  this.conversationId = dataObj.conversation_id;
                  this.saveActiveMessages();
                }
                if (dataObj.event === 'message_end') {
                  // 确保思考区块正确关闭
                  if (inThinkBlock && currentThinkContent) {
                    this.messages[botIndex].think += currentThinkContent;
                    currentThinkContent = '';
                    inThinkBlock = false;
                  }
                  done = true;
                  break;
                }
              } catch (err) { console.error('JSON parse error', jsonStr, err); }
            }
          }
        }
      } catch (err) {
        console.error(err);
        this.messages[botIndex].text = `抱歉，获取回复时出错。\n${err.message || ''}`;
      } finally {
        this.loading = false;
        this.saveActiveMessages();
        this.$nextTick(() => this.scrollToBottom());
      }
      },
    clearMessages() { this.messages = []; this.saveActiveMessages(); this.$nextTick(() => this.scrollToBottom()); },
    scrollToBottom() {
      window.scrollTo({ top: document.documentElement.scrollHeight, behavior: 'smooth' });
    },
  },
};
</script>


<style scoped>
/* 全局重置 */
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html, body {
  width: 100%;
  height: 100%;
  overflow-x: hidden;
  background: #f5f6f8;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
}

/* 核心修复：移除内部滚动 */
.app-shell {
  width: 100vw;
  min-height: 100vh;
  display: flex;
  background: #f5f6f8;
}

/* 修复侧栏 */
.sidebar {
  width: 20%;
  min-width: 240px;
  max-width: 360px;
  height: 100vh;
  border-right: 1px solid #e5e7eb;
  background: #fff;
  display: flex;
  flex-direction: column;
  flex-shrink: 0;
  position: fixed;
  left: 0;
  top: 0;
  bottom: 0;
  z-index: 100;
}

.sidebar-hidden {
  display: none;
}

.sidebar-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 14px 12px;
  border-bottom: 1px solid #eef0f3;
}

.brand {
  font-weight: 700;
  font-size: 16px;
}

.new-chat-btn {
  padding: 6px 10px;
  border: none;
  background: #007bff;
  color: #fff;
  border-radius: 10px;
  cursor: pointer;
}

.new-chat-btn.small {
  padding: 6px 10px;
  font-size: 12px;
  border-radius: 8px;
}

.sidebar-search {
  padding: 10px 12px;
  border-bottom: 1px solid #f1f1f1;
}

.sidebar-search input {
  width: 100%;
  padding: 8px 10px;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  outline: none;
}

.sidebar-search input:focus {
  border-color: #007bff;
  box-shadow: 0 0 0 3px rgba(0, 123, 255, .08);
}

.conversation-list {
  flex: 1;
  overflow-y: auto;
  padding: 6px;
}

.empty-tip {
  text-align: center;
  color: #9ca3af;
  font-size: 14px;
  padding: 20px;
}

.conversation-item {
  padding: 10px 10px 10px 12px;
  border-radius: 12px;
  cursor: pointer;
  position: relative;
  margin: 4px 2px;
  border: 1px solid transparent;
}

.conversation-item:hover {
  background: #f7f8fa;
}

.conversation-item.active {
  background: #eef5ff;
  border-color: #cfe2ff;
}

.conversation-title {
  font-size: 14px;
  font-weight: 600;
  margin-bottom: 4px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.conversation-sub {
  font-size: 12px;
  color: #6b7280;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.delete-btn {
  position: absolute;
  right: 8px;
  top: 8px;
  border: none;
  background: transparent;
  color: #9ca3af;
  cursor: pointer;
  font-size: 12px;
}

.delete-btn:hover {
  color: #ef4444;
}

.rename-input {
  width: 100%;
  padding: 6px 8px;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  outline: none;
}

/* 修复主面板 */
.main-pane {
  flex: 1;
  margin-left: 20%; /* 与侧栏宽度匹配 */
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  padding-bottom: 80px; /* 为输入区域留出空间 */
}

.topbar {
  height: 52px;
  min-height: 52px;
  border-bottom: 1px solid #e5e7eb;
  background: #fff;
  display: flex;
  align-items: center;
  padding: 0 12px;
  gap: 8px;
  flex-shrink: 0;
  position: sticky;
  top: 0;
  z-index: 10;
}

.hamburger {
  width: 36px;
  height: 32px;
  border-radius: 8px;
  border: 1px solid #e5e7eb;
  background: #fff;
  cursor: pointer;
}

.title {
  font-weight: 600;
  flex: 0.9;
  overflow: hidden;
  text-overflow: ellipsis;
}

.topbar-actions {
  display: flex;
  gap: 6px;
  flex-shrink: 0;
}

.clear-btn.small {
  padding: 6px 10px;
  font-size: 12px;
  border: none;
  background: #e5e5e5;
  border-radius: 8px;
  cursor: pointer;
}

.clear-btn.small:disabled {
  opacity: .6;
  cursor: not-allowed;
}

.chat-content {
  display: flex;
  flex-direction: column;
  padding: 20px 0;
  max-width: 720px;
  margin: 0 auto;
  width: 100%;
  padding-bottom: 80px;
}

/* 修复消息对齐 - 关键修复 */
.message {
  margin-bottom: 15px;
  padding: 12px 16px;
  border-radius: 18px;
  max-width: 90%;
  word-wrap: break-word;
  line-height: 1.5;
  overflow: visible;
  display: inline-block; /* 改为 inline-block 以便使用 margin-left: auto */
  width: auto;
}

/* 用户消息显示在右侧 */
.message.user-msg {
  background-color: #007bff;
  color: white;
  border-bottom-right-radius: 5px;
  margin-left: auto; /* 关键：将整条消息推到右侧 */
  text-align: left;
}

/* 机器人消息显示在左侧 */
.message.bot-msg {
  background-color: #e9ecef;
  color: #212529;
  border-bottom-left-radius: 5px;
  margin-right: auto; /* 将机器人消息推到左侧 */
  text-align: left;
}

.typing-indicator {
  display: inline-block; /* 与消息保持一致 */
  padding: 12px 16px;
  background-color: #e9ecef;
  border-radius: 18px;
  margin-right: auto; /* 机器人打字指示器靠左 */
  margin-bottom: 15px;
}

.typing-indicator .dot {
  display: inline-block;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background-color: #666;
  margin-right: 4px;
  animation: typing 1.4s infinite ease-in-out;
}

.typing-indicator .dot:nth-child(1) {
  animation-delay: 0s;
}

.typing-indicator .dot:nth-child(2) {
  animation-delay: 0.2s;
}

.typing-indicator .dot:nth-child(3) {
  animation-delay: 0.4s;
}

@keyframes typing {
  0%, 60%, 100% {
    transform: translateY(0);
  }
  30% {
    transform: translateY(-4px);
  }
}

/* 输入区域固定在底部 */
.input-area {
  position: fixed;
  bottom: 0;
  left: 20%; /* 与侧栏宽度匹配 */
  right: 0;
  background-color: #fff;
  border-top: 1px solid #dee2e6;
  padding: 12px 20px;
  max-width: 720px;
  margin: 0 auto;
  width: 100%;
  display: flex;
  z-index: 100;
}

textarea {
  flex: 1;
  padding: 12px 15px;
  font-size: 16px;
  border: 1px solid #ced4da;
  border-radius: 8px;
  resize: none;
  outline: none;
  max-height: 200px;
  overflow-y: auto;
}

textarea:disabled {
  background: #e9ecef;
}

.send-btn {
  margin-left: 8px;
  padding: 0;
  border: none;
  background: #007bff;
  color: white;
  cursor: pointer;
  width: 48px;
  height: 48px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 8px;
  font-size: 20px;
}

.send-btn:disabled {
  background: #a5c8ff;
  cursor: not-allowed;
}

/* 内容显示 */
.final-answer-container {
  display: flex;
  align-items: flex-start;
  gap: 6px;
  max-width: 100%;
}

.final-answer {
  margin: 0;
  padding: 0;
  font-size: inherit;
  line-height: inherit;
  word-wrap: break-word;
  overflow-wrap: break-word;
}

.final-answer.truncated {
  display: -webkit-box;
  -webkit-line-clamp: 10;
  -webkit-box-orient: vertical;
  overflow: hidden;
  flex: 1;
}

.expanded-content {
  display: flex;
  flex-direction: column;
  width: 100%;
}

.expanded-content .final-answer {
  margin-bottom: 4px;
}

.expand-btn, .collapse-btn {
  flex-shrink: 0;
  background: transparent;
  border: none;
  color: #007bff;
  cursor: pointer;
  font-size: 12px;
  padding: 0;
  margin-top: 2px;
}

/* 修复用户消息中的按钮位置 */
.user-msg .expand-btn,
.user-msg .collapse-btn {
  margin-left: auto; /* 在用户消息中，按钮也靠右 */
}

.collapse-btn {
  color: #6b7280;
}

.bot-response {
  display: flex;
  flex-direction: column;
  width: 100%;
}

.bot-message-actions {
  display: flex;
  justify-content: flex-end;
  margin: -4px 0 4px 0;
}

.copy-btn {
  background: transparent;
  border: none;
  font-size: 14px;
  cursor: pointer;
  opacity: 0.6;
}

.copy-btn:hover {
  opacity: 1;
}

.think-block {
  margin-bottom: 8px;
  font-size: 12px;
  color: #6b7280;
  border-left: 2px solid #cfd4d9;
  padding-left: 8px;
}

.think-block pre {
  white-space: pre-wrap;
  word-wrap: break-word;
  overflow-wrap: break-word;
  margin: 0;
  font-family: inherit; /* 保持和聊天字体一致 */
  line-height: 1.5;
}

.think-block summary {
  cursor: pointer;
  outline: none;
}
</style>