
复制
询问
解释

翻译(zh-CN)





<template>
  <div class="qwen-chat-container">
    <!-- 顶部导航栏 -->
    <header class="qwen-header">
      <div class="header-content">
        <div class="logo-section">
          <div class="logo">Q</div>
          <span class="logo-text">通义千问</span>
        </div>
        <nav class="nav-links">
          <!-- 可以添加导航链接 -->
        </nav>
        <div class="header-actions">
          <!-- 可以添加用户头像、设置等 -->
        </div>
      </div>
    </header>

    <div class="main-layout">
      <!-- 左侧边栏 (简化版) -->
      <aside class="sidebar">
        <button class="new-chat-btn" @click="createNewConversation">
          <i class="fas fa-plus"></i> 新建对话
        </button>
        <div class="conversations-list">
          <div
              v-for="(conv, index) in conversationHistory"
              :key="index"
              class="conversation-item"
              :class="{ active: conv.id === currentConversationId }"
              @click="switchConversation(conv.id)"
          >
            <i class="fas fa-comment"></i>
            <span class="conv-title">{{ conv.title || '新对话' }}</span>
          </div>
        </div>
      </aside>

      <!-- 主聊天区域 -->
      <main class="chat-main">
        <div class="messages-container" ref="messagesContainer">
          <!-- 欢迎信息 -->
          <div v-if="messages.length === 0" class="welcome-message">
            <div class="welcome-logo">Q</div>
            <h1>你好，我是你的专属智能体</h1>
            <p>我可以帮助你回答问题、创作文字、进行逻辑推理、编程等。</p>
          </div>

          <!-- 聊天消息 -->
          <div
              v-for="(msg, index) in messages"
              :key="index"
              :class="['message-row', msg.from === 'user' ? 'user-row' : 'bot-row']"
          >
            <div class="avatar" :class="msg.from === 'user' ? 'user-avatar' : 'bot-avatar'">
              {{ msg.from === 'user' ? '你' : 'Q' }}
            </div>
            <div class="message-content" :class="msg.from === 'user' ? 'user-message' : 'bot-message'">
              {{ msg.text }}
            </div>
          </div>

          <!-- 正在输入指示器 -->
          <div v-if="loading" class="message-row bot-row">
            <div class="avatar bot-avatar">Q</div>
            <div class="message-content bot-message typing-indicator">
              <span class="dot"></span>
              <span class="dot"></span>
              <span class="dot"></span>
            </div>
          </div>
        </div>

        <!-- 输入区域 -->
        <div class="input-area-container">
          <div class="input-wrapper">
            <input
                v-model="inputText"
                @keyup.enter="sendMessage"
                :disabled="loading"
                placeholder="请输入消息..."
                class="chat-input"
            />
            <button @click="sendMessage" :disabled="loading" class="send-button">
              <i :class="loading ? 'fas fa-spinner fa-spin' : 'fas fa-paper-plane'"></i>
            </button>
          </div>
        </div>
      </main>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      inputText: "",
      messages: [], // 当前对话的消息
      conversationId: "", // 当前对话ID
      userId: "abc-123",
      loading: false,

      // --- 侧边栏数据 ---
      currentConversationId: 1, // 当前激活的对话ID
      conversationHistory: [ // 简化的对话历史
        { id: 1, title: "新对话" }
      ],
      conversationsData: { // 存储所有对话的数据
        1: [
          { from: "user", text: "如何学习Vue.js?" },
        ],
        2: [] // 新对话初始为空
      }
    };
  },
  watch: {
    // 监听当前对话ID变化，加载对应消息
    currentConversationId(newId) {
      this.loadConversation(newId);
    }
  },
  mounted() {
    // 组件挂载时加载当前对话
    this.loadConversation(this.currentConversationId);
  },
  methods: {
    // 加载指定ID的对话
    loadConversation(id) {
      this.messages = this.conversationsData[id] || [];
      // 如果是新对话，重置 conversationId
      if (id === this.conversationHistory.find(c => c.title === '新对话')?.id) {
        this.conversationId = "";
      }
      this.$nextTick(() => this.scrollToBottom());
    },

    // 创建新对话
    createNewConversation() {
      const newId = Date.now(); // 简单生成新ID
      this.conversationHistory.push({ id: newId, title: "新对话" });
      this.conversationsData[newId] = [];
      this.currentConversationId = newId;
      this.conversationId = ""; // 重置后端对话ID
      this.inputText = "";
    },

    // 切换对话
    switchConversation(id) {
      this.currentConversationId = id;
      this.inputText = "";
    },

    async sendMessage() {
      const userMessage = this.inputText.trim();
      if (!userMessage || this.loading) return;

      // 1. 添加用户消息到当前对话数据
      if (!this.conversationsData[this.currentConversationId]) {
        this.conversationsData[this.currentConversationId] = [];
      }
      this.conversationsData[this.currentConversationId].push({ from: "user", text: userMessage });
      // 同步到当前显示的消息列表
      this.messages = this.conversationsData[this.currentConversationId];

      // 更新对话标题 (如果是新对话)
      const currentConv = this.conversationHistory.find(c => c.id === this.currentConversationId);
      if (currentConv && currentConv.title === "新对话") {
        currentConv.title = userMessage.length > 15 ? userMessage.substring(0, 15) + '...' : userMessage;
      }

      this.inputText = ""; // 清空输入框
      this.$nextTick(() => this.scrollToBottom()); // 滚动到底部

      // 2. 设置加载状态
      this.loading = true;
      let botMessageIndex = -1; // 用于追踪最新 bot 消息的索引

      // 3. 构造请求载荷
      // 注意：这里需要根据实际后端要求调整 conversation_id 的传递
      // 如果是新对话，可能需要传递空字符串或特定标识
      const payload = {
        query: userMessage,
        conversation_id: this.conversationId || "", // 使用后端返回的ID或空
        user: this.userId,
      };

      console.log("Sending payload:", payload);

      try {
        // 4. 发起 fetch 请求
        const response = await fetch("http://127.0.0.1:8090/api/chat/stream", {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify(payload),
        });

        if (!response.ok) {
          let errorMsg = `HTTP error! status: ${response.status}`;
          try {
            const errorText = await response.text();
            errorMsg += ` - ${errorText}`;
          } catch (e) {
            console.warn("Could not read error response body:", e);
          }
          throw new Error(errorMsg);
        }

        if (!response.body) {
          throw new Error("ReadableStream not supported in this browser.");
        }

        const reader = response.body.getReader();
        const decoder = new TextDecoder("utf-8");
        let done = false;
        let accumulatedData = "";

        // 8. 添加初始的 bot 消息占位符到当前对话数据
        botMessageIndex = this.conversationsData[this.currentConversationId].push({ from: "bot", text: "" }) - 1;
        // 同步到当前显示的消息列表
        this.messages = this.conversationsData[this.currentConversationId];
        this.$nextTick(() => this.scrollToBottom());

        while (!done) {
          const { value, done: readerDone } = await reader.read();
          done = readerDone;
          if (value) {
            const chunk = decoder.decode(value, { stream: true });
            accumulatedData += chunk;

            const lines = accumulatedData.split("\n");
            accumulatedData = lines.pop() || "";

            for (const line of lines) {
              const trimmedLine = line.trim();
              if (!trimmedLine) continue;

              let jsonDataStr = trimmedLine;
              if (trimmedLine.startsWith("data:")) {
                jsonDataStr = trimmedLine.substring(5).trim();
              } else {
                console.warn('Received non-SSE data line (expected "data:" prefix):', trimmedLine);
                continue;
              }

              if (jsonDataStr === "[DONE]") {
                console.log("Stream finished with [DONE] signal");
                done = true;
                break;
              }

              try {
                const dataObj = JSON.parse(jsonDataStr);

                if (dataObj.event === "message") {
                  if (dataObj.answer !== undefined) {
                    const answerToCheck = dataObj.answer.trim();
                    if (answerToCheck !== "<think>" && answerToCheck !== "</think>") {
                      if (botMessageIndex >= 0 && botMessageIndex < this.conversationsData[this.currentConversationId].length) {
                        this.conversationsData[this.currentConversationId][botMessageIndex].text += dataObj.answer;
                        // 同步到当前显示的消息列表
                        this.messages = this.conversationsData[this.currentConversationId];
                        this.$nextTick(() => this.scrollToBottom());
                      } else {
                        console.error("Invalid botMessageIndex during stream processing:", botMessageIndex);
                      }
                    } else {
                      console.log("Skipping filtered answer content:", dataObj.answer);
                    }
                  }

                  if (dataObj.conversation_id) {
                    console.log("Updating conversation ID from backend:", dataObj.conversation_id);
                    this.conversationId = dataObj.conversation_id;
                  }

                } else if (dataObj.event === "message_end") {
                  console.log("Stream finished with message_end event");
                  if (dataObj.metadata) {
                    console.log("Stream metadata:", dataObj.metadata);
                  }
                  done = true;
                  break;
                } else {
                  console.warn("Received unexpected event type:", dataObj.event, dataObj);
                }

              } catch (parseError) {
                console.error("Error parsing JSON after removing 'data:' prefix:", jsonDataStr, parseError);
              }
            }
          }
        }

        if (accumulatedData.trim()) {
          let finalJsonDataStr = accumulatedData.trim();
          if (finalJsonDataStr.startsWith("data:")) {
            finalJsonDataStr = finalJsonDataStr.substring(5).trim();
          }
          if (finalJsonDataStr === "[DONE]") {
            console.log("Stream finished with final [DONE] signal");
          } else if (finalJsonDataStr) {
            try {
              const dataObj = JSON.parse(finalJsonDataStr);
              if (dataObj.event === "message" && dataObj.answer !== undefined) {
                const answerToCheck = dataObj.answer.trim();
                if (answerToCheck !== "<think>" && answerToCheck !== "</think>") {
                  if (botMessageIndex >= 0 && botMessageIndex < this.conversationsData[this.currentConversationId].length) {
                    this.conversationsData[this.currentConversationId][botMessageIndex].text += dataObj.answer;
                    this.messages = this.conversationsData[this.currentConversationId];
                    this.$nextTick(() => this.scrollToBottom());
                  }
                }
              }
              if (dataObj.conversation_id) {
                this.conversationId = dataObj.conversation_id;
              }
              if (dataObj.event === "message_end") {
                console.log("Stream finished with final message_end event (accumulated data)");
              }
            } catch (parseError) {
              console.error("Error parsing final accumulated JSON chunk:", finalJsonDataStr, parseError);
            }
          }
        }

        console.log("Stream processing completed successfully.");

      } catch (error) {
        console.error("Error in sendMessage (stream processing):", error);
        if (botMessageIndex >= 0 && botMessageIndex < this.conversationsData[this.currentConversationId].length) {
          this.conversationsData[this.currentConversationId][botMessageIndex].text = `抱歉，获取回复时遇到问题，请稍后重试。\n详细信息: ${error.message || "未知错误"}`;
          this.messages = this.conversationsData[this.currentConversationId];
        } else {
          const errorMsg = `抱歉，发送请求时遇到问题，请稍后重试。\n详细信息: ${error.message || "未知错误"}`;
          this.conversationsData[this.currentConversationId].push({ from: "bot", text: errorMsg });
          this.messages = this.conversationsData[this.currentConversationId];
        }
      } finally {
        this.loading = false;
        this.$nextTick(() => this.scrollToBottom());
      }
    },

    scrollToBottom() {
      const container = this.$refs.messagesContainer;
      if (container) {
        container.scrollTo({ top: container.scrollHeight, behavior: 'smooth' });
      }
    },
  },
};
</script>
<style scoped>
/* --- 基础重置和全局样式 --- */
.qwen-chat-container {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background-color: #fafafa; /* 浅灰色背景 */
  color: #1a1a1a; /* 深色文字 */
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

/* --- 顶部导航栏 --- */
.qwen-header {
  background-color: #ffffff;
  border-bottom: 1px solid #e5e7eb;
  padding: 0 20px;
  height: 60px;
  display: flex;
  align-items: center;
  flex-shrink: 0;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
  z-index: 10;
}

.header-content {
  width: 100%;
  max-width: 1200px; /* 限制最大宽度 */
  margin: 0 auto;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.logo-section {
  display: flex;
  align-items: center;
  gap: 12px;
}

.logo {
  width: 32px;
  height: 32px;
  background: linear-gradient(135deg, #0077cc, #0055aa); /* Qwen 蓝色 */
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-weight: bold;
  font-size: 18px;
}

.logo-text {
  font-size: 20px;
  font-weight: 600;
  color: #1a1a1a;
}

.nav-links {
  display: flex;
  gap: 24px;
}

.nav-links a {
  text-decoration: none;
  color: #666666;
  font-size: 14px;
  transition: color 0.2s;
}

.nav-links a:hover {
  color: #0077cc;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 16px;
}

/* --- 主布局 --- */
.main-layout {
  display: flex;
  flex: 1;
  overflow: hidden;
}

/* --- 左侧边栏 --- */
.sidebar {
  width: 260px;
  background-color: #ffffff;
  border-right: 1px solid #e5e7eb;
  display: flex;
  flex-direction: column;
  padding: 16px;
  flex-shrink: 0;
  overflow-y: auto;
}

.new-chat-btn {
  background: #0077cc;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 12px 16px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  transition: background-color 0.2s;
  margin-bottom: 20px;
  width: 100%;
}

.new-chat-btn:hover {
  background: #0055aa;
}

.new-chat-btn i {
  font-size: 14px;
}

.conversations-list {
  flex: 1;
  overflow-y: auto;
}

.conversation-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 12px;
  border-radius: 8px;
  cursor: pointer;
  margin-bottom: 6px;
  font-size: 14px;
  color: #374151;
  transition: background-color 0.2s;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.conversation-item:hover {
  background-color: #f3f4f6;
}

.conversation-item.active {
  background-color: #dbeafe;
  color: #1d4ed8;
  font-weight: 500;
}

.conversation-item i {
  font-size: 16px;
  color: #6b7280;
}

.conv-title {
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* --- 主聊天区域 --- */
.chat-main {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;
}

/* --- 消息容器 --- */
.messages-container {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  display: flex;
  flex-direction: column;
  gap: 20px;
}

/* --- 欢迎信息 --- */
.welcome-message {
  text-align: center;
  padding: 40px 20px;
  color: #6b7280;
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.welcome-logo {
  width: 64px;
  height: 64px;
  background: linear-gradient(135deg, #0077cc, #0055aa);
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-weight: bold;
  font-size: 32px;
  margin-bottom: 24px;
}

.welcome-message h1 {
  font-size: 32px;
  font-weight: 600;
  margin-bottom: 16px;
  color: #111827;
}

.welcome-message p {
  font-size: 18px;
  max-width: 600px;
  line-height: 1.6;
}

/* --- 消息行 --- */
.message-row {
  display: flex;
  gap: 16px;
  max-width: 100%;
  animation: fadeIn 0.2s ease-out;
}

.user-row {
  justify-content: flex-end;
}

.bot-row {
  justify-content: flex-start;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

/* --- 头像 --- */
.avatar {
  width: 36px;
  height: 36px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  font-weight: 500;
  font-size: 14px;
}

.user-avatar {
  background: linear-gradient(135deg, #0077cc, #0055aa);
  color: white;
}

.bot-avatar {
  background: linear-gradient(135deg, #10a37f, #1fc486); /* Qwen 绿色 */
  color: white;
}

/* --- 消息内容 --- */
.message-content {
  line-height: 1.6;
  padding: 12px 16px;
  border-radius: 8px;
  max-width: calc(100% - 52px);
  word-wrap: break-word;
  white-space: pre-wrap; /* 保留换行和空格 */
}

.user-message {
  background-color: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 16px 16px 4px 16px;
  color: #1a1a1a;
}

.bot-message {
  background-color: #f1f5f9; /* 浅蓝色背景 */
  border-radius: 16px 16px 16px 4px;
  color: #1a1a1a;
}

/* --- 输入区域容器 --- */
.input-area-container {
  padding: 16px 20px 24px;
  background-color: #ffffff;
  border-top: 1px solid #e5e7eb;
  flex-shrink: 0;
}

/* --- 输入框包装器 --- */
.input-wrapper {
  border: 1px solid #e5e7eb;
  border-radius: 20px;
  background-color: #ffffff;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.02);
  transition: border-color 0.2s, box-shadow 0.2s;
  position: relative;
  display: flex;
  align-items: center;
}

.input-wrapper:focus-within {
  border-color: #0077cc;
  box-shadow: 0 0 0 3px rgba(0, 119, 204, 0.1);
}

/* --- 聊天输入框 --- */
.chat-input {
  flex: 1;
  padding: 16px 56px 16px 20px;
  border: none;
  border-radius: 20px;
  font-size: 16px;
  line-height: 1.5;
  outline: none;
  background: transparent;
  resize: none; /* 禁止调整大小 */
  max-height: 200px;
}

/* --- 发送按钮 --- */
.send-button {
  position: absolute;
  right: 12px;
  width: 36px;
  height: 36px;
  border-radius: 10px;
  background: #0077cc;
  color: white;
  border: none;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background-color 0.2s;
}

.send-button:hover:not(:disabled) {
  background: #0055aa;
}

.send-button:disabled {
  background: #9ca3af;
  cursor: not-allowed;
}

.send-button i {
  font-size: 16px;
}

/* --- 正在输入指示器 --- */
.typing-indicator {
  display: flex;
  align-items: center;
  padding: 12px 16px;
  background-color: #f1f5f9;
  color: #64748b;
  font-style: italic;
  border-radius: 16px 16px 16px 4px;
}

.dot {
  width: 8px;
  height: 8px;
  background-color: #94a3b8;
  border-radius: 50%;
  margin: 0 3px;
  animation: typing 1.4s infinite ease-in-out;
}

.dot:nth-child(1) { animation-delay: -0.32s; }
.dot:nth-child(2) { animation-delay: -0.16s; }

@keyframes typing {
  0%, 80%, 100% { transform: scale(0); opacity: 0.5; }
  40% { transform: scale(1); opacity: 1; }
}

/* --- 滚动条样式 --- */
.messages-container::-webkit-scrollbar,
.conversations-list::-webkit-scrollbar {
  width: 8px;
}

.messages-container::-webkit-scrollbar-track,
.conversations-list::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 4px;
}

.messages-container::-webkit-scrollbar-thumb,
.conversations-list::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 4px;
}

.messages-container::-webkit-scrollbar-thumb:hover,
.conversations-list::-webkit-scrollbar-thumb:hover {
  background: #a8a8a8;
}

/* --- 响应式 (简化) --- */
@media (max-width: 768px) {
  .sidebar {
    width: 80px; /* 折叠侧边栏 */
  }
  .conv-title {
    display: none; /* 隐藏文字 */
  }
  .new-chat-btn span {
    display: none; /* 隐藏按钮文字 */
  }
  .new-chat-btn {
    justify-content: center;
    padding: 12px;
  }
  .new-chat-btn i {
    margin: 0;
  }
}
</style>