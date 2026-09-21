<template>
  <div class="vue-floating-chatbot-root">
    <!-- 플로팅 런처 버튼 (우측 하단) -->
    <button 
      class="floating-launcher-btn" 
      :class="{ 'is-active': isOpen }" 
      @click="toggleChat"
      aria-label="챗봇 열기/닫기"
    >
      <div v-if="!isOpen && unreadCount > 0" class="floating-launcher-badge">{{ unreadCount }}</div>
      
      <svg v-if="!isOpen" class="launcher-icon" width="28" height="28" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"></path>
        <circle cx="9" cy="10" r="1.2" fill="currentColor"></circle>
        <circle cx="15" cy="10" r="1.2" fill="currentColor"></circle>
      </svg>
      <svg v-else class="launcher-icon" width="26" height="26" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
        <line x1="18" y1="6" x2="6" y2="18"></line>
        <line x1="6" y1="6" x2="18" y2="18"></line>
      </svg>
    </button>

    <!-- 드래그 가능한 플로팅 챗봇 대화창 (v-show 기반 상태 보존) -->
    <div 
      v-show="isOpen" 
      ref="chatWindow"
      class="chatbot-window"
      :style="{ top: windowPosition.top + 'px', left: windowPosition.left + 'px' }"
    >
      <!-- 상단 드래그 헤더 -->
      <div 
        class="chat-header" 
        @mousedown="startDrag"
        @touchstart="startTouchDrag"
      >
        <div class="header-left">
          <div class="header-avatar">{{ botAvatar }}</div>
          <div class="header-title-wrap">
            <div class="header-title">
              <span>{{ botName }}</span>
              <span class="online-indicator"></span>
              <span class="drag-hint">드래그하여 이동</span>
            </div>
            <div class="header-subtitle">{{ botStatusText }}</div>
          </div>
        </div>

        <div class="header-actions" @mousedown.stop @touchstart.stop>
          <!-- 닫기가 아닌 숨기기 버튼 (상태 보존) -->
          <button 
            class="header-btn hide-btn" 
            @click="hideChat" 
            title="대화창 숨기기 (상태 유지)"
          >
            <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">
              <line x1="5" y1="12" x2="19" y2="12"></line>
            </svg>
          </button>
          <!-- 원래 위치로 리셋 버튼 -->
          <button 
            class="header-btn" 
            @click="resetPosition" 
            title="창 위치 초기화"
          >
            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <path d="M3 12a9 9 0 0 1 15-6.7L21 8"></path>
              <path d="M21 3v5h-5"></path>
              <path d="M21 12a9 9 0 0 1-15 6.7L3 16"></path>
              <path d="M3 21v-5h5"></path>
            </svg>
          </button>
        </div>
      </div>

      <!-- 본체: 좌측 채팅 이력 목록 + 우측 대화 화면 -->
      <div class="chat-body">
        <!-- 좌측: 이전 채팅 이력 사이드바 -->
        <aside class="chat-sidebar">
          <div class="sidebar-header">
            <span class="sidebar-title">채팅 이력</span>
            <button class="btn-new-chat" @click="createNewSession">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
                <line x1="12" y1="5" x2="12" y2="19"></line>
                <line x1="5" y1="12" x2="19" y2="12"></line>
              </svg>
              새 대화
            </button>
          </div>

          <ul class="history-list">
            <li 
              v-for="session in sessions" 
              :key="session.id"
              class="history-item"
              :class="{ active: currentSessionId === session.id }"
              @click="switchSession(session.id)"
            >
              <div class="history-item-top">
                <span class="history-title">{{ session.title }}</span>
                <span class="history-time">{{ session.updatedAt }}</span>
              </div>
              <div class="history-snippet">
                {{ getLastMessagePreview(session) }}
              </div>
            </li>
          </ul>
        </aside>

        <!-- 우측: 챗봇 대화 화면 -->
        <section class="chat-main">
          <!-- 현재 선택된 세션 정보 바 -->
          <div class="session-info-bar">
            <div class="session-topic">
              <span>{{ currentSession.title }}</span>
              <span class="topic-pill">{{ currentSession.category || '일반 상담' }}</span>
            </div>
            <div>
              <span>메시지 {{ currentSession.messages.length }}개</span>
            </div>
          </div>

          <!-- 메시지 목록 스크롤 컨테이너 -->
          <div class="message-container" ref="messageContainer">
            <div class="date-divider">
              <span>오늘</span>
            </div>

            <div 
              v-for="(msg, index) in currentSession.messages" 
              :key="index"
              class="message-row"
              :class="msg.sender"
            >
              <div class="msg-avatar">
                <span v-if="msg.sender === 'bot'">{{ botAvatar }}</span>
                <span v-else>👤</span>
              </div>

              <div class="msg-content-wrap">
                <span class="msg-sender">{{ msg.sender === 'bot' ? botName : '나' }}</span>
                
                <div class="msg-bubble">
                  {{ msg.text }}

                  <!-- 템플릿 카드 -->
                  <div v-if="msg.templateCard" class="template-card">
                    <div class="template-card-title">{{ msg.templateCard.title }}</div>
                    <div class="template-card-desc">{{ msg.templateCard.description }}</div>
                    <div class="template-actions">
                      <button 
                        v-for="(btn, bIdx) in msg.templateCard.actions" 
                        :key="bIdx" 
                        class="template-action-btn"
                        @click="handleTemplateAction(btn)"
                      >
                        {{ btn.label }}
                      </button>
                    </div>
                  </div>
                </div>

                <span class="msg-time">{{ msg.time }}</span>
              </div>
            </div>

            <!-- 챗봇 타이핑 애니메이션 -->
            <div v-if="isTyping" class="message-row bot">
              <div class="msg-avatar">{{ botAvatar }}</div>
              <div class="msg-content-wrap">
                <span class="msg-sender">{{ botName }}</span>
                <div class="typing-indicator">
                  <span class="typing-dot"></span>
                  <span class="typing-dot"></span>
                  <span class="typing-dot"></span>
                </div>
              </div>
            </div>
          </div>

          <!-- 추천 질문 빠른 입력 칩스 -->
          <div class="quick-suggestions">
            <button 
              v-for="(suggestion, sIdx) in quickSuggestions" 
              :key="sIdx"
              class="suggestion-chip"
              @click="sendPresetMessage(suggestion)"
            >
              <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2">
                <polygon points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2"></polygon>
              </svg>
              {{ suggestion }}
            </button>
          </div>

          <!-- 입력창 -->
          <footer class="chat-input-area">
            <div class="input-box-wrapper">
              <input 
                type="text" 
                ref="chatInput"
                class="chat-input"
                v-model="inputMessage" 
                placeholder="메시지를 입력하세요... (Enter를 누르면 전송됩니다)"
                @keydown.enter="handleEnterKey"
              />
              <button 
                class="send-btn" 
                :disabled="!inputMessage.trim() || isTyping"
                @click="sendMessage"
                title="메시지 전송"
              >
                <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">
                  <line x1="22" y1="2" x2="11" y2="13"></line>
                  <polygon points="22 2 15 22 11 13 2 9 22 2"></polygon>
                </svg>
              </button>
            </div>
            <div class="input-sub-hint">
              <span>Shift+Enter 줄바꿈 • Enter 전송</span>
              <span>숨김 시 작성 중인 내용이 보존됩니다</span>
            </div>
          </footer>
        </section>
      </div>
    </div>
  </div>
