<template>
  <div class="vue-floating-chatbot-root" :style="rootStyle">
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
              <button 
                class="btn-sidebar-clear" 
                @click="clearAllSessions" 
                :disabled="isHistoryEmpty"
                title="대화 이력 전체 삭제"
                aria-label="대화 이력 전체 삭제"
              >
                <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                  <polyline points="3 6 5 6 21 6"></polyline>
                  <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"></path>
                </svg>
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

          <!-- 사이드바 하단 전체 삭제 푸터 -->
          <div class="sidebar-footer">
            <button 
              type="button"
              class="btn-clear-all-sessions" 
              @click="clearAllSessions" 
              :disabled="isHistoryEmpty"
              title="대화 이력 전체 삭제"
            >
              <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <polyline points="3 6 5 6 21 6"></polyline>
                <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"></path>
                <line x1="10" y1="11" x2="10" y2="17"></line>
                <line x1="14" y1="11" x2="14" y2="17"></line>
              </svg>
              <span>대화 이력 전체 삭제</span>
            </button>
          </div>
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
                <div class="welcome-suggestions-list">
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
                :key="msg.id || index"
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
                    <!-- 메시지 개별 삭제 버튼 -->
                    <button 
                      type="button" 
                      class="btn-delete-msg" 
                      @click.stop="deleteMessage(index)" 
                      title="메시지 삭제"
                      aria-label="메시지 삭제"
                    >
                      <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                        <polyline points="3 6 5 6 21 6"></polyline>
                        <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"></path>
                      </svg>
                    </button>

                    <!-- 첨부파일 목록 렌더링 (다중 또는 단일) -->
                    <div v-if="msg.attachments && msg.attachments.length > 0" class="msg-attachments-list">
                      <div 
                        v-for="(att, aIdx) in msg.attachments" 
                        :key="aIdx" 
                        class="file-attachment-item"
                      >
                        <img v-if="att.previewUrl" :src="att.previewUrl" class="msg-attachment-thumb" alt="첨부 이미지" />
                        <svg v-else-if="att.isImage || (att.type && att.type.indexOf('image') !== -1)" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                          <rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect>
                          <circle cx="8.5" cy="8.5" r="1.5"></circle>
                          <polyline points="21 15 16 10 5 21"></polyline>
                        </svg>
                        <svg v-else width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                          <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path>
                          <polyline points="14 2 14 8 20 8"></polyline>
                          <line x1="16" y1="13" x2="8" y2="13"></line>
                          <line x1="16" y1="17" x2="8" y2="17"></line>
                        </svg>
                        <div class="file-attachment-info">
                          <span class="file-attachment-name">{{ att.name }}</span>
                          <span class="file-attachment-size">{{ att.sizeStr || att.size }}</span>
                        </div>
                      </div>
                    </div>
                    <!-- 단일 첨부파일 하위 호환 -->
                    <div v-else-if="msg.fileAttachment" class="file-attachment-item">
                      <img v-if="msg.fileAttachment.previewUrl" :src="msg.fileAttachment.previewUrl" class="msg-attachment-thumb" alt="첨부 이미지" />
                      <svg v-else-if="msg.fileAttachment.isImage || (msg.fileAttachment.type && msg.fileAttachment.type.indexOf('image') !== -1)" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                        <rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect>
                        <circle cx="8.5" cy="8.5" r="1.5"></circle>
                        <polyline points="21 15 16 10 5 21"></polyline>
                      </svg>
                      <svg v-else width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                        <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path>
                        <polyline points="14 2 14 8 20 8"></polyline>
                        <line x1="16" y1="13" x2="8" y2="13"></line>
                        <line x1="16" y1="17" x2="8" y2="17"></line>
                      </svg>
                      <div class="file-attachment-info">
                        <span class="file-attachment-name">{{ msg.fileAttachment.name }}</span>
                        <span class="file-attachment-size">{{ msg.fileAttachment.sizeStr || msg.fileAttachment.size }}</span>
                      </div>
                    </div>

                    <!-- 텍스트 메시지 본문 -->
                    <div v-if="msg.text" :class="{ 'msg-text-with-files': (msg.attachments && msg.attachments.length > 0) || msg.fileAttachment }">
                      {{ msg.text }}
                    </div>

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
            <!-- 전송 대기 첨부파일 목록 바 -->
            <div v-if="attachedFiles.length > 0" class="staged-files-bar">
              <div class="staged-files-label">
                <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                  <path d="M21.44 11.05l-9.19 9.19a6 6 0 0 1-8.49-8.49l9.19-9.19a4 4 0 0 1 5.66 5.66l-9.2 9.19a2 2 0 0 1-2.83-2.83l8.49-8.48"></path>
                </svg>
                <span>첨부파일 ({{ attachedFiles.length }})</span>
              </div>
              <div class="staged-files-list">
                <div 
                  v-for="(file, fIdx) in attachedFiles" 
                  :key="file.id || fIdx"
                  class="staged-file-chip"
                  :title="file.name + ' (' + file.sizeStr + ')'"
                >
                  <img v-if="file.previewUrl" :src="file.previewUrl" class="staged-file-thumb" alt="이미지 미리보기" />
                  <svg v-else-if="file.isImage || (file.type && file.type.indexOf('image') !== -1)" width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                    <rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect>
                    <circle cx="8.5" cy="8.5" r="1.5"></circle>
                    <polyline points="21 15 16 10 5 21"></polyline>
                  </svg>
                  <svg v-else width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path>
                    <polyline points="14 2 14 8 20 8"></polyline>
                  </svg>
                  <span class="staged-file-name">{{ file.name }}</span>
                  <span class="staged-file-size">{{ file.sizeStr }}</span>
                  <button 
                    type="button" 
                    class="btn-remove-staged-file" 
                    @click="removeAttachedFile(fIdx)" 
                    title="파일 제거"
                    aria-label="파일 제거"
                  >
                    <svg width="10" height="10" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
                      <line x1="18" y1="6" x2="6" y2="18"></line>
                      <line x1="6" y1="6" x2="18" y2="18"></line>
                    </svg>
                  </button>
                </div>
              </div>
            </div>

            <div class="input-box-wrapper">
              <input 
                type="file" 
                ref="fileInput" 
                class="file-input-hidden" 
                multiple
                @change="handleFileUpload" 
                style="display: none;" 
              />
              <input 
                type="text" 
                ref="chatInput"
                class="chat-input"
                v-model="inputMessage" 
                placeholder="메시지를 입력하세요... (Enter를 누르면 전송됩니다)"
                @keydown.enter="handleEnterKey"
                @paste="handlePaste"
              />
              <button 
                class="upload-btn" 
                @click="triggerFileUpload" 
                title="파일 첨부" 
                type="button"
                :disabled="isTyping"
              >
                <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                  <path d="M21.44 11.05l-9.19 9.19a6 6 0 0 1-8.49-8.49l9.19-9.19a4 4 0 0 1 5.66 5.66l-9.2 9.19a2 2 0 0 1-2.83-2.83l8.49-8.48"></path>
                </svg>
              </button>
              <button 
                class="send-btn" 
                :disabled="(!inputMessage.trim() && attachedFiles.length === 0) || isTyping"
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
              <span>숨김 시 작성 중인 내용과 첨부파일이 보존됩니다</span>
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

    <!-- 숨김 안내 토스트 (숨기기 버튼 클릭 시 상태 보존 안내) -->
    <div v-if="showHideToast" class="hide-toast">
      <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="#10b981" stroke-width="2.5">
        <path d="M20 6L9 17l-5-5"></path>
      </svg>
      <span>대화창이 숨겨졌습니다. 하단 플로팅 버튼으로 언제든 이전 상태 그대로 다시 열 수 있습니다.</span>
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
    },
    enableMockReply: {
      type: Boolean,
      default: true
    },
    primaryColor: {
      type: String,
      default: ''
    }
  },
  data: function() {
    return {
      isOpen: this.initialOpen,
      isSidebarOpen: true,
      isTyping: false,
      inputMessage: '',
      unreadCount: 1,
      attachedFiles: [],
      showHideToast: false,
      hideToastTimer: null,
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
    rootStyle: function() {
      if (this.primaryColor) {
        return {
          '--primary': this.primaryColor
        };
      }
      return {};
    },
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
    },
    isHistoryEmpty: function() {
      if (!this.sessions || this.sessions.length === 0) return true;
      if (this.sessions.length === 1 && (!this.sessions[0].messages || this.sessions[0].messages.length === 0)) {
        return true;
      }
      return false;
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
    clampWindowPosition: function() {
      if (this.isMaximized) return;
      var winWidth = this.windowSize ? this.windowSize.width : 860;
      var winHeight = this.windowSize ? this.windowSize.height : 600;
      var maxLeft = Math.max(0, window.innerWidth - winWidth - 10);
      var maxTop = Math.max(0, window.innerHeight - winHeight - 10);
      this.windowPosition.left = Math.min(Math.max(10, this.windowPosition.left), maxLeft);
      this.windowPosition.top = Math.min(Math.max(10, this.windowPosition.top), maxTop);
    },
    onWindowResize: function() {
      this.clampWindowPosition();
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
      this.showHideToast = true;
      if (this.hideToastTimer) clearTimeout(this.hideToastTimer);
      var self = this;
      this.hideToastTimer = setTimeout(function() {
        self.showHideToast = false;
      }, 3200);
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
      this.$emit('session-change', this.currentSession);
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
    clearAllSessions: function() {
      if (this.isHistoryEmpty) {
        alert('삭제할 대화 이력이 없습니다.');
        return;
      }

      if (!confirm('모든 대화 이력을 삭제하시겠습니까?\n삭제된 모든 대화 내용은 복구할 수 없습니다.')) {
        return;
      }

      var newId = Date.now();
      var freshSession = {
        id: newId,
        title: '새로운 상담 시작',
        category: '시작하기',
        updatedAt: '방금',
        messages: []
      };

      this.sessions = [freshSession];
      this.currentSessionId = newId;
      this.editingSessionId = null;
      this.editingTitle = '';

      this.$emit('sessions-cleared', {
        newSession: freshSession
      });
      this.$emit('session-change', freshSession);

      var self = this;
      this.$nextTick(function() {
        self.scrollToBottom();
        if (self.$refs.chatInput) self.$refs.chatInput.focus();
      });
    },
    deleteMessage: function(index) {
      if (!confirm('이 메시지를 삭제하시겠습니까?')) {
        return;
      }
      var deletedMsg = this.currentSession.messages.splice(index, 1)[0];
      if (this.currentSession.messages.length > 0) {
        this.currentSession.updatedAt = this.currentSession.messages[this.currentSession.messages.length - 1].time;
      } else {
        this.currentSession.updatedAt = this.getCurrentTimeString();
      }
      this.$emit('message-deleted', {
        message: deletedMsg,
        index: index,
        session: this.currentSession
      });
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
      this.$emit('session-change', newSession);
      var self = this;
      this.$nextTick(function() {
        self.scrollToBottom();
        if (self.$refs.chatInput) self.$refs.chatInput.focus();
      });
    },
    getLastMessagePreview: function(session) {
      if (!session.messages || session.messages.length === 0) return '대화 내용이 없습니다';
      var lastMsg = session.messages[session.messages.length - 1];
      if (lastMsg.text) return lastMsg.text;
      if (lastMsg.attachments && lastMsg.attachments.length > 0) {
        return '📎 파일: ' + lastMsg.attachments[0].name + (lastMsg.attachments.length > 1 ? ' 외 ' + (lastMsg.attachments.length - 1) + '개' : '');
      }
      if (lastMsg.fileAttachment) {
        return '📎 파일: ' + lastMsg.fileAttachment.name;
      }
      return '메시지';
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
      var hasFiles = this.attachedFiles && this.attachedFiles.length > 0;
      if ((!text && !hasFiles) || this.isTyping) return;

      var filesToSend = this.attachedFiles.slice();

      // 첫 메시지 전송 시 기본 제목 자동 설정
      if (this.currentSession.messages.length === 0) {
        if (this.currentSession.title.indexOf('새로운 대화') === 0 || this.currentSession.title === '새로운 상담 시작') {
          if (text) {
            this.currentSession.title = text.length > 16 ? text.substring(0, 16) + '...' : text;
          } else if (hasFiles) {
            var firstFileName = filesToSend[0].name;
            var titleSuffix = filesToSend.length > 1 ? ' 외 ' + (filesToSend.length - 1) + '개' : '';
            this.currentSession.title = firstFileName.length > 12 ? firstFileName.substring(0, 12) + '...' + titleSuffix : firstFileName + titleSuffix;
          }
        }
      }

      var timeString = this.getCurrentTimeString();
      this.currentSession.messages.push({
        sender: 'user',
        text: text,
        attachments: filesToSend,
        fileAttachment: filesToSend.length === 1 ? filesToSend[0] : null,
        time: timeString
      });
      this.currentSession.updatedAt = timeString;
      this.inputMessage = '';
      this.attachedFiles = [];

      this.$emit('send-message', {
        text: text,
        attachments: filesToSend,
        session: this.currentSession
      });

      var self = this;
      this.$nextTick(function() {
        self.scrollToBottom();
      });

      if (!this.enableMockReply) {
        return;
      }

      this.isTyping = true;
      setTimeout(function() {
        self.isTyping = false;
        var botReplyText = '';
        var templateCard = null;

        if (filesToSend.length > 0 && text) {
          botReplyText = '첨부해 주신 ' + filesToSend.length + '개 파일(「' + filesToSend.map(function(f) { return f.name; }).join('」, 「') + '」)과 함께 문의하신 내용을 확인했습니다. 분석 후 안내해 드리겠습니다.';
        } else if (filesToSend.length > 0) {
          botReplyText = '첨부해 주신 ' + filesToSend.length + '개 파일(「' + filesToSend.map(function(f) { return f.name; }).join('」, 「') + '」)을(를) 정상 수신했습니다. 추가로 궁금하신 사항이 있으신가요?';
        } else {
          var botResponse = self.generateMockReply(text);
          botReplyText = botResponse.text;
          templateCard = botResponse.templateCard || null;
        }

        self.currentSession.messages.push({
          sender: 'bot',
          text: botReplyText,
          time: self.getCurrentTimeString(),
          templateCard: templateCard
        });
        self.currentSession.updatedAt = self.getCurrentTimeString();
        self.$nextTick(function() {
          self.scrollToBottom();
        });
      }, 800);
    },
    generateMockReply: function(userQuery) {
      if (userQuery.indexOf('사용법') > -1 || userQuery.indexOf('기본') > -1) {
        return {
          text: '챗봇 기본 사용 가이드입니다. 상단 헤더를 마우스로 잡고 드래그하면 화면 어디든 창을 이동할 수 있습니다. 상단의 [-] 버튼을 누르면 작성 중인 내용과 현재 위치가 그대로 보존된 채 숨겨집니다.',
          templateCard: {
            title: '📌 주요 단축키 & 제스처',
            description: 'Enter: 전송 / Shift+Enter: 줄바꿈 / 헤더 드래그: 창 이동',
            actions: [
              { label: 'API 연동 템플릿 보기', query: 'API 연동 템플릿 보기' },
              { label: '이력 추가 방법', query: '대화 이력 추가 방법' }
            ]
          }
        };
      } else if (userQuery.indexOf('API') > -1 || userQuery.indexOf('연동') > -1) {
        return {
          text: '실제 백엔드 LLM 서버와의 통신은 `sendMessage` 메서드 내부에서 axios나 fetch 호출로 간단히 교체할 수 있습니다. 수신된 데이터는 messages 배열에 push하는 구조입니다.',
          templateCard: {
            title: '💻 개발자 연동 팁',
            description: 'WebSocket을 이용한 실시간 스트리밍도 messages 배열의 마지막 항목을 점진적으로 갱신하여 구현 가능합니다.',
            actions: [
              { label: '기본 사용법 알려줘', query: '기본 사용법 알려줘' }
            ]
          }
        };
      } else if (userQuery.indexOf('상담원') > -1) {
        return {
          text: '전문 상담원 연결을 요청하셨습니다. 평일 09:00 ~ 18:00 사이에는 1분 내외로 상담원이 배정됩니다. 현재 대기 인원은 1명입니다.',
          templateCard: {
            title: '👨‍💼 상담원 연결 대기',
            description: '상담 요청 번호: #REQ-2026-0921',
            actions: [
              { label: '요청 취소', query: '상담원 요청 취소할게요' }
            ]
          }
        };
      } else {
        return {
          text: '입력해 주신 "' + userQuery + '"에 대한 안내입니다. 본 화면은 Vue 2.5.17 컴포넌트 템플릿으로 제공되며, 원하시는 비즈니스 로직과 API를 손쉽게 연결할 수 있습니다.',
          templateCard: {
            title: '⚡ 추가 옵션',
            description: '관련된 빠른 액션을 선택하세요.',
            actions: [
              { label: '새 대화 시작', query: '새로운 대화 시작' },
              { label: '도움말 보기', query: '기본 사용법 알려줘' }
            ]
          }
        };
      }
    },
    triggerFileUpload: function() {
      if (this.$refs.fileInput) {
        this.$refs.fileInput.click();
      }
    },
    formatFileSize: function(bytes) {
      if (!bytes || bytes === 0) return '0 B';
      var k = 1024;
      var sizes = ['B', 'KB', 'MB', 'GB'];
      var i = Math.floor(Math.log(bytes) / Math.log(k));
      return parseFloat((bytes / Math.pow(k, i)).toFixed(1)) + ' ' + sizes[i];
    },
    handleFileUpload: function(event) {
      var files = event.target.files;
      if (!files || files.length === 0) return;

      for (var i = 0; i < files.length; i++) {
        var file = files[i];
        var sizeStr = this.formatFileSize(file.size);
        var isImage = file.type ? file.type.indexOf('image') !== -1 : false;
        var previewUrl = (isImage && window.URL && typeof window.URL.createObjectURL === 'function')
          ? window.URL.createObjectURL(file)
          : null;

        this.attachedFiles.push({
          id: Date.now() + '_' + i + '_' + Math.random().toString(36).substr(2, 5),
          name: file.name,
          size: file.size,
          sizeStr: sizeStr,
          type: file.type,
          isImage: isImage,
          previewUrl: previewUrl,
          rawFile: file
        });
      }

      this.$emit('files-staged', {
        files: this.attachedFiles,
        session: this.currentSession,
        source: 'upload'
      });

      event.target.value = '';
      if (this.$refs.chatInput) {
        this.$refs.chatInput.focus();
      }
    },
    handlePaste: function(event) {
      var clipboardData = event.clipboardData || window.clipboardData;
      if (!clipboardData) return;

      var items = clipboardData.items;
      if (!items || items.length === 0) return;

      var imageFiles = [];
      var seenSignatures = {};
      for (var i = 0; i < items.length; i++) {
        var item = items[i];
        if (item.type && item.type.indexOf('image') !== -1) {
          var file = item.getAsFile();
          if (file) {
            var signature = file.size + '_' + file.type + '_' + (file.name || '');
            if (!seenSignatures[signature]) {
              seenSignatures[signature] = true;
              imageFiles.push(file);
            }
          }
        }
      }

      if (imageFiles.length === 0) {
        return;
      }

      event.preventDefault();
      if (typeof event.stopPropagation === 'function') {
        event.stopPropagation();
      }

      var now = new Date();
      var nowTime = now.getTime();
      if (this._lastPasteTime && (nowTime - this._lastPasteTime) < 300) {
        return;
      }
      this._lastPasteTime = nowTime;

      for (var j = 0; j < imageFiles.length; j++) {
        var imgFile = imageFiles[j];
        var fileName = imgFile.name;
        if (!fileName || fileName === 'image.png' || fileName === 'blob') {
          var ext = (imgFile.type && imgFile.type.split('/')[1]) || 'png';
          if (ext === 'jpeg') ext = 'jpg';
          var timeTag = now.getFullYear() +
            ('0' + (now.getMonth() + 1)).slice(-2) +
            ('0' + now.getDate()).slice(-2) + '_' +
            ('0' + now.getHours()).slice(-2) +
            ('0' + now.getMinutes()).slice(-2) +
            ('0' + now.getSeconds()).slice(-2);
          fileName = 'clipboard_image_' + timeTag + (imageFiles.length > 1 ? '_' + (j + 1) : '') + '.' + ext;
        }

        var sizeStr = this.formatFileSize(imgFile.size);
        var previewUrl = (window.URL && typeof window.URL.createObjectURL === 'function') 
          ? window.URL.createObjectURL(imgFile) 
          : null;

        this.attachedFiles.push({
          id: Date.now() + '_' + j + '_' + Math.random().toString(36).substr(2, 5),
          name: fileName,
          size: imgFile.size,
          sizeStr: sizeStr,
          type: imgFile.type || 'image/png',
          isImage: true,
          previewUrl: previewUrl,
          rawFile: imgFile
        });
      }

      this.$emit('files-staged', {
        files: this.attachedFiles,
        session: this.currentSession,
        source: 'clipboard'
      });
    },
    removeAttachedFile: function(index) {
      this.attachedFiles.splice(index, 1);
      this.$emit('file-removed', {
        remainingFiles: this.attachedFiles,
        session: this.currentSession
      });
    }
  }
};
</script>

<style scoped>
.vue-floating-chatbot-root {
  --primary: #4f46e5;
  --primary-hover: #4338ca;
  --primary-light: #eef2ff;
  --primary-border: #c7d2fe;
  --bg-dark: #0f172a;
  --bg-surface: #ffffff;
  --bg-surface-secondary: #f8fafc;
  --bg-surface-tertiary: #f1f5f9;
  --border-subtle: #e2e8f0;
  --border-strong: #cbd5e1;
  --text-main: #1e293b;
  --text-muted: #64748b;
  --text-subtle: #94a3b8;
  --success: #10b981;
  --radius-sm: 8px;
  --radius-md: 12px;
  --radius-lg: 16px;
  --radius-xl: 20px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Noto Sans KR', Helvetica, Arial, sans-serif;
  color: #1e293b;
}

.vue-floating-chatbot-root,
.vue-floating-chatbot-root *,
.vue-floating-chatbot-root *::before,
.vue-floating-chatbot-root *::after {
  box-sizing: border-box;
  -webkit-font-smoothing: antialiased;
}

.vue-floating-chatbot-root button,
.vue-floating-chatbot-root input {
  font-family: inherit;
}

/* 컴포넌트 내부 스크롤바 커스텀 */
.vue-floating-chatbot-root ::-webkit-scrollbar {
  width: 6px;
  height: 6px;
}
.vue-floating-chatbot-root ::-webkit-scrollbar-track {
  background: transparent;
}
.vue-floating-chatbot-root ::-webkit-scrollbar-thumb {
  background: #cbd5e1;
  border-radius: 3px;
}
.vue-floating-chatbot-root ::-webkit-scrollbar-thumb:hover {
  background: #94a3b8;
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
  animation: chatWindowPop 0.22s cubic-bezier(0.16, 1, 0.3, 1);
  transition: border-radius 0.2s ease, box-shadow 0.2s ease;
}

@keyframes chatWindowPop {
  0% {
    opacity: 0;
    transform: scale(0.96) translateY(8px);
  }
  100% {
    opacity: 1;
    transform: scale(1) translateY(0);
  }
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
  width: 6px;
  height: 100%;
  cursor: ew-resize;
}

.resize-handle-b {
  bottom: 0;
  left: 0;
  width: 100%;
  height: 6px;
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

.btn-sidebar-clear {
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

.btn-sidebar-clear:hover:not(:disabled) {
  background: #fee2e2;
  border-color: #fca5a5;
  color: #ef4444;
}

.btn-sidebar-clear:disabled {
  opacity: 0.4;
  cursor: not-allowed;
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

/* 사이드바 하단 전체 삭제 푸터 */
.sidebar-footer {
  padding: 10px 14px;
  border-top: 1px solid #e2e8f0;
  background: #f8fafc;
  min-width: 270px;
}

.btn-clear-all-sessions {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  padding: 8px 12px;
  border-radius: 8px;
  border: 1px dashed #cbd5e1;
  background: #ffffff;
  color: #64748b;
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s ease;
}

.btn-clear-all-sessions:hover:not(:disabled) {
  background: #fee2e2;
  border-color: #fca5a5;
  color: #ef4444;
}

.btn-clear-all-sessions:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  border-style: solid;
  border-color: #e2e8f0;
  background: #f1f5f9;
  color: #94a3b8;
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
  padding: 20px 14px 20px 20px;
  display: flex;
  flex-direction: column;
  gap: 18px;
  margin-right: 6px;
  scrollbar-width: thin;
  scrollbar-color: #cbd5e1 transparent;
}

.chatbot-window.is-maximized .message-container {
  margin-right: 0;
  padding-right: 20px;
}

.message-container::-webkit-scrollbar {
  width: 8px;
}

.message-container::-webkit-scrollbar-track {
  background: transparent;
}

.message-container::-webkit-scrollbar-thumb {
  background: #cbd5e1;
  border-radius: 4px;
}

.message-container::-webkit-scrollbar-thumb:hover {
  background: #64748b;
}

.message-container::-webkit-scrollbar-thumb:active {
  background: #475569;
}

/* 중앙 웰컴 화면 (처음 진입 시) */
.welcome-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 20px 16px;
  max-width: 540px;
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
  margin-bottom: 20px;
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
  margin-bottom: 18px;
}

.welcome-section-label {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  font-size: 12px;
  font-weight: 700;
  color: #4f46e5;
  margin-bottom: 10px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.welcome-suggestions-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  width: 100%;
}

.welcome-suggestion-card {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 10px 16px;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  cursor: pointer;
  text-align: left;
  transition: all 0.2s ease;
  width: 100%;
  box-sizing: border-box;
}

.welcome-suggestion-card:hover {
  background: #ffffff;
  border-color: #c7d2fe;
  box-shadow: 0 4px 12px rgba(79, 70, 229, 0.08);
  transform: translateY(-1px);
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
  margin-bottom: 2px;
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
  position: relative;
  padding: 12px 32px 12px 16px;
  border-radius: 16px;
  font-size: 14px;
  line-height: 1.55;
  word-break: break-word;
  min-width: 58px;
}

/* 메시지 개별 삭제 버튼 */
.btn-delete-msg {
  position: absolute;
  top: 6px;
  right: 6px;
  width: 22px;
  height: 22px;
  border-radius: 6px;
  border: none;
  background: rgba(0, 0, 0, 0.05);
  color: #64748b;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  padding: 0;
  opacity: 0;
  pointer-events: none;
  transition: all 0.15s ease;
  z-index: 2;
}

.message-row:hover .btn-delete-msg {
  opacity: 0.7;
  pointer-events: auto;
}

.btn-delete-msg:hover {
  opacity: 1 !important;
  background: #fee2e2 !important;
  color: #ef4444 !important;
  transform: scale(1.08);
}

.user .btn-delete-msg {
  background: rgba(255, 255, 255, 0.2);
  color: #ffffff;
}

.user .btn-delete-msg:hover {
  background: #fee2e2 !important;
  color: #ef4444 !important;
}

@media (hover: none) {
  .btn-delete-msg {
    opacity: 0.55;
    pointer-events: auto;
  }
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

/* 전송 대기 첨부파일 바 */
.staged-files-bar {
  margin-bottom: 10px;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  padding: 8px 12px;
  display: flex;
  flex-direction: column;
  gap: 6px;
  animation: fadeInDown 0.2s ease;
}

.staged-files-label {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 11px;
  font-weight: 700;
  color: #4f46e5;
  text-transform: uppercase;
  letter-spacing: 0.3px;
}

.staged-files-list {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  max-height: 90px;
  overflow-y: auto;
}

.staged-file-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  background: #ffffff;
  border: 1px solid #cbd5e1;
  border-radius: 20px;
  padding: 3px 6px 3px 10px;
  font-size: 11.5px;
  color: #1e293b;
  max-width: 260px;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04);
}

.staged-file-chip svg {
  color: #64748b;
  flex-shrink: 0;
}

.staged-file-thumb {
  width: 18px;
  height: 18px;
  border-radius: 4px;
  object-fit: cover;
  flex-shrink: 0;
  border: 1px solid rgba(0, 0, 0, 0.1);
}

.staged-file-name {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 140px;
  font-weight: 500;
}

.staged-file-size {
  font-size: 10.5px;
  color: #94a3b8;
  flex-shrink: 0;
}

.btn-remove-staged-file {
  width: 16px;
  height: 16px;
  border-radius: 50%;
  border: none;
  background: #f1f5f9;
  color: #64748b;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  padding: 0;
  flex-shrink: 0;
  transition: all 0.15s ease;
}

.btn-remove-staged-file:hover {
  background: #fee2e2;
  color: #ef4444;
}

/* 말풍선 내부 다중 첨부파일 목록 */
.msg-attachments-list {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin-bottom: 4px;
}

.msg-text-with-files {
  margin-top: 6px;
  padding-top: 6px;
  border-top: 1px solid rgba(255, 255, 255, 0.2);
}

.message-row.user .msg-text-with-files {
  border-top-color: rgba(255, 255, 255, 0.25);
}

.message-row.bot .msg-text-with-files {
  border-top-color: #e2e8f0;
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

.upload-btn {
  width: 36px;
  height: 36px;
  border-radius: 10px;
  background: transparent;
  color: #64748b;
  border: none;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-right: 4px;
  flex-shrink: 0;
  transition: all 0.15s ease;
}

.upload-btn:hover:not(:disabled) {
  background: #e2e8f0;
  color: #4f46e5;
}

.upload-btn:disabled {
  color: #cbd5e1;
  cursor: not-allowed;
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
  flex-shrink: 0;
  transition: background-color 0.15s ease;
}

.send-btn:hover:not(:disabled) {
  background: #4338ca;
}

.send-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

/* 파일 첨부 말풍선 아이템 */
.file-attachment-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 2px 0;
}

.file-attachment-item svg {
  flex-shrink: 0;
}

.msg-attachment-thumb {
  width: 32px;
  height: 32px;
  border-radius: 6px;
  object-fit: cover;
  flex-shrink: 0;
  border: 1px solid rgba(0, 0, 0, 0.1);
}

.file-attachment-info {
  display: flex;
  flex-direction: column;
  min-width: 0;
  text-align: left;
}

.file-attachment-name {
  font-weight: 600;
  font-size: 13px;
  line-height: 1.3;
  word-break: break-all;
}

.file-attachment-size {
  font-size: 11px;
  opacity: 0.85;
  margin-top: 2px;
}

.input-sub-hint {
  display: flex;
  justify-content: space-between;
  margin-top: 6px;
  font-size: 11px;
  color: #94a3b8;
  padding: 0 4px;
}

/* 숨김 안내 토스트 배너 */
.hide-toast {
  position: fixed;
  top: 24px;
  right: 24px;
  background: #1e293b;
  color: #f8fafc;
  padding: 12px 18px;
  border-radius: 12px;
  font-size: 13.5px;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.25);
  display: flex;
  align-items: center;
  gap: 10px;
  z-index: 10000;
  border: 1px solid rgba(255, 255, 255, 0.1);
  animation: fadeInDown 0.3s ease;
}

@keyframes fadeInDown {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>
