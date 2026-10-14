# Vue 2.5.17 플로팅 드래그 챗봇 대화창

Vue 2.5.17과 100% 호환되는 드래그 가능한 플로팅 챗봇 UI 컴포넌트 및 테스트 데모 환경입니다.

## 🌟 주요 요구사항 구현 내역

1. **플로팅 버튼**: 바탕화면 우측 하단에 고정된 원형 플로팅 런처 버튼으로 클릭 시 부드러운 인터랙션과 함께 채팅창을 엽니다.
2. **드래그 이동**: 채팅창 상단 헤더(`chat-header`) 영역을 마우스나 모바일 터치로 잡고 화면 어디든 자유롭게 이동할 수 있으며, 화면 밖으로 이탈하지 않도록 경계 제한(Boundary clamping)이 적용되어 있습니다.
3. **챗봇 컨셉 대화**: 아바타, 봇/사용자 말풍선 분리, 전송 시간, 추천 질문 칩, 대화 템플릿 카드 및 자동 응답 타이핑 시뮬레이션이 포함되어 있습니다.
4. **좌측 채팅 이력 리스트 & 관리(수정/삭제/접기/펼치기)**: 과거 상담 및 대화 내역(세션 목록)을 표시하고 클릭하여 대화를 전환할 수 있으며, '+ 새 대화' 생성 기능도 제공됩니다. 각 이력 아이템에 호버/선택 시 나타나는 제목 수정(인라인 수정 폼) 및 삭제(확인 팝업 및 세션 자동 전환) 버튼이 탑재되어 있으며, 사이드바 접기/펼치기 토글도 지원합니다.
5. **상태 보존 숨기기(Minimize)**: 상단 헤더의 숨기기(`-`) 버튼을 누르면 `v-show`를 통해 창만 숨겨지며, 작성 중이던 입력창 메시지, 선택된 세션, 스크롤 위치, 대화 내역이 그대로 유지됩니다.
6. **화면 최대화 - 기본사이즈 토글**: 상단 우측 최대화 토글 버튼(또는 상단 헤더 더블클릭)을 통해 전체 화면 최대화 모드와 이전 크기/위치로의 복원을 즉시 전환할 수 있습니다.
7. **마우스 창 크기 리사이즈(Resize)**: 창 우측 가장자리, 하단 가장자리, 우하단 모서리 핸들을 마우스나 터치로 드래그하여 대화창 크기를 자유롭게 늘리거나 줄일 수 있습니다.
8. **개발용 대화 템플릿**: 질문에 따른 스마트 목업 응답, 카드 액션 버튼, 추천 질문 칩 등이 구비되어 있어 실제 백엔드 API 연동 전 UI/UX 테스트를 즉시 진행할 수 있습니다.
9. **초기 진입 화면 (세로 리스트 추천 질문 & 유의사항 안내)**: 새 대화 시작 또는 초기 진입 시 중앙에 서비스 안내, 아래로 내려오는 세로형 추천 질문 리스트 카드(클릭 시 즉시 질문 전송), 하단 유의사항/면책 안내 박스가 표시되어 처음 사용하는 사용자도 쉽게 질문을 시작할 수 있습니다.
10. **첨부파일 목록 대기 및 동시 전송**: 메시지 전송 버튼 좌측에 파일 첨부 클립 버튼(`📎`)을 제공하며, 파일 선택 시 바로 전송되지 않고 입력창 상단 파일 목록 바(칩 형태)에 대기 상태로 보관됩니다. 목록에서 각 파일의 개별 삭제(`×`)가 가능하며, 메시지 작성 후 전송 시 텍스트와 파일이 함께 묶여 전송됩니다 (다중 파일 첨부 지원).
11. **개별 대화 메시지 삭제 기능**: 대화창 내 각 말풍선에 호버(터치 기기 지원) 시 나타나는 삭제 버튼(휴지통 아이콘)을 클릭하여 원하는 메시지만 개별 삭제할 수 있으며, 삭제 확인 팝업을 거쳐 안전하게 제거되고 세션 미리보기 및 메시지 수가 자동 갱신됩니다.
12. **대화 이력 전체 삭제 기능**: 좌측 사이드바 상단 헤더의 전체 삭제 아이콘 버튼 및 하단 전용 푸터 버튼(`대화 이력 전체 삭제`)을 제공하며, 삭제 확인 팝업을 거쳐 모든 이전 대화를 일괄 정리하고 깨끗한 새 대화 세션으로 안전하게 초기화합니다.
13. **클립보드 이미지 붙여넣기(Paste) 첨부 지원**: 스크린샷 캡처나 클립보드에 복사된 이미지를 메시지 입력창에 붙여넣기(`Cmd+V` / `Ctrl+V`)하면 자동으로 감지하여 첨부파일 목록에 썸네일 미리보기와 함께 대기 상태로 등록되며, 메시지와 함께 간편하게 전송할 수 있습니다.