</template>

<script>
/**
 * Vue 2.5.17 호환 드래그 플로팅 챗봇 컴포넌트
 */
export default {
  name: 'FloatingChatbot',
  props: {
    botName: {
      type: String,
      default: '루나 AI 챗봇'
    },
    botAvatar: {
      type: String,
      default: '🤖'
    },
    botStatusText: {
      type: String,
      default: '무엇이든 물어보세요 • 실시간 자동응답'
    },
    initialOpen: {
      type: Boolean,
      default: false
    }
  },
  data: function() {
    return {
      isOpen: this.initialOpen,
      isTyping: false,
      inputMessage: '',
      unreadCount: 1,
      windowPosition: {
        top: 100,
        left: 300
      },
      dragging: false,
      dragOffset: {
        x: 0,
        y: 0
      },
      quickSuggestions: [
        '기본 사용법 알려줘',
        'API 연동 템플릿 보기',
        '대화 이력 추가 방법',
        '상담원 연결 요청'
      ],
      currentSessionId: 1,
      sessions: [
        {
          id: 1,
          title: '신규 프로젝트 시작 가이드',
          category: '시작하기',
          updatedAt: '14:20',
          messages: [
            {
              sender: 'bot',
              text: '안녕하세요! Vue 2.5.17 환경에 최적화된 스마트 챗봇입니다. 어떤 작업을 도와드릴까요?',
              time: '14:18',
              templateCard: {
                title: '🚀 빠른 시작 템플릿',
                description: '자주 쓰이는 주요 기능 가이드를 확인해보세요.',
                actions: [
                  { label: '컴포넌트 연동 안내', query: '컴포넌트 연동 안내해줘' },
                  { label: '드래그 설정 팁', query: '드래그 설정 팁 알려줘' }
                ]
              }
            },
            {
              sender: 'user',
              text: 'Vue 2.5.17에서 컴포넌트로 바로 임포트해서 쓸 수 있나요?',
              time: '14:19'
            },
            {
              sender: 'bot',
              text: '네, 가능합니다! Options API 기반으로 작성되어 별도의 외부 의존성 없이 src/FloatingChatbot.vue 파일을 바로 import하여 사용할 수 있도록 구성되어 있습니다.',
              time: '14:20'
            }
          ]
        },
        {
          id: 2,
          title: 'REST API 연동 규격 문의',
          category: '기술 지원',
          updatedAt: '어제',
          messages: [
            {
              sender: 'user',
              text: '백엔드 메시지 API 연동 규격이 어떻게 되나요?',
              time: '어제 17:02'
            },
            {
              sender: 'bot',
              text: '표준 JSON 형식으로 { sender: "user"|"bot", text: string, time: string } 규격을 지원합니다.',
              time: '어제 17:03'
            }
          ]
        }
      ]
    };
  },
  computed: {
    currentSession: function() {
      var self = this;
      for (var i = 0; i < this.sessions.length; i++) {
        if (this.sessions[i].id === self.currentSessionId) {
          return this.sessions[i];
        }
      }
      return this.sessions[0];
    }
  },
  mounted: function() {
    this.calculateInitialPosition();
    window.addEventListener('resize', this.onWindowResize);
    window.addEventListener('mousemove', this.onDrag);
    window.addEventListener('mouseup', this.stopDrag);
    window.addEventListener('touchmove', this.onTouchDrag, { passive: false });
    window.addEventListener('touchend', this.stopDrag);
  },
  beforeDestroy: function() {
    window.removeEventListener('resize', this.onWindowResize);
    window.removeEventListener('mousemove', this.onDrag);
    window.removeEventListener('mouseup', this.stopDrag);
    window.removeEventListener('touchmove', this.onTouchDrag);
    window.removeEventListener('touchend', this.stopDrag);
  },
  methods: {
    calculateInitialPosition: function() {
      var screenWidth = window.innerWidth;
      var screenHeight = window.innerHeight;
      var winWidth = 860;
      var winHeight = 600;

      var left = Math.max(20, screenWidth - winWidth - 40);
      var top = Math.max(40, (screenHeight - winHeight) / 2);

      this.windowPosition = {
        top: top,
        left: left
      };
    },
    resetPosition: function() {
      this.calculateInitialPosition();
    },
    onWindowResize: function() {
      var winEl = this.$refs.chatWindow;
      var winWidth = winEl ? winEl.offsetWidth : 860;
      var winHeight = winEl ? winEl.offsetHeight : 600;

      var maxLeft = window.innerWidth - winWidth - 10;
      var maxTop = window.innerHeight - winHeight - 10;

      this.windowPosition.left = Math.min(Math.max(10, this.windowPosition.left), Math.max(10, maxLeft));
      this.windowPosition.top = Math.min(Math.max(10, this.windowPosition.top), Math.max(10, maxTop));
    },
    startDrag: function(e) {
      if (e.button !== 0) return;
      this.dragging = true;
      this.dragOffset.x = e.clientX - this.windowPosition.left;
      this.dragOffset.y = e.clientY - this.windowPosition.top;
    },
    onDrag: function(e) {
      if (!this.dragging) return;
      var newLeft = e.clientX - this.dragOffset.x;
      var newTop = e.clientY - this.dragOffset.y;

      var winEl = this.$refs.chatWindow;
      var winWidth = winEl ? winEl.offsetWidth : 860;
      var winHeight = winEl ? winEl.offsetHeight : 600;

      var maxLeft = window.innerWidth - winWidth - 10;
      var maxTop = window.innerHeight - winHeight - 10;

      this.windowPosition.left = Math.min(Math.max(10, newLeft), maxLeft);
      this.windowPosition.top = Math.min(Math.max(10, newTop), maxTop);
    },
    startTouchDrag: function(e) {
      if (e.touches.length !== 1) return;
      this.dragging = true;
      var touch = e.touches[0];
      this.dragOffset.x = touch.clientX - this.windowPosition.left;
      this.dragOffset.y = touch.clientY - this.windowPosition.top;
    },
    onTouchDrag: function(e) {
      if (!this.dragging) return;
      e.preventDefault();
      var touch = e.touches[0];
      var newLeft = touch.clientX - this.dragOffset.x;
      var newTop = touch.clientY - this.dragOffset.y;

      var winEl = this.$refs.chatWindow;
      var winWidth = winEl ? winEl.offsetWidth : 860;
      var winHeight = winEl ? winEl.offsetHeight : 600;

      var maxLeft = window.innerWidth - winWidth - 10;
      var maxTop = window.innerHeight - winHeight - 10;

      this.windowPosition.left = Math.min(Math.max(10, newLeft), maxLeft);
      this.windowPosition.top = Math.min(Math.max(10, newTop), maxTop);
    },
    stopDrag: function() {
      this.dragging = false;
    },
    toggleChat: function() {
      this.isOpen = !this.isOpen;
      if (this.isOpen) {
        this.unreadCount = 0;
        var self = this;
        this.$nextTick(function() {
          self.scrollToBottom();
          if (self.$refs.chatInput) self.$refs.chatInput.focus();
        });
      }
    },
    hideChat: function() {
      this.isOpen = false;
      this.$emit('hide');
    },
    switchSession: function(sessionId) {
      this.currentSessionId = sessionId;
      var self = this;
      this.$nextTick(function() {
        self.scrollToBottom();
      });
    },
    createNewSession: function() {
      var newId = Date.now();
      var newSession = {
        id: newId,
        title: '새로운 대화 ' + (this.sessions.length + 1),
        category: '일반 문의',
        updatedAt: '방금',
        messages: [
          {
            sender: 'bot',
            text: '새로운 대화가 시작되었습니다. 무엇이든 편하게 말씀해 주세요!',
            time: this.getCurrentTimeString()
          }
        ]
      };
      this.sessions.unshift(newSession);
      this.currentSessionId = newId;
      var self = this;
      this.$nextTick(function() {
        self.scrollToBottom();
        if (self.$refs.chatInput) self.$refs.chatInput.focus();
      });
    },
    getLastMessagePreview: function(session) {
      if (!session.messages || session.messages.length === 0) return '메시지 없음';
      return session.messages[session.messages.length - 1].text;
    },
    getCurrentTimeString: function() {
      var now = new Date();
      var hours = now.getHours();
      var minutes = now.getMinutes();
      var formattedMinutes = minutes < 10 ? '0' + minutes : minutes;
      return hours + ':' + formattedMinutes;
    },
    scrollToBottom: function() {
      var container = this.$refs.messageContainer;
      if (container) {
        container.scrollTop = container.scrollHeight;
      }
    },
    handleEnterKey: function(e) {
      if (!e.shiftKey) {
        e.preventDefault();
        this.sendMessage();
      }
    },
    sendPresetMessage: function(presetText) {
      this.inputMessage = presetText;
      this.sendMessage();
    },
    handleTemplateAction: function(action) {
      this.inputMessage = action.query || action.label;
      this.sendMessage();
    },
    sendMessage: function() {
      var text = this.inputMessage.trim();
      if (!text || this.isTyping) return;

      var timeString = this.getCurrentTimeString();
      this.currentSession.messages.push({
        sender: 'user',
        text: text,
        time: timeString
      });
      this.currentSession.updatedAt = timeString;
      this.inputMessage = '';

      var self = this;
      this.$nextTick(function() {
        self.scrollToBottom();
      });

      this.isTyping = true;
      setTimeout(function() {
        self.isTyping = false;
        self.currentSession.messages.push({
          sender: 'bot',
          text: '입력해 주신 "' + text + '"에 대한 답변입니다. 백엔드 API 연동을 위해 자유롭게 수정하실 수 있습니다.',
          time: self.getCurrentTimeString()
        });
        self.$nextTick(function() {
          self.scrollToBottom();
        });
      }, 800);
    }
  }
};
</script>

