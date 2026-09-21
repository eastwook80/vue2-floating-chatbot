# Vue 2.5.17 플로팅 드래그 챗봇 대화창

Vue 2.5.17과 100% 호환되는 드래그 가능한 플로팅 챗봇 UI 컴포넌트 및 테스트 데모 환경입니다.

## 🌟 주요 요구사항 구현 내역

1. **플로팅 버튼**: 바탕화면 우측 하단에 고정된 원형 플로팅 런처 버튼으로 클릭 시 부드러운 인터랙션과 함께 채팅창을 엽니다.
2. **드래그 이동**: 채팅창 상단 헤더(`chat-header`) 영역을 마우스나 모바일 터치로 잡고 화면 어디든 자유롭게 이동할 수 있으며, 화면 밖으로 이탈하지 않도록 경계 제한(Boundary clamping)이 적용되어 있습니다.
3. **챗봇 컨셉 대화**: 아바타, 봇/사용자 말풍선 분리, 전송 시간, 추천 질문 칩, 대화 템플릿 카드 및 자동 응답 타이핑 시뮬레이션이 포함되어 있습니다.
4. **좌측 채팅 이력 리스트 & 접기/펼치기**: 과거 상담 및 대화 내역(세션 목록)을 표시하고 클릭하여 대화를 전환할 수 있으며, '+ 새 대화' 생성 기능도 제공됩니다. 상단 헤더, 세션 정보 바, 사이드바 헤더에 배치된 토글 버튼을 통해 이력 화면을 매끄러운 애니메이션으로 접었다 폈다 할 수 있습니다.
5. **상태 보존 숨기기(Minimize)**: 상단 헤더의 숨기기(`-`) 버튼을 누르면 `v-show`를 통해 창만 숨겨지며, 작성 중이던 입력창 메시지, 선택된 세션, 스크롤 위치, 대화 내역이 그대로 유지됩니다.
6. **개발용 대화 템플릿**: 질문에 따른 스마트 목업 응답, 카드 액션 버튼, 추천 질문 칩 등이 구비되어 있어 실제 백엔드 API 연동 전 UI/UX 테스트를 즉시 진행할 수 있습니다.

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

    <!-- 플로팅 챗봇 컴포넌트 -->
    <floating-chatbot 
      :initial-open="false"
      bot-name="루나 AI 챗봇"
      bot-avatar="🤖"
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
    onChatHide() {
      console.log('챗봇 창이 숨겨졌습니다 (상태 보존).');
    }
  }
};
</script>
```