---

## 📂 파일 구성

- **`index.html`**: Vue 2.5.17 CDN 기반의 즉시 실행 가능한 완성형 단일 데모 파일 (브라우저에서 바로 열람 가능)
- **`src/FloatingChatbot.vue`**: Vue CLI / Webpack 등 기존 Vue 2.5.17 프로젝트에 바로 복사하여 사용할 수 있는 Single File Component (SFC)
- **`package.json`**: 로컬 테스트 개발 서버 구동 스크립트

---

## 🚀 실행 방법

### 1) 로컬 개발 서버 실행
```bash
npm run dev
# 또는
npx serve -l 3000 .
```
브라우저에서 `http://localhost:3000` 접속

### 2) 브라우저에서 직접 열기
`index.html` 파일을 더블클릭하여 Chrome, Safari, Edge 등 임의의 브라우저에서 바로 열어도 정상 동작합니다.

---

## 💡 기존 Vue 2.5.17 프로젝트에 적용하기

`src/FloatingChatbot.vue`를 프로젝트 컴포넌트 디렉토리에 복사한 후 아래와 같이 사용합니다:

```vue
<template>
  <div id="app">
    <!-- 메인 서비스 화면 -->
    <router-view />

    <!-- 플로팅 챗봇 컴포넌트 (모든 CSS와 로직이 내장되어 단독 사용 가능) -->
    <floating-chatbot 
      :initial-open="false"
      bot-name="루나 AI 챗봇"
      bot-avatar="🤖"
      :enable-mock-reply="true"
      @send-message="onSendMessage"
      @file-upload="onFileUpload"
      @session-change="onSessionChange"
      @message-deleted="onMessageDeleted"
      @hide="onChatHide"
    />
  </div>
</template>

<script>
import FloatingChatbot from '@/components/FloatingChatbot.vue';

export default {
  name: 'App',
  components: {
    FloatingChatbot
  },
  methods: {
    onSendMessage(payload) {
      console.log('메시지 전송 이벤트:', payload.text, payload.attachments, payload.session);
    },
    onFileUpload(payload) {
      console.log('파일 업로드 이벤트:', payload.file.name, payload.session);
    },
    onSessionChange(session) {
      console.log('세션 변경 이벤트:', session.title);
    },
    onMessageDeleted(payload) {
      console.log('메시지 삭제 이벤트:', payload.message, payload.index);
    },
    onChatHide() {
      console.log('챗봇 창이 숨겨졌습니다 (상태 보존 및 토스트 안내).');
    }
  }
};
</script>
```

### ⚙️ Component Props & Events

| Prop 명 | 타입 | 기본값 | 설명 |
| :--- | :--- | :--- | :--- |
| `initial-open` | Boolean | `false` | 컴포넌트 마운트 시 최초 열림 여부 |
| `bot-name` | String | `'루나 AI 챗봇'` | 챗봇 이름 헤더 및 말풍선 발신자명 |
| `bot-avatar` | String | `'🤖'` | 챗봇 아바타 아이콘 또는 이모지 |
| `bot-status-text` | String | `'무엇이든 물어보세요 • 실시간 자동응답'` | 헤더 서브타이틀 상태 문구 |
| `enable-mock-reply` | Boolean | `true` | 스마트 목업 자동 응답 생성기 활성화 여부 |
| `primary-color` | String | `''` | 브랜드 메인 컬러 오버라이드 (예: `#2563eb`) |

| Event 명 | 전달 인자 (Payload) | 설명 |
| :--- | :--- | :--- |
| `@send-message` | `{ text, session, attachments }` | 사용자가 메시지 및 첨부파일 전송 시 발생 (실제 API 연동에 활용) |
| `@file-upload` | `{ file, session }` | 사용자가 파일 첨부 업로드 시 발생 (개별 파일 이벤트) |
| `@session-change` | `session` | 세션 전환 또는 새 대화 생성 시 발생 |
| `@sessions-cleared` | `{ newSession }` | 모든 대화 이력 전체 삭제 시 발생 |
| `@message-deleted` | `{ message, index, session }` | 특정 말풍선 메시지 삭제 시 발생 |
| `@hide` | - | 상단 숨기기(`-`) 버튼 클릭으로 창 숨김 시 발생 |