<style scoped>
.vue-floating-chatbot-root {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
  color: #1e293b;
}

/* 플로팅 런처 버튼 */
.floating-launcher-btn {
  position: fixed;
  bottom: 28px;
  right: 28px;
  width: 64px;
  height: 64px;
  border-radius: 50%;
  background: linear-gradient(135deg, #4f46e5 0%, #7c3aed 100%);
  color: #ffffff;
  border: none;
  outline: none;
  cursor: pointer;
  box-shadow: 0 10px 25px -3px rgba(79, 70, 229, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9998;
  transition: transform 0.25s cubic-bezier(0.34, 1.56, 0.64, 1);
}

.floating-launcher-btn:hover {
  transform: scale(1.08) translateY(-2px);
}

.floating-launcher-badge {
  position: absolute;
  top: -2px;
  right: -2px;
  background: #ef4444;
  color: white;
  font-size: 11px;
  font-weight: 700;
  height: 20px;
  min-width: 20px;
  padding: 0 6px;
  border-radius: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 2px solid #ffffff;
}

/* 챗봇 윈도우 */
.chatbot-window {
  position: fixed;
  width: 860px;
  height: 600px;
  min-width: 480px;
  min-height: 400px;
  max-width: calc(100vw - 32px);
  max-height: calc(100vh - 32px);
  background: #ffffff;
  border-radius: 20px;
  box-shadow: 0 24px 60px -12px rgba(15, 23, 42, 0.35), 0 0 0 1px rgba(0, 0, 0, 0.06);
  display: flex;
  flex-direction: column;
  z-index: 9999;
  overflow: hidden;
  user-select: none;
}

.chat-header {
  height: 58px;
  background: #ffffff;
  border-bottom: 1px solid #e2e8f0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 18px;
  cursor: grab;
  flex-shrink: 0;
}

.chat-header:active {
  cursor: grabbing;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 12px;
  pointer-events: none;
}

.header-avatar {
  width: 36px;
  height: 36px;
  border-radius: 10px;
  background: linear-gradient(135deg, #4f46e5, #818cf8);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 16px;
}

.header-title-wrap {
  display: flex;
  flex-direction: column;
}

.header-title {
  font-size: 15px;
  font-weight: 700;
  color: #1e293b;
  display: flex;
  align-items: center;
  gap: 6px;
}

.online-indicator {
  width: 8px;
  height: 8px;
  background: #10b981;
  border-radius: 50%;
  display: inline-block;
}

.drag-hint {
  font-size: 11px;
  color: #94a3b8;
  background: #f1f5f9;
  padding: 2px 8px;
  border-radius: 4px;
  font-weight: 500;
}

.header-subtitle {
  font-size: 12px;
  color: #64748b;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 6px;
}

.header-btn {
  width: 34px;
  height: 34px;
  border-radius: 8px;
  border: none;
  background: transparent;
  color: #64748b;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}

.header-btn:hover {
  background: #f1f5f9;
  color: #1e293b;
}

.header-btn.hide-btn:hover {
  background: #e0e7ff;
  color: #4f46e5;
}

.chat-body {
  display: flex;
  flex: 1;
  overflow: hidden;
  background: #ffffff;
  user-select: text;
}

.chat-sidebar {
  width: 270px;
  background: #f8fafc;
  border-right: 1px solid #e2e8f0;
  display: flex;
  flex-direction: column;
  flex-shrink: 0;
}

.sidebar-header {
  padding: 14px 16px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  border-bottom: 1px solid #e2e8f0;
}

.sidebar-title {
  font-size: 13px;
  font-weight: 700;
  color: #64748b;
  text-transform: uppercase;
}

.btn-new-chat {
  background: #ffffff;
  color: #4f46e5;
  border: 1px solid #c7d2fe;
  padding: 6px 10px;
  border-radius: 8px;
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 4px;
}

.btn-new-chat:hover {
  background: #eef2ff;
}

.history-list {
  flex: 1;
  overflow-y: auto;
  padding: 10px 8px;
  list-style: none;
}

.history-item {
  padding: 10px 12px;
  border-radius: 12px;
  margin-bottom: 6px;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  gap: 4px;
  border: 1px solid transparent;
}

.history-item:hover {
  background: #f1f5f9;
}

.history-item.active {
  background: #ffffff;
  border-color: #c7d2fe;
  box-shadow: 0 2px 6px rgba(79, 70, 229, 0.08);
}

.history-item-top {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.history-title {
  font-size: 13.5px;
  font-weight: 600;
  color: #1e293b;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 170px;
}

.history-item.active .history-title {
  color: #4f46e5;
}

.history-time {
  font-size: 11px;
  color: #94a3b8;
}

.history-snippet {
  font-size: 12px;
  color: #64748b;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.chat-main {
  flex: 1;
  display: flex;
  flex-direction: column;
  background: #ffffff;
}

.session-info-bar {
  padding: 8px 18px;
  background: #f8fafc;
  border-bottom: 1px solid #e2e8f0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 12.5px;
  color: #64748b;
}

.session-topic {
  display: flex;
  align-items: center;
  gap: 6px;
  font-weight: 600;
  color: #1e293b;
}

.topic-pill {
  background: #eef2ff;
  color: #4f46e5;
  padding: 2px 8px;
  border-radius: 9999px;
  font-size: 11px;
}

.message-container {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  display: flex;
  flex-direction: column;
  gap: 18px;
}

.date-divider {
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 8px 0;
}

.date-divider span {
  font-size: 11.5px;
  color: #94a3b8;
  background: #f1f5f9;
  padding: 4px 12px;
  border-radius: 12px;
}

.message-row {
  display: flex;
  gap: 12px;
  max-width: 82%;
}

.message-row.bot {
  align-self: flex-start;
}

.message-row.user {
  align-self: flex-end;
  flex-direction: row-reverse;
}

.msg-avatar {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
}

.bot .msg-avatar {
  background: linear-gradient(135deg, #4f46e5, #818cf8);
  color: white;
}

.user .msg-avatar {
  background: #475569;
  color: white;
}

.msg-content-wrap {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.user .msg-content-wrap {
  align-items: flex-end;
}

.msg-sender {
  font-size: 12px;
  font-weight: 600;
  color: #64748b;
}

.msg-bubble {
  padding: 12px 16px;
  border-radius: 16px;
  font-size: 14px;
  line-height: 1.55;
  word-break: break-word;
}

.bot .msg-bubble {
  background: #f1f5f9;
  color: #1e293b;
  border-top-left-radius: 4px;
}

.user .msg-bubble {
  background: linear-gradient(135deg, #4f46e5 0%, #6366f1 100%);
  color: #ffffff;
  border-top-right-radius: 4px;
}

.msg-time {
  font-size: 10.5px;
  color: #94a3b8;
  margin-top: 2px;
}

.template-card {
  margin-top: 10px;
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  padding: 12px;
}

.template-card-title {
  font-size: 13px;
  font-weight: 700;
  color: #1e293b;
  margin-bottom: 6px;
}

.template-card-desc {
  font-size: 12px;
  color: #64748b;
  line-height: 1.45;
  margin-bottom: 10px;
}

.template-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.template-action-btn {
  background: #eef2ff;
  color: #4f46e5;
  border: 1px solid #c7d2fe;
  border-radius: 6px;
  padding: 5px 10px;
  font-size: 11.5px;
  font-weight: 600;
  cursor: pointer;
}

.quick-suggestions {
  padding: 8px 18px;
  display: flex;
  align-items: center;
  gap: 8px;
  overflow-x: auto;
  white-space: nowrap;
  border-top: 1px solid #e2e8f0;
  background: #fafafa;
}

.suggestion-chip {
  background: #ffffff;
  border: 1px solid #e2e8f0;
  padding: 6px 12px;
  border-radius: 20px;
  font-size: 12px;
  color: #1e293b;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 4px;
  flex-shrink: 0;
}

.typing-indicator {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 10px 14px;
  background: #f1f5f9;
  border-radius: 16px;
  border-top-left-radius: 4px;
  width: fit-content;
}

.typing-dot {
  width: 6px;
  height: 6px;
  background: #94a3b8;
  border-radius: 50%;
  animation: typingBlink 1.4s infinite both;
}
.typing-dot:nth-child(2) { animation-delay: 0.2s; }
.typing-dot:nth-child(3) { animation-delay: 0.4s; }

@keyframes typingBlink {
  0%, 80%, 100% { opacity: 0.3; transform: scale(0.8); }
  40% { opacity: 1; transform: scale(1.1); }
}

.chat-input-area {
  padding: 14px 18px;
  background: #ffffff;
  border-top: 1px solid #e2e8f0;
}

.input-box-wrapper {
  display: flex;
  align-items: center;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 16px;
  padding: 4px 6px 4px 14px;
}

.chat-input {
  flex: 1;
  border: none;
  outline: none;
  background: transparent;
  font-size: 14px;
  color: #1e293b;
  padding: 8px 0;
}

.send-btn {
  width: 38px;
  height: 38px;
  border-radius: 10px;
  background: #4f46e5;
  color: #ffffff;
  border: none;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}

.send-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

.input-sub-hint {
  display: flex;
  justify-content: space-between;
  margin-top: 6px;
  font-size: 11px;
  color: #94a3b8;
  padding: 0 4px;
}
</style>
