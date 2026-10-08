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
      :class="{ 'is-maximized': isMaximized }"
      :style="windowStyle"
    >
      <!-- 상단 드래그 헤더 (더블클릭 시 최대화 토글) -->
      <div 
        class="chat-header" 
        @mousedown="startDrag"
        @touchstart="startTouchDrag"
        @dblclick="toggleMaximize"
      >
        <div class="header-left">
          <div class="header-avatar">{{ botAvatar }}</div>
          <div class="header-title-wrap">
            <div class="header-title">
              <span>{{ botName }}</span>
              <span class="online-indicator"></span>
              <span class="drag-hint">{{ isMaximized ? '화면 최대화됨' : '드래그하여 이동' }}</span>
            </div>
            <div class="header-subtitle">{{ botStatusText }}</div>
          </div>
        </div>

        <div class="header-actions" @mousedown.stop @touchstart.stop>
          <!-- 사이드바 열기/닫기 토글 버튼 -->
          <button 
            class="header-btn" 
            :class="{ 'is-active': isSidebarOpen }"
            @click="toggleSidebar" 
            :title="isSidebarOpen ? '채팅 이력 접기' : '채팅 이력 펼치기'"
          >
            <svg width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect>
              <line x1="9" y1="3" x2="9" y2="21"></line>
            </svg>
          </button>
          <!-- 화면 최대화 / 기본사이즈 토글 버튼 -->
          <button 
            class="header-btn maximize-btn" 
            :class="{ 'is-active': isMaximized }"
            @click="toggleMaximize" 
            :title="isMaximized ? '기본 크기로 복원' : '화면 최대화'"
            :aria-label="isMaximized ? '기본 크기로 복원' : '화면 최대화'"
          >
            <!-- 기본 크기일 때: 최대화 아이콘 -->
            <svg v-if="!isMaximized" width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect>
            </svg>
            <!-- 최대화 상태일 때: 복원 아이콘 -->
            <svg v-else width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <rect x="7" y="7" width="14" height="14" rx="1.5" ry="1.5"></rect>
              <path d="M4 17V5a2 2 0 0 1 2-2h12"></path>
            </svg>
          </button>
          <!-- 원래 위치로 리셋 버튼 -->
          <button 
            class="header-btn" 
            @click="resetPosition" 
            title="창 크기 및 위치 초기화"
          >
            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <path d="M3 12a9 9 0 0 1 15-6.7L21 8"></path>
              <path d="M21 3v5h-5"></path>
              <path d="M21 12a9 9 0 0 1-15 6.7L3 16"></path>
              <path d="M3 21v-5h5"></path>
            </svg>
          </button>
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
        </div>
      </div>

      <!-- 본체: 좌측 채팅 이력 목록 + 우측 대화 화면 -->
      <div class="chat-body" :class="{ 'sidebar-collapsed': !isSidebarOpen }">
        <!-- 좌측: 이전 채팅 이력 사이드바 -->
        <aside class="chat-sidebar" :class="{ 'collapsed': !isSidebarOpen }">
          <div class="sidebar-header">
            <span class="sidebar-title">채팅 이력</span>
            <div class="sidebar-header-actions">
              <button class="btn-new-chat" @click="createNewSession" title="새 대화 시작">
                <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
                  <line x1="12" y1="5" x2="12" y2="19"></line>
                  <line x1="5" y1="12" x2="19" y2="12"></line>
                </svg>
                새 대화
              </button>
              <button class="btn-sidebar-collapse" @click="toggleSidebar" title="채팅 이력 접기">
                <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">
                  <polyline points="15 18 9 12 15 6"></polyline>
                </svg>
              </button>
            </div>
          </div>

          <ul class="history-list">
            <li 
              v-for="session in sessions" 
              :key="session.id"
              class="history-item"
              :class="{ active: currentSessionId === session.id, 'is-editing': editingSessionId === session.id }"
              @click="switchSession(session.id)"
            >
              <!-- 인라인 제목 수정 모드 -->
              <div v-if="editingSessionId === session.id" class="history-edit-box" @click.stop>
                <input 
                  type="text" 
                  class="history-edit-input" 
                  v-model="editingTitle" 
                  @keydown.enter.prevent="saveSessionTitle(session.id)"
                  @keydown.esc.prevent="cancelEditSessionTitle"
                  ref="editTitleInput"
                  placeholder="대화 제목 입력"
                />
                <div class="history-edit-btns">
                  <button class="btn-history-ctrl btn-save" @click="saveSessionTitle(session.id)" title="저장">
                    <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.8">
                      <polyline points="20 6 9 17 4 12"></polyline>
                    </svg>
                  </button>
                  <button class="btn-history-ctrl btn-cancel" @click="cancelEditSessionTitle" title="취소">
                    <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.8">
                      <line x1="18" y1="6" x2="6" y2="18"></line>
                      <line x1="6" y1="6" x2="18" y2="18"></line>
                    </svg>
                  </button>
                </div>
              </div>

              <!-- 일반 표시 모드 -->
              <template v-else>
                <div class="history-item-top">
                  <span class="history-title" :title="session.title">{{ session.title }}</span>
                  <div class="history-meta-wrap">
                    <span class="history-time">{{ session.updatedAt }}</span>
                    <div class="history-action-btns" @click.stop>
                      <button 
                        class="btn-history-ctrl btn-edit" 
                        @click="startEditSessionTitle(session)" 
                        title="제목 수정"
                      >
                        <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                          <path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"></path>
                          <path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z"></path>
                        </svg>
                      </button>
                      <button 
                        class="btn-history-ctrl btn-delete" 
                        @click="deleteSession(session.id)" 
                        title="대화 삭제"
                      >
                        <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                          <polyline points="3 6 5 6 21 6"></polyline>
                          <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"></path>
                        </svg>
                      </button>
                    </div>
                  </div>
                </div>
                <div class="history-snippet">
                  {{ getLastMessagePreview(session) }}
                </div>
              </template>
            </li>
          </ul>
        </aside>

        <!-- 우측: 챗봇 대화 화면 -->
        <section class="chat-main">
          <!-- 현재 선택된 세션 정보 바 -->
          <div class="session-info-bar">
            <div class="session-topic">
              <button 
                class="btn-sidebar-toggle" 
                @click="toggleSidebar" 
                :class="{ 'is-collapsed': !isSidebarOpen }"
                :title="isSidebarOpen ? '채팅 이력 접기' : '채팅 이력 펼치기'"
                :aria-label="isSidebarOpen ? '채팅 이력 접기' : '채팅 이력 펼치기'"
              >
                <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">
                  <rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect>
                  <line x1="9" y1="3" x2="9" y2="21"></line>
                  <path v-if="!isSidebarOpen" d="M13 12h5m-2-2l2 2-2 2"></path>
                </svg>
              </button>
              <span>{{ currentSession.title }}</span>
              <span class="topic-pill">{{ currentSession.category || '일반 상담' }}</span>
            </div>
            <div>
              <span>메시지 {{ currentSession.messages.length }}개</span>
            </div>
          </div>

          <!-- 메시지 목록 스크롤 컨테이너 -->
          <div class="message-container" ref="messageContainer">
            <!-- 처음 진입 및 메시지가 없을 때: 중앙 추천 질문 리스트 + 유의사항 화면 -->
            <div v-if="currentSession.messages.length === 0" class="welcome-container">
              <div class="welcome-header">
                <div class="welcome-avatar">{{ botAvatar }}</div>
                <h2 class="welcome-title">{{ botName }}에 오신 것을 환영합니다!</h2>
                <p class="welcome-subtitle">궁금한 내용을 아래 추천 질문에서 선택하거나 직접 입력하여 대화를 시작해 보세요.</p>
              </div>

              <!-- 중앙 추천 질문 리스트 -->
              <div class="welcome-suggestions-section">
                <div class="welcome-section-label">
                  <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <polygon points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2"></polygon>
                  </svg>
                  자주 묻는 추천 질문
                </div>
                <div class="welcome-suggestions-grid">
                  <button 
                    v-for="(item, sIdx) in welcomeSuggestions" 
                    :key="sIdx"
                    class="welcome-suggestion-card"
                    @click="sendPresetMessage(item.query || item.title)"
                  >
                    <div class="suggestion-card-icon">{{ item.icon || '💬' }}</div>
                    <div class="suggestion-card-content">
                      <div class="suggestion-card-title">{{ item.title }}</div>
                      <div class="suggestion-card-desc">{{ item.desc }}</div>
                    </div>
                    <svg class="suggestion-card-arrow" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                      <line x1="5" y1="12" x2="19" y2="12"></line>
                      <polyline points="12 5 19 12 12 19"></polyline>
                    </svg>
                  </button>
                </div>
              </div>

              <!-- 하단 유의사항 박스 -->
              <div class="welcome-notice-box">
                <div class="notice-header">
                  <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <circle cx="12" cy="12" r="10"></circle>
                    <line x1="12" y1="8" x2="12" y2="12"></line>
                    <line x1="12" y1="16" x2="12.01" y2="16"></line>
                  </svg>
                  <span>서비스 이용 시 유의사항</span>
                </div>
                <ul class="notice-list">
                  <li>AI 답변은 학습된 데이터 기반으로 자동 생성되므로 중요 결정 시 추가 검토가 필요합니다.</li>
                  <li>주민등록번호, 금융정보, 비밀번호 등 민감한 개인정보는 입력하지 마십시오.</li>
                  <li>원활한 서비스 제공 및 품질 개선을 위해 대화 내역이 안전하게 보관될 수 있습니다.</li>
                </ul>
              </div>
            </div>

            <!-- 대화 메시지가 있을 때: 기존 메시지 목록 렌더링 -->
            <template v-else>
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
            </template>
          </div>

          <!-- 추천 질문 빠른 입력 칩스 (대화가 시작된 후 표시) -->
          <div v-if="currentSession.messages.length > 0" class="quick-suggestions">
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

      <!-- 마우스 창 리사이즈 핸들 -->
      <div v-if="!isMaximized" class="resize-handle resize-handle-r" @mousedown.stop.prevent="startResize($event, 'e')"></div>
      <div v-if="!isMaximized" class="resize-handle resize-handle-b" @mousedown.stop.prevent="startResize($event, 's')"></div>
      <div 
        v-if="!isMaximized" 
        class="resize-handle resize-handle-se" 
        @mousedown.stop.prevent="startResize($event, 'se')"
        @touchstart.stop.prevent="startTouchResize($event, 'se')"
        title="드래그하여 창 크기 조절"
      >
        <svg width="10" height="10" viewBox="0 0 10 10" fill="currentColor">
          <circle cx="8" cy="8" r="1.2" />
          <circle cx="8" cy="4.5" r="1.2" />
          <circle cx="4.5" cy="8" r="1.2" />
          <circle cx="8" cy="1" r="1.2" />
          <circle cx="4.5" cy="4.5" r="1.2" />
          <circle cx="1" cy="8" r="1.2" />
        </svg>
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
      isSidebarOpen: true,
      isTyping: false,
      inputMessage: '',
      unreadCount: 1,
      windowPosition: {
        top: 100,
        left: 300
      },
      windowSize: {
        width: 860,
        height: 600
      },
      isMaximized: false,
      preMaximizeState: null,
      resizing: false,
      resizeDirection: '',
      resizeStart: {
        x: 0,
        y: 0,
        width: 860,
        height: 600
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
      welcomeSuggestions: [
        {
          icon: '🚀',
          title: '신규 프로젝트 시작 가이드',
          desc: 'Vue 2.5.17 환경에 플로팅 챗봇을 연동하는 방법 안내',
          query: '신규 프로젝트 시작 가이드 알려줘'
        },
        {
          icon: '📡',
          title: 'REST API 연동 규격 문의',
          desc: '백엔드 메시지 전송 및 응답 JSON 데이터 규격',
          query: '백엔드 REST API 연동 규격 알려줘'
        },
        {
          icon: '🎨',
          title: '테마 및 UI 커스터마이징',
          desc: '브랜드 컬러, 창 크기 및 폰트 커스텀 팁',
          query: '챗봇 UI 스타일과 테마 커스터마이징 방법'
        },
        {
          icon: '💡',
          title: '자주 묻는 질문 FAQ',
          desc: '상태 보존(Minimize) 및 드래그 제약 조건 안내',
          query: '상태 보존 및 드래그 동작 원리 알려줘'
        }
      ],
      currentSessionId: 1,
      editingSessionId: null,
      editingTitle: '',
      sessions: [
        {
          id: 1,
          title: '새로운 상담 시작',
          category: '시작하기',
          updatedAt: '방금',
          messages: []
        },
        {
          id: 2,
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
          id: 3,
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
    windowStyle: function() {
      if (this.isMaximized) {
        return {
          top: '0px',
          left: '0px',
          width: '100vw',
          height: '100vh',
          maxWidth: '100vw',
          maxHeight: '100vh',
          borderRadius: '0px'
        };
      }
      return {
        top: this.windowPosition.top + 'px',
        left: this.windowPosition.left + 'px',
        width: this.windowSize.width + 'px',
        height: this.windowSize.height + 'px'
      };
    },
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
      var winWidth = this.windowSize ? this.windowSize.width : 860;
      var winHeight = this.windowSize ? this.windowSize.height : 600;

      var left = Math.max(20, screenWidth - winWidth - 40);
      var top = Math.max(40, (screenHeight - winHeight) / 2);

      this.windowPosition = {
        top: top,
        left: left
      };
    },
    resetPosition: function() {
      this.isMaximized = false;
      this.windowSize = {
        width: 860,
        height: 600
      };
      this.calculateInitialPosition();
    },
    toggleMaximize: function() {
      if (!this.isMaximized) {
        this.preMaximizeState = {
          top: this.windowPosition.top,
          left: this.windowPosition.left,
          width: this.windowSize.width,
          height: this.windowSize.height
        };
        this.isMaximized = true;
      } else {
        this.isMaximized = false;
        if (this.preMaximizeState) {
          this.windowPosition.top = this.preMaximizeState.top;
          this.windowPosition.left = this.preMaximizeState.left;
          this.windowSize.width = this.preMaximizeState.width;
          this.windowSize.height = this.preMaximizeState.height;
        }
      }
    },
    startResize: function(e, direction) {
      if (e.button !== 0 || this.isMaximized) return;
      this.resizing = true;
      this.resizeDirection = direction || 'se';
      this.resizeStart = {
        x: e.clientX,
        y: e.clientY,
        width: this.windowSize.width,
        height: this.windowSize.height
      };
    },
    startTouchResize: function(e, direction) {
      if (e.touches.length !== 1 || this.isMaximized) return;
      this.resizing = true;
      this.resizeDirection = direction || 'se';
      var touch = e.touches[0];
      this.resizeStart = {
        x: touch.clientX,
        y: touch.clientY,
        width: this.windowSize.width,
        height: this.windowSize.height
      };
    },
    handleResizeMove: function(clientX, clientY) {
      var minWidth = 420;
      var minHeight = 380;
      var maxWidth = window.innerWidth - this.windowPosition.left - 10;
      var maxHeight = window.innerHeight - this.windowPosition.top - 10;

      var deltaX = clientX - this.resizeStart.x;
      var deltaY = clientY - this.resizeStart.y;

      if (this.resizeDirection === 'se' || this.resizeDirection === 'e') {
        var newWidth = this.resizeStart.width + deltaX;
        this.windowSize.width = Math.max(minWidth, Math.min(newWidth, Math.max(minWidth, maxWidth)));
      }
      if (this.resizeDirection === 'se' || this.resizeDirection === 's') {
        var newHeight = this.resizeStart.height + deltaY;
        this.windowSize.height = Math.max(minHeight, Math.min(newHeight, Math.max(minHeight, maxHeight)));
      }
    },
    onWindowResize: function() {
      if (this.isMaximized) return;
      var winWidth = this.windowSize.width;
      var winHeight = this.windowSize.height;

      var maxLeft = window.innerWidth - winWidth - 10;
      var maxTop = window.innerHeight - winHeight - 10;

      this.windowPosition.left = Math.min(Math.max(10, this.windowPosition.left), Math.max(10, maxLeft));
      this.windowPosition.top = Math.min(Math.max(10, this.windowPosition.top), Math.max(10, maxTop));
    },
    startDrag: function(e) {
      if (e.button !== 0 || this.isMaximized) return;
      this.dragging = true;
      this.dragOffset.x = e.clientX - this.windowPosition.left;
      this.dragOffset.y = e.clientY - this.windowPosition.top;
    },
    onDrag: function(e) {
      if (this.resizing) {
        this.handleResizeMove(e.clientX, e.clientY);
        return;
      }
      if (!this.dragging || this.isMaximized) return;
      var newLeft = e.clientX - this.dragOffset.x;
      var newTop = e.clientY - this.dragOffset.y;

      var winWidth = this.windowSize.width;
      var winHeight = this.windowSize.height;

      var maxLeft = window.innerWidth - winWidth - 10;
      var maxTop = window.innerHeight - winHeight - 10;

      this.windowPosition.left = Math.min(Math.max(10, newLeft), maxLeft);
      this.windowPosition.top = Math.min(Math.max(10, newTop), maxTop);
    },
    startTouchDrag: function(e) {
      if (e.touches.length !== 1 || this.isMaximized) return;
      this.dragging = true;
      var touch = e.touches[0];
      this.dragOffset.x = touch.clientX - this.windowPosition.left;
      this.dragOffset.y = touch.clientY - this.windowPosition.top;
    },
    onTouchDrag: function(e) {
      if (this.resizing) {
        e.preventDefault();
        var touch = e.touches[0];
        this.handleResizeMove(touch.clientX, touch.clientY);
        return;
      }
      if (!this.dragging || this.isMaximized) return;
      e.preventDefault();
      var touch = e.touches[0];
      var newLeft = touch.clientX - this.dragOffset.x;
      var newTop = touch.clientY - this.dragOffset.y;

      var winWidth = this.windowSize.width;
      var winHeight = this.windowSize.height;

      var maxLeft = window.innerWidth - winWidth - 10;
      var maxTop = window.innerHeight - winHeight - 10;

      this.windowPosition.left = Math.min(Math.max(10, newLeft), maxLeft);
      this.windowPosition.top = Math.min(Math.max(10, newTop), maxTop);
    },
    stopDrag: function() {
      this.dragging = false;
      this.resizing = false;
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
    toggleSidebar: function() {
      this.isSidebarOpen = !this.isSidebarOpen;
    },
    switchSession: function(sessionId) {
      if (this.editingSessionId) {
        this.cancelEditSessionTitle();
      }
      this.currentSessionId = sessionId;
      var self = this;
      this.$nextTick(function() {
        self.scrollToBottom();
      });
    },
    startEditSessionTitle: function(session) {
      this.editingSessionId = session.id;
      this.editingTitle = session.title;
      var self = this;
      this.$nextTick(function() {
        var inputs = self.$refs.editTitleInput;
        var input = Array.isArray(inputs) ? inputs[0] : inputs;
        if (input) {
          input.focus();
          input.select();
        }
      });
    },
    saveSessionTitle: function(sessionId) {
      var newTitle = this.editingTitle.trim();
      if (!newTitle) return;
      for (var i = 0; i < this.sessions.length; i++) {
        if (this.sessions[i].id === sessionId) {
          this.sessions[i].title = newTitle;
          break;
        }
      }
      this.editingSessionId = null;
      this.editingTitle = '';
    },
    cancelEditSessionTitle: function() {
      this.editingSessionId = null;
      this.editingTitle = '';
    },
    deleteSession: function(sessionId) {
      if (this.sessions.length === 1) {
        if (!confirm('현재 마지막 대화입니다. 대화를 삭제하고 새 대화로 초기화하시겠습니까?')) {
          return;
        }
      } else {
        if (!confirm('이 대화 이력을 삭제하시겠습니까?')) {
          return;
        }
      }

      var targetIndex = -1;
      for (var i = 0; i < this.sessions.length; i++) {
        if (this.sessions[i].id === sessionId) {
          targetIndex = i;
          break;
        }
      }

      if (targetIndex !== -1) {
        this.sessions.splice(targetIndex, 1);
      }

      if (this.sessions.length === 0) {
        this.createNewSession();
        return;
      }

      if (this.currentSessionId === sessionId) {
        var nextIndex = Math.min(targetIndex, this.sessions.length - 1);
        this.currentSessionId = this.sessions[nextIndex].id;
        var self = this;
        this.$nextTick(function() {
          self.scrollToBottom();
        });
      }
    },
    createNewSession: function() {
      var newId = Date.now();
      var newSession = {
        id: newId,
        title: '새로운 대화 ' + (this.sessions.length + 1),
        category: '일반 문의',
        updatedAt: '방금',
        messages: []
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
      if (!session.messages || session.messages.length === 0) return '추천 질문으로 대화를 시작해 보세요';
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

      if (this.currentSession.messages.length === 0) {
        if (this.currentSession.title.indexOf('새로운 대화') === 0 || this.currentSession.title === '새로운 상담 시작') {
          this.currentSession.title = text.length > 16 ? text.substring(0, 16) + '...' : text;
        }
      }

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
  min-width: 420px;
  min-height: 380px;
  max-width: 100vw;
  max-height: 100vh;
  background: #ffffff;
  border-radius: 20px;
  box-shadow: 0 24px 60px -12px rgba(15, 23, 42, 0.35), 0 0 0 1px rgba(0, 0, 0, 0.06);
  display: flex;
  flex-direction: column;
  z-index: 9999;
  overflow: hidden;
  user-select: none;
  transition: border-radius 0.2s ease, box-shadow 0.2s ease;
}

.chatbot-window.is-maximized {
  top: 0 !important;
  left: 0 !important;
  width: 100vw !important;
  height: 100vh !important;
  max-width: 100vw !important;
  max-height: 100vh !important;
  border-radius: 0 !important;
  box-shadow: none !important;
}

.chatbot-window.is-maximized .chat-header {
  cursor: default;
}

.chatbot-window.is-maximized .drag-hint {
  display: none;
}

/* 창 리사이즈 핸들 */
.resize-handle {
  position: absolute;
  z-index: 100;
}

.resize-handle-r {
  top: 0;
  right: 0;
  width: 7px;
  height: 100%;
  cursor: ew-resize;
}

.resize-handle-b {
  bottom: 0;
  left: 0;
  width: 100%;
  height: 7px;
  cursor: ns-resize;
}

.resize-handle-se {
  bottom: 0;
  right: 0;
  width: 18px;
  height: 18px;
  cursor: nwse-resize;
  display: flex;
  align-items: flex-end;
  justify-content: flex-end;
  padding: 3px 4px;
  color: #94a3b8;
  transition: color 0.15s ease;
  user-select: none;
}

.resize-handle-se:hover {
  color: #4f46e5;
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

.header-btn.is-active {
  background: #eef2ff;
  color: #4f46e5;
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
  transition: width 0.28s cubic-bezier(0.4, 0, 0.2, 1), 
              opacity 0.2s ease,
              border-color 0.25s ease;
  overflow: hidden;
  position: relative;
}

.chat-sidebar.collapsed {
  width: 0 !important;
  min-width: 0 !important;
  opacity: 0;
  border-right: 1px solid transparent;
  pointer-events: none;
}

.sidebar-header {
  padding: 14px 16px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  border-bottom: 1px solid #e2e8f0;
  min-width: 270px;
}

.sidebar-title {
  font-size: 13px;
  font-weight: 700;
  color: #64748b;
  text-transform: uppercase;
}

.sidebar-header-actions {
  display: flex;
  align-items: center;
  gap: 6px;
}

.btn-sidebar-collapse {
  width: 28px;
  height: 28px;
  border-radius: 8px;
  border: 1px solid #e2e8f0;
  background: #ffffff;
  color: #64748b;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.15s ease;
}

.btn-sidebar-collapse:hover {
  background: #f1f5f9;
  color: #1e293b;
  border-color: #cbd5e1;
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
  transition: all 0.15s ease;
}

.btn-new-chat:hover {
  background: #eef2ff;
}

.history-list {
  flex: 1;
  overflow-y: auto;
  padding: 10px 8px;
  list-style: none;
  min-width: 270px;
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
  gap: 6px;
  min-height: 22px;
}

.history-title {
  font-size: 13.5px;
  font-weight: 600;
  color: #1e293b;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  flex: 1;
}

.history-item.active .history-title {
  color: #4f46e5;
}

.history-meta-wrap {
  display: flex;
  align-items: center;
  position: relative;
  flex-shrink: 0;
}

.history-time {
  font-size: 11px;
  color: #94a3b8;
  transition: opacity 0.15s ease;
}

.history-action-btns {
  display: none;
  align-items: center;
  gap: 2px;
}

.history-item:hover .history-time,
.history-item.active .history-time {
  display: none;
}

.history-item:hover .history-action-btns,
.history-item.active .history-action-btns {
  display: flex;
}

.btn-history-ctrl {
  width: 22px;
  height: 22px;
  border-radius: 4px;
  border: none;
  background: transparent;
  color: #64748b;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.15s ease;
  padding: 0;
}

.btn-history-ctrl.btn-edit:hover {
  background: #e0e7ff;
  color: #4f46e5;
}

.btn-history-ctrl.btn-delete:hover {
  background: #fee2e2;
  color: #ef4444;
}

/* 인라인 제목 수정 박스 */
.history-edit-box {
  display: flex;
  align-items: center;
  gap: 4px;
  width: 100%;
}

.history-edit-input {
  flex: 1;
  min-width: 0;
  padding: 4px 8px;
  font-size: 12.5px;
  font-weight: 600;
  border: 1.5px solid #4f46e5;
  border-radius: 6px;
  background: #ffffff;
  color: #1e293b;
  outline: none;
}

.history-edit-btns {
  display: flex;
  align-items: center;
  gap: 2px;
}

.btn-history-ctrl.btn-save {
  background: #4f46e5;
  color: #ffffff;
}

.btn-history-ctrl.btn-save:hover {
  background: #4338ca;
}

.btn-history-ctrl.btn-cancel {
  background: #f1f5f9;
  color: #64748b;
}

.btn-history-ctrl.btn-cancel:hover {
  background: #e2e8f0;
  color: #1e293b;
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
  gap: 8px;
  font-weight: 600;
  color: #1e293b;
}

.btn-sidebar-toggle {
  width: 28px;
  height: 28px;
  border-radius: 8px;
  border: 1px solid #e2e8f0;
  background: #ffffff;
  color: #64748b;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.15s ease;
}

.btn-sidebar-toggle:hover {
  background: #eef2ff;
  color: #4f46e5;
  border-color: #c7d2fe;
}

.btn-sidebar-toggle.is-collapsed {
  background: #eef2ff;
  color: #4f46e5;
  border-color: #c7d2fe;
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

/* 중앙 웰컴 화면 (처음 진입 시) */
.welcome-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 24px 16px;
  max-width: 620px;
  margin: auto;
  text-align: center;
  animation: welcomeFadeIn 0.3s ease;
  width: 100%;
}

@keyframes welcomeFadeIn {
  from { opacity: 0; transform: translateY(12px); }
  to { opacity: 1; transform: translateY(0); }
}

.welcome-header {
  margin-bottom: 22px;
}

.welcome-avatar {
  font-size: 38px;
  margin-bottom: 10px;
  display: inline-block;
  filter: drop-shadow(0 4px 12px rgba(79, 70, 229, 0.25));
}

.welcome-title {
  font-size: 19px;
  font-weight: 700;
  color: #1e293b;
  margin-bottom: 6px;
  letter-spacing: -0.5px;
}

.welcome-subtitle {
  font-size: 13px;
  color: #64748b;
  line-height: 1.5;
}

.welcome-suggestions-section {
  width: 100%;
  margin-bottom: 20px;
}

.welcome-section-label {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  font-size: 12px;
  font-weight: 700;
  color: #4f46e5;
  margin-bottom: 12px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.welcome-suggestions-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 10px;
  width: 100%;
}

@media (max-width: 620px) {
  .welcome-suggestions-grid {
    grid-template-columns: 1fr;
  }
}

.welcome-suggestion-card {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 14px;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  cursor: pointer;
  text-align: left;
  transition: all 0.2s ease;
}

.welcome-suggestion-card:hover {
  background: #ffffff;
  border-color: #c7d2fe;
  box-shadow: 0 4px 12px rgba(79, 70, 229, 0.08);
  transform: translateY(-2px);
}

.suggestion-card-icon {
  font-size: 22px;
  flex-shrink: 0;
}

.suggestion-card-content {
  flex: 1;
  min-width: 0;
}

.suggestion-card-title {
  font-size: 13px;
  font-weight: 700;
  color: #1e293b;
  margin-bottom: 3px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.suggestion-card-desc {
  font-size: 11.5px;
  color: #64748b;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.suggestion-card-arrow {
  color: #94a3b8;
  flex-shrink: 0;
  transition: transform 0.2s ease, color 0.2s ease;
}

.welcome-suggestion-card:hover .suggestion-card-arrow {
  color: #4f46e5;
  transform: translateX(3px);
}

/* 유의사항 박스 */
.welcome-notice-box {
  width: 100%;
  background: #fffbeb;
  border: 1px solid #fef3c7;
  border-radius: 12px;
  padding: 12px 16px;
  text-align: left;
}

.notice-header {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  font-weight: 700;
  color: #b45309;
  margin-bottom: 6px;
}

.notice-list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.notice-list li {
  font-size: 11.5px;
  color: #92400e;
  line-height: 1.45;
  position: relative;
  padding-left: 12px;
}

.notice-list li::before {
  content: '•';
  position: absolute;
  left: 2px;
  top: 0;
  color: #d97706;
  font-size: 13px;
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
