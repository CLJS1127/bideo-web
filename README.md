# 🎬 BIDEO — 담당 영역 기술 문서

> 본 문서는 **Frontend + Auth + Real-time** 영역의 동작 방식을 코드와 함께 정리한 자료입니다.

---

## 🙋 담당 영역 한눈에

| 영역 | 핵심 책임 |
|---|---|
| [1. 인트로 페이지](#1-인트로-페이지-intro-main) | 비로그인 진입 / 서비스 첫인상 |
| [2. 메인 페이지](#2-메인-페이지-main) | 외부 추천 API 호출 + 갤러리 무한 스크롤 |
| [3. 로그인 / 회원가입](#3-로그인--회원가입) | JWT 인증, SMS 본인 인증 |
| [4. 소셜 로그인](#4-소셜-로그인-oauth-20) | OAuth 2.0, 분산 환경 호환 state |
| [5. 공통 레이아웃](#5-공통-레이아웃) | Thymeleaf fragment, 헤더·사이드바·채팅 FAB, 다크모드, 반응형 |
| [6. 실시간 채팅](#6-실시간-채팅) | WebSocket + STOMP + RabbitMQ + 무한 스크롤 |
| [7. 워터마크 검증 페이지](#7-워터마크-검증-페이지) | 업로드 파일에서 작가 식별자 추출 → 작가 카드 노출 |

---

## 1. 인트로 페이지 (`intro-main`)

비로그인 사용자가 가장 먼저 마주하는 화면. 서비스를 모르는 방문자에게 **3초 안에 "왜 BIDEO 인지"** 를 전달하는 게 목표.

### 구성

- **Hero 섹션** — 슬로건과 CTA 버튼
- **기능 소개** — 3분할 카드 (거래 / 커뮤니티 / 추천)
- **FAQ 탭패널** — 가입·결제·정산·환불·저작권
- **푸터** — 약관·문의 링크

### 라우팅

`HomeController` 에서 인증 여부에 따라 라우팅 분기.

```java
@GetMapping("/")
public String home(@AuthenticationPrincipal CustomUserDetails user) {
    return user == null ? "main/intro-main" : "redirect:/main";
}
```

비로그인 → `intro-main.html`, 로그인 사용자는 곧바로 `/main` 으로 이동.

### FAQ 탭 인터랙션

JS 한 줄로 탭 전환을 처리하면서 ARIA 속성도 같이 토글하여 접근성 확보.

```javascript
tabs.forEach((tab, idx) => tab.addEventListener("click", () => {
    panels.forEach(p => p.hidden = true);
    tabs.forEach(t => t.setAttribute("aria-selected", "false"));
    panels[idx].hidden = false;
    tab.setAttribute("aria-selected", "true");
}));
```

---

## 2. 메인 페이지 (`main`)

로그인 사용자의 홈. 두 가지 데이터 흐름이 동시에 동작:
1. **서버 사이드 렌더링** — Thymeleaf 가 첫 화면을 즉시 표시
2. **클라이언트 사이드 갱신** — 추천 슬라이드 / 무한 스크롤 fetch

### 외부 추천 API 비동기 호출 (`main-curation.js`)

페이지 진입 시 추천 결과를 비동기로 받아 슬라이드에 채워 넣는다.

```javascript
fetch('/api/prediction/curation?k=10', {
    method: 'GET',
    headers: { 'Accept': 'application/json' },
    credentials: 'same-origin'
})
.then(res => res.ok ? res.json() : Promise.reject('HTTP ' + res.status))
.then(render)
.catch(handleError);

function render(data) {
    track.innerHTML = data.items.map(buildCard).join('');
    subEl.textContent = '진행 중 ' + data.total_active + '건 중 Top ' + data.items.length;
}
```

외부 서버가 죽어도 페이지 자체는 동작해야 하므로 실패 시 섹션만 `hidden` 처리하고 다른 영역은 그대로 노출.

> 추천 모델의 설계/학습/서빙은 별도 [bideo-ai/README.md](../../../ai/workspace/llm/bideo-ai/README.md) 참조.

### 무한 스크롤 (`main.js`)

`IntersectionObserver` 로 sentinel 요소가 보일 때 다음 페이지를 prepend 가 아닌 **append** 방식으로 추가.

```javascript
let observer = new IntersectionObserver((entries) => {
    if (entries[0].isIntersecting && !loading && !ended) {
        loading = true;
        fetchNextPage().finally(() => { loading = false; });
    }
}, { rootMargin: "200px" });
observer.observe(sentinel);
```

`rootMargin: 200px` 로 화면 아래 200px 안에 들어오면 미리 호출 → 사용자 체감 끊김 X.

---

## 3. 로그인 / 회원가입

`SessionCreationPolicy.STATELESS` 로 세션 없이 **JWT (Access + Refresh)** 기반 인증.

### JWT 발급 흐름

```java
public void onAuthenticationSuccess(HttpServletRequest req, HttpServletResponse res,
                                    Authentication auth) {
    MemberVO member = (MemberVO) auth.getPrincipal();
    jwtTokenProvider.createAccessToken(member.getEmail(), "LOCAL", res);
    jwtTokenProvider.createRefreshToken(member.getEmail(), "LOCAL", res);
    res.sendRedirect("/main");
}
```

- **Access Token**: 짧은 수명(30분), 매 요청마다 검증
- **Refresh Token**: 긴 수명(2주), Redis 에 저장하여 탈취 시 무효화 가능

### 인증 필터

`AuthenticationFilter` 가 매 요청의 쿠키에서 토큰을 꺼내 검증 후 SecurityContext 에 주입.

```java
@Override
protected void doFilterInternal(HttpServletRequest req, ...) {
    String token = jwtTokenProvider.resolveToken(req);
    if (token != null && jwtTokenProvider.validate(token)) {
        Authentication auth = jwtTokenProvider.getAuthentication(token);
        SecurityContextHolder.getContext().setAuthentication(auth);
    }
    chain.doFilter(req, res);
}
```

### SMS 본인 인증

회원가입 시 Solapi 로 6자리 인증코드 발송 → Redis 에 5분 TTL 로 저장 → 검증.

---

## 4. 소셜 로그인 (OAuth 2.0)

네이버 · 카카오 · 구글 통합 지원. `oauth2Login` + 커스텀 `CustomOAuth2UserService` + `OAuth2SuccessHandler` 조합.

### Provider 통합 처리

각 provider 마다 응답 JSON 구조가 달라 `OAuth2Attribute` 로 정규화.

```java
public static OAuth2Attribute of(String provider, Map<String, Object> attributes) {
    return switch (OAuthProvider.from(provider)) {
        case KAKAO  -> ofKakao(attributes);
        case NAVER  -> ofNaver(attributes);
        case GOOGLE -> ofGoogle(attributes);
    };
}
```

### SuccessHandler 에서 회원 upsert

OAuth 응답을 받아 기존 회원이면 `providerId` 매칭, 없으면 신규 생성 후 JWT 발급.

```java
public void onAuthenticationSuccess(...) {
    OAuth2User oAuth2User = (OAuth2User) authentication.getPrincipal();
    MemberVO member = authService.upsertOAuthMember(
            oAuth2User.getAttribute("provider"),
            oAuth2User.getAttribute("id"),
            oAuth2User.getAttribute("email"),
            oAuth2User.getAttribute("name"),
            oAuth2User.getAttribute("profileImage")
    );
    jwtTokenProvider.createAccessToken(member.getEmail(), provider, response);
    response.sendRedirect("/");
}
```

### 🔧 트러블슈팅 — 분산 환경 OAuth state 실종

**증상**: EC2 두 대 + nginx `least_conn` 환경에서 네이버 로그인 시 콜백 단계에서 에러 페이지로 이동.

**원인**: Spring Security 기본 `HttpSessionOAuth2AuthorizationRequestRepository` 가 OAuth state 를 **JVM 메모리** 에 저장. 콜백이 다른 EC2 로 가면 state 매칭 실패.

**해결**: `AuthorizationRequestRepository` 를 직접 구현해 state 를 **HttpOnly + Secure 쿠키** 에 저장.

```java
@Override
public void saveAuthorizationRequest(OAuth2AuthorizationRequest req,
                                     HttpServletRequest request,
                                     HttpServletResponse response) {
    if (req == null) { deleteCookie(response); return; }
    Cookie cookie = new Cookie(COOKIE_NAME, serialize(req));
    cookie.setHttpOnly(true);
    cookie.setSecure(true);
    cookie.setPath("/");
    cookie.setMaxAge(COOKIE_EXPIRE_SECONDS);
    cookie.setAttribute("SameSite", "Lax");
    response.addCookie(cookie);
}
```

SecurityConfig 에 등록:

```java
.oauth2Login(oauth -> oauth
    .authorizationEndpoint(authz -> authz
        .authorizationRequestRepository(cookieOAuth2AuthorizationRequestRepository))
    .userInfoEndpoint(userInfo -> userInfo.userService(customOAuth2UserService))
    .successHandler(oAuth2SuccessHandler)
)
```

**결과**: `STATELESS` 유지하면서 어떤 EC2 가 콜백 받아도 쿠키에서 state 복원. ip_hash 같은 우회 없이 정석 해결.

---

## 5. 공통 레이아웃

모든 페이지가 공유하는 YouTube 스타일 shell (헤더·사이드바·채팅 FAB·다크모드). Thymeleaf
fragment 로 한 군데에서 정의하고 각 페이지가 갈아끼우는 방식.

### Thymeleaf fragment 3종 패턴

`templates/layout/layout.html` 한 파일에 3개 fragment 를 정의해 각 페이지가 자기 자리에서 호출.

```html
<!-- layout/layout.html -->
<th:block th:fragment="ytShellHead(pageTitle)">
    <!-- meta / CSS / SockJS / STOMP / 전역 JS -->
</th:block>

<th:block th:fragment="ytShellChrome(activeItemKey)">
    <!-- header + sidebar + 채팅 FAB + 검색 portal -->
</th:block>

<th:block th:fragment="ytShellScripts()">
    <!-- 페이지 진입 후 실행할 부트스트랩 스크립트 -->
</th:block>
```

```html
<!-- main/main.html — 각 페이지가 fragment 끼워 넣음 -->
<head>
    <th:block th:replace="~{layout/layout :: ytShellHead('BIDEO')}"></th:block>
</head>
<body>
    <div th:replace="~{layout/layout :: ytShellChrome('home')}"></div>
    <main class="bd-shell__page-content">
        <!-- 페이지 고유 컨텐츠 -->
    </main>
</body>
```

`activeItemKey` 로 사이드바 메뉴 중 어디가 켜져야 할지 알려줌 (`'home'` / `'workdetail'` /
`'contest'` / `'profile'` 등). fragment 안에서 `th:classappend` 로 `.is-active` 부여.

### 헤더 (`bd-shell__header`)

3 단 구성 — 로고 / 검색바 / 액션 버튼.

```
[BIDEO 로고] ─── [검색 인풋 + 음성검색 마이크] ─── [만들기] [알림] [프로필]
```

- **검색바**: 데스크탑은 항상 노출, 모바일은 돋보기 아이콘 누르면 `.bd-shell__mobile-search` 오버레이로 펼침.
- **음성검색**: Web Speech API 로 즉시 텍스트 변환 후 같은 검색 form submit.
- **알림 / 프로필**: 드롭다운 — `data-bd-shell-notification-toggle` / `data-bd-shell-profile-toggle` 로 토글.

### 사이드바 (`bd-shell__guide`)

4 개 섹션을 동적으로 렌더링.

| 섹션 | 출처 | 비고 |
|---|---|---|
| 가이드 | 정적 (홈 / 작품 / 공모전) | 항상 표시 |
| 팔로우 한 아티스트 | `followingArtists` 모델 | 최대 5명 + "더 보기" |
| 보관함 | 정적 (내 페이지 / 찜한 작품) | 로그인 시 |
| 인기 태그 | `popularTags` 모델 | 비어있으면 섹션 숨김 |

각 페이지 컨트롤러가 `Model` 에 `followingArtists`, `popularTags` 를 넣어주면 fragment 안에서
`th:if` 로 자연스럽게 분기됨. 모델 안 넣어도 깨지지 않음.

### 채팅 FAB (`bd-chat-fab`)

전역 우하단 떠 있는 채팅 진입 버튼. 어떤 페이지에서든 클릭 한 번으로 `bd-chat-panel` 드로어가
오버레이로 열림. 안 읽은 메시지가 있으면 빨간 뱃지(`bd-chat-fab__badge`) 노출.

```html
<button class="bd-chat-fab" aria-controls="bd-chat-panel" data-bd-chat-toggle>
    <span class="bd-chat-fab__icon">...</span>
    <span class="bd-chat-fab__badge">3</span>   <!-- 안 읽은 개수 -->
</button>
<div class="bd-chat-layer" hidden data-bd-chat-layer>
    <button class="bd-chat-layer__backdrop" data-bd-chat-dismiss></button>
    <section id="bd-chat-panel" class="bd-chat-panel" data-bd-chat-panel>
        <div class="bd-chat-panel__body">
            <aside class="bd-chat-sidebar">  <!-- 채팅방 리스트 --></aside>
            <section class="bd-chat-thread"> <!-- 선택된 방의 메시지 --></section>
        </div>
    </section>
</div>
```

→ 채팅 진입을 위해 별도 페이지 이동이 없음. 작품 보다가도 / 결제하다가도 즉시 대화 가능.
**실시간 데이터 흐름은 6장 참조.**

### 다크/라이트 테마

CSS 변수 기반. `<html data-theme="dark">` 속성 토글로 한 번에 전환.

```css
:root {
    --bd-color-bg: #ffffff;
    --bd-color-text: #1a1a1a;
    --bd-color-accent: #6800EA;
}

:root[data-theme="dark"] {
    --bd-color-bg: #0f0f0f;
    --bd-color-text: #f5f5f5;
    --bd-color-accent: #9D4EDD;
}
```

JS 로 토글 + localStorage 에 저장.

```javascript
function toggleTheme() {
    const next = html.dataset.theme === "dark" ? "light" : "dark";
    html.dataset.theme = next;
    localStorage.setItem("bd-theme", next);
}
```

`/css/common/theme.css` 와 `/js/layout/theme.js` 는 `<head>` 에서 **defer 없이** 먼저 로드 —
페이지 그려지기 전에 테마 결정해 깜빡임(FOUC) 방지.

### 반응형

미디어 쿼리로 두 가지 전환점.
- `≤ 1024px`: 사이드바 접힘 — 아이콘만 표시
- `≤ 768px`: 사이드바 완전 hidden — 헤더 검색바도 아이콘 토글식 오버레이로 전환

`.bd-shell` 단일 루트에 `data-bd-shell-root` 마커. JS 가 이 마커로 전체 shell 의 상태(검색 오픈 /
사이드바 접힘 / 채팅 패널 열림) 를 관리.

### 접근성

- `.bd-shell__skip` — "탐색 건너뛰기" 링크 (스크린리더 사용자 첫 Tab 에서 본문 직행)
- 모든 아이콘 버튼에 `aria-label`
- 토글 버튼에 `aria-expanded`, 패널에 `aria-controls`
- SVG 아이콘은 `aria-hidden="true"` — 라벨은 옆 텍스트로

### 🔧 트러블슈팅 — 다크 모드 채팅 버블이 네모 박스로 표시됨

**증상**: 다크모드에서 채팅 말풍선이 둥근 형태가 아닌 사각형 보라색 박스로 보임.

**원인**: `.bd-chat-bubble--self` (그리드 컨테이너)에 잘못 `background-color: accent` 가 적용됨. 둥근 모서리(`border-radius`)는 자식 `.bd-chat-bubble__body` 에만 있는데 컨테이너에 색이 칠해져 바깥 영역까지 보라색이 번짐.

**해결**: 컨테이너 배경을 `transparent` 로 변경, 색상은 `__body` 에만 유지.

```css
:root[data-theme="dark"] .bd-chat-bubble--self {
    background: transparent !important;   /* 컨테이너는 색 X */
}

:root[data-theme="dark"] .bd-chat-bubble--self .bd-chat-bubble__body {
    background: var(--bd-color-accent) !important;
    border-radius: 18px;                  /* 진짜 버블에만 둥근 모서리 */
}
```

---

## 6. 실시간 채팅

가장 복잡한 모듈. WebSocket + STOMP + RabbitMQ + 무한 스크롤 페이징을 종합.

### 기본 구조

```
[브라우저] ←SockJS+STOMP→ [Spring WebSocket /ws]
                                    │
                                    ▼
                            [SimpleBroker (in-memory)]
                                    │
                                    └→ broadcast → 구독자 WebSocket
```

`WebSocketConfig`:

```java
@Override
public void registerStompEndpoints(StompEndpointRegistry registry) {
    registry.addEndpoint("/ws")
            .setAllowedOriginPatterns("*")
            .withSockJS();
}
```

클라이언트:

```javascript
let socket = new SockJS('/ws');
stompClient = Stomp.over(socket);
stompClient.connect({}, () => {
    stompClient.subscribe('/topic/room.' + roomId, (frame) => {
        handleRealtimeEvent(JSON.parse(frame.body));
    });
});
```

### 🔧 트러블슈팅 1 — 분산 환경에서 메시지가 다른 EC2 사용자에게 안 감

**증상**: User A (EC2 #1) 가 보낸 메시지가 같은 방의 User B (EC2 #2) 에게 도달하지 않음.

**원인**: `SimpleBroker` 는 JVM 인메모리. 인스턴스 간 메시지 공유 불가.

**해결**: **RabbitMQ Fanout Exchange** 도입. 메시지를 `RabbitTemplate.convertAndSend()` 로 발행하면 모든 인스턴스의 `@RabbitListener` 가 수신 후 각자 자기 WebSocket 구독자에게 broadcast.

```java
@Configuration
public class RabbitConfig {
    public static final String CHAT_EXCHANGE = "bideo.chat.exchange";

    @Bean
    public FanoutExchange chatExchange() { return new FanoutExchange(CHAT_EXCHANGE); }

    @Bean
    public Queue chatQueue() { return new AnonymousQueue(); }   // 인스턴스별 임시 큐

    @Bean
    public Binding chatBinding(Queue q, FanoutExchange ex) {
        return BindingBuilder.bind(q).to(ex);
    }
}
```

발행:

```java
private void publishRelay(String destination, Object payload) {
    rabbitTemplate.convertAndSend(
            RabbitConfig.CHAT_EXCHANGE, "",
            ChatRelayMessage.builder()
                    .destination(destination)
                    .payload(payload)
                    .build());
}
```

수신:

```java
@RabbitListener(queues = "#{chatQueue.name}")
public void onRelay(ChatRelayMessage relay) {
    messagingTemplate.convertAndSend(relay.getDestination(), relay.getPayload());
}
```

### 🔧 트러블슈팅 2 — 메시지 50개 초과 시 화면 미표시

**증상**: 채팅 메시지가 50개 넘어가면 새로 들어온 메시지가 안 보임.

**원인**: 매퍼가 `ORDER BY created_datetime ASC LIMIT 50` 으로 가장 오래된 50개만 반환. 클라이언트도 항상 page 0 호출.

**해결 1**: 매퍼를 `DESC` 로 변경 → 최신 50개를 가져오고 서비스에서 reverse.

```java
List<MessageResponseDTO> messages = messageDAO.findByRoomId(roomId, memberId, page * 50, 50);
Collections.reverse(messages);   // 화면은 시간순
return messages;
```

**해결 2**: 클라이언트에 무한 스크롤 페이징 추가. 스크롤이 최상단 근처(80px 이내) 도달 시 page++ 호출 후 prepend, 스크롤 위치 보존.

```javascript
function loadMoreMessages() {
    if (isLoadingMore || !activeRoomHasMore) return;
    isLoadingMore = true;
    let prevScrollHeight = messagesNode.scrollHeight;
    let prevScrollTop    = messagesNode.scrollTop;

    fetch(`/api/messages/rooms/${activeRoomId}/messages?page=${++activeRoomPage}`)
        .then(r => r.json())
        .then(messages => {
            if (!messages.length) { activeRoomHasMore = false; return; }
            room.messages = messages.concat(room.messages);   // prepend
            renderMessages(room);
            // 스크롤 위치 보정 (튀지 않게)
            messagesNode.scrollTop = prevScrollTop +
                (messagesNode.scrollHeight - prevScrollHeight);
        })
        .finally(() => { isLoadingMore = false; });
}

messagesNode.addEventListener('scroll', () => {
    if (messagesNode.scrollTop < 80) loadMoreMessages();
});
```

### 🔧 트러블슈팅 3 — 실시간 broadcast 가 페이징 상태를 망가뜨림

**증상**: 옛 메시지 페이징해서 보고 있는 중 새 메시지가 오면 전체 reload 되어 page 0 으로 돌아감.

**원인**: `handleRealtimeEvent` 가 새 이벤트 받을 때마다 `loadMessages` 로 전체 재호출.

**해결**: broadcast 페이로드를 직접 활용. `CREATED` 는 append, `UPDATED`/`DELETED`/`LIKED` 는 in-place replace.

```javascript
function handleRealtimeEvent(event) {
    let room = findRoom(activeRoomId);
    if (event.type === "CREATED") {
        room.messages.push(event.message);
        renderMessages(room);
        messagesNode.scrollTop = messagesNode.scrollHeight;
    } else {
        let idx = room.messages.findIndex(m => m.id === event.message.id);
        if (idx >= 0) {
            room.messages[idx] = event.message;
            renderMessages(room);
        }
    }
}
```

---

## 7. 워터마크 검증 페이지

작품 업로드 시 이미지/영상에 작가 식별자가 비가시성 워터마크(DWT-DCT)로 박힌다. 저작권 분쟁
시 추출해서 원본 업로더를 확인하는 페이지. 일반 작가도 본인 작품이 도용된 의심이 들면 그 파일을
올려 진짜 작가가 누군지 확인 가능.

> 워터마크를 박는 ML 동작과 알고리즘 자체는 별도 [bideo-ai/README.md](../../../ai/workspace/llm/bideo-ai/README.md) 참조.
> 이 섹션은 **검증 UI / 호출 흐름** 만 다룬다.

### 진입점 — 프로필 드롭다운 (사이드바 X)

우상단 프로필 아이콘 클릭 → 드롭다운의 **"아티스트 도구"** 섹션 아래에 배치:

```
[작품 관리]      내 작품 · 찜한 작품
[아티스트 도구]  아티스트 대시보드 · 공모전 참여 현황 · 워터마크 검증   ← 여기
[설정]           디자인 테마 · 로그아웃
```

사이드바는 "탐색" 컨텍스트(홈/작품/공모전) 라 검증 같은 도구는 의도적으로 프로필 메뉴로 분리.

### 페이지 흐름

```
사용자가 우상단 프로필 → "워터마크 검증" 클릭
        ↓
GET /watermark/verify                  → verify.html 렌더 (드롭존 + 미리보기)
        ↓
이미지/영상 드래그&드롭 또는 파일 선택 → JS 가 즉시 <img>/<video> 미리보기
        ↓
"워터마크 검증" 버튼
        ↓
POST /api/watermark/verify  (multipart: file)
        ↓
WatermarkController.verify()
   ├─ WatermarkApiClient.extract(bytes, contentType, filename)
   │     → FastAPI /api/watermark/extract
   │     → payload (10진수 member_id 문자열)
   ├─ Long.parseLong(payload) → member_id
   ├─ MemberRepository.findById(memberId)
   └─ S3 프로필 이미지 presigned URL 생성
        ↓
응답: { valid, payload, source, creator: {id, nickname, profileImage, ...}, message }
        ↓
verify.js → 검증 상태 배지(녹/적/회) + 작가 카드 + 메시지 노출
```

### 컨트롤러 — 비동기 호출의 동기 적응

ML 호출은 `Mono` 인데 컨트롤러는 동기 응답을 돌려줘야 함. `blockOptional(Duration.ofMinutes(2))` 으로
**2분 타임아웃 안에 결과 받거나 빈 결과** 패턴. 영상 워터마크 추출이 길어질 수 있어 여유를 둠.

```java
ExtractedResult extracted = watermarkApiClient
        .extract(fileBytes, contentType, file.getOriginalFilename())
        .blockOptional(Duration.ofMinutes(2))
        .orElse(null);

if (extracted != null && extracted.valid() && extracted.payload() != null) {
    Long memberId = Long.parseLong(extracted.payload().trim());
    MemberVO member = memberRepository.findById(memberId).orElse(null);
    // creator 카드 생성 ...
}
```

ML 서버가 꺼져있어도 `Mono.empty()` 로 흡수되어 페이지가 죽지 않고 "워터마크를 찾지 못했습니다"
메시지가 표시됨.

### 진단용 — workId 로 S3 파일 직접 검증

내가 올린 작품에 워터마크가 제대로 박혔는지 다운로드/재업로드 라운드트립 없이 바로 확인하는
엔드포인트:

```
POST /api/watermark/verify/work/{workId}
```

`WorkDAO.findFilesByWorkId(workId)` → sort_order 우선순위로 미디어 파일 1개 고름 →
`S3FileService.downloadBytes(fileUrl)` → `extract()` → 응답. 시연에서 "방금 올린 작품 워터마크
확인해 보자" 같은 시나리오용.

### JS — 드래그앤드롭 + 즉시 미리보기

`URL.createObjectURL(file)` 로 업로드 전에 미리보기. 영상은 `<video controls muted>` 로 인라인
재생까지 가능. submit 시 같은 `FormData` 를 fetch 로 전송.

```javascript
drop.addEventListener('drop', e => {
    const file = e.dataTransfer?.files?.[0];
    if (!file) return;
    const dt = new DataTransfer();
    dt.items.add(file);
    fileInput.files = dt.files;   // <input type="file"> 에 강제 주입
    showPreview(file);
});
```

`<input type="file">` 에 프로그래매틱으로 파일을 넣을 때 `DataTransfer` 우회 — 브라우저 보안상
직접 `fileInput.files = [file]` 은 안 됨.

### 🔧 트러블슈팅 — 두 워크스페이스 사이에서 검증 페이지가 사라져 있었음

**증상**: 분명히 만들었던 `/watermark/verify` 페이지가 현재 워크스페이스에 없음.

**원인**: 프로젝트가 두 워크스페이스로 갈라져있었고 (`aws/workspace/bideo`, `spring/workspace/bideo`),
검증 페이지는 후자에만 있었음. 메인 작업이 전자로 옮겨오면서 미동기화.

**해결**: 컨트롤러 / HTML / CSS / JS 를 복사하면서 호환성 조정:
- 구버전이 의존하던 `tbl_work_file.file_hash` 컬럼 — 우리 워크스페이스엔 없어서 **해시 fallback 분기 제거**.
  FastAPI extract 결과만 사용.
- 구버전 verify.html 의 `yt-shell__page-content` → 우리 워크스페이스 컨벤션 `bd-shell__page-content` 로 클래스명 통일.
- 메뉴 진입점은 사이드바가 아닌 프로필 드롭다운에 배치 (재논의 결과).

---

## 🎓 회고

### 잘한 점
- **분산 환경의 무상태성**을 정석으로 풀어냄 — OAuth state는 쿠키로, 메시지는 RabbitMQ Fanout 으로
- **사용자 체감 품질** 신경 — 무한 스크롤의 스크롤 위치 보존, 다크모드 CSS 변수, 비동기 페이지 끊김 없음
- **인프라까지 보고 해결** — Spring 코드만이 아니라 nginx 설정, AWS 보안그룹, Cloudflare 영역도 함께 운영

### 아쉬운 점
- **테스트 코드 부재** — JWT 필터·OAuth flow 같은 인증 로직은 통합 테스트가 꼭 필요
- **WebSocket 부하 테스트 미실시** — 동시 채팅 1000명 같은 상황을 가정한 부하 테스트가 없음

### 배운 점
- **메모리에 있는 것은 모두 분산 환경의 적** — 세션·state·메시지 어느 것도 JVM 메모리에 두면 안 됨
- **인프라 한 줄이 코드 100줄 보다 영향력 클 수 있다** — nginx `proxy_buffering off` 한 줄로 채팅 실시간 살아남, mkcert → Let's Encrypt 전환으로 OAuth 콜백 살아남
- **사용자 흐름을 끝까지 따라가야 한다** — "메시지를 보낸다" 라는 한 액션 뒤에 직렬화·broker·구독자 분기·페이징 갱신 같은 7~8개 단계가 숨어있음

---

🔗 추천/예측 모델 문서: [bideo-ai/README.md](../../../ai/workspace/llm/bideo-ai/README.md)
