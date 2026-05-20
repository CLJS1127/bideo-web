# 🎬 BIDEO — 영상 크리에이터를 위한 프리미엄 마켓플레이스 (Web)

> 영상은 누구나 만들 수 있는 시대,
> 정작 그 영상이 **작품으로 거래되는 시장**은 없었습니다.
> BIDEO는 그 빈 자리를 채우는, **영상 작품 거래·경매 플랫폼**입니다.

---

## 📌 프로젝트 기획 배경

| 문제 | BIDEO의 해결 |
|---|---|
| 영상 크리에이터에게 **정당한 수익 창구**가 없음 | 즉시 판매 + 경매, 두 가지 거래 방식 제공 |
| 좋은 작품을 **알아볼 안목**이 부족 | AI 기반 큐레이션·예측 모델 (별도 서버) 연동 |
| 단순 콘텐츠 소비를 넘어선 **소장 욕구** 부재 | 작가별 예술관(갤러리)·팔로우·뱃지 시스템 |
| 작품 도용·무단 사용 우려 | AI 워터마크 자동 삽입으로 저작권 보호 |

---

## 🚀 핵심 기능

| 영역 | 기능 |
|---|---|
| **거래** | 즉시 구매 / 경매 입찰 / 결제(BootPay) / 정산·환불 자동화 |
| **커뮤니티** | 작가별 예술관(갤러리), 팔로우, 좋아요, 북마크, 댓글, 신고 |
| **실시간** | 1:1 채팅, 알림, 메시지 좋아요 (WebSocket + STOMP + RabbitMQ) |
| **인증** | JWT, OAuth 2.0 (네이버 · 카카오 · 구글) |
| **AI 연동** | ML 서버 API 호출 — 큐레이션 · 성장 예측 · RAG 추천 · 워터마크 |

---

## 🛠 기술 스택

| 분류 | 사용 기술 |
|---|---|
| **Backend** | Spring Boot 3.5, Java 17, Spring Security, Spring WebSocket, MyBatis |
| **Frontend** | Thymeleaf, HTML / JavaScript / CSS, SockJS, STOMP |
| **Database** | PostgreSQL, Redis (세션·캐시) |
| **Message Broker** | **RabbitMQ** (Fanout Exchange) — 분산 환경 채팅 메시지 일관성 |
| **Infra** | AWS EC2 × 2 (Spring), AWS S3, nginx (load balancer + TLS 종료) |
| **Auth** | Spring Security, JWT, OAuth 2.0, BCrypt |
| **External API** | BootPay (결제), Solapi (SMS), Gmail SMTP |
| **CI/CD** | GitHub Actions, Docker, Let's Encrypt (certbot) |
| **협업** | Git / GitHub, Notion, Figma, Discord |

### 인프라 아키텍처

```
                       ┌─────────────────┐
                       │   Route 53      │  bideo.ai.kr
                       └────────┬────────┘
                                ▼
                       ┌─────────────────┐
                       │   nginx (LB)    │  Let's Encrypt SSL
                       │   least_conn    │
                       └────────┬────────┘
                    ┌───────────┴───────────┐
                    ▼                       ▼
             ┌─────────────┐         ┌─────────────┐
             │ Spring EC2 1│         │ Spring EC2 2│
             │  (Docker)   │         │  (Docker)   │
             └──────┬──────┘         └──────┬──────┘
                    │                        │
        ┌───────────┼────────────────────────┼───────────┐
        ▼           ▼                        ▼           ▼
  ┌──────────┐ ┌──────────┐         ┌──────────┐  ┌──────────┐
  │PostgreSQL│ │  Redis   │         │ RabbitMQ │  │   S3     │
  └──────────┘ └──────────┘         └──────────┘  └──────────┘
                    │
                    └──→ [bideo-ai 서버] (FastAPI / Cloudflare Tunnel)
```

---

## 🙋 담당 영역

### Frontend + Auth + Real-time

| 영역 | 핵심 구현 |
|---|---|
| 🏠 **인트로 페이지** (`intro-main`) | 비로그인 진입 화면, FAQ 탭패널, CTA 섹션 |
| 🎬 **메인 페이지** (`main`) | AI 큐레이션 슬라이드, 카테고리별 작품, 무한 스크롤 |
| 🔐 **로그인 / 회원가입** | JWT (Access + Refresh), 이메일 검증, SMS 인증 |
| 🔑 **소셜 로그인** | OAuth 2.0 (네이버·카카오·구글), 커스텀 `OAuth2SuccessHandler` |
| 🎨 **공통 레이아웃** | 다크/라이트 테마, 반응형 (모바일·태블릿·PC) |
| 💬 **실시간 채팅** | WebSocket + STOMP + SockJS, 무한 스크롤 페이징, 좋아요·삭제·답장 |

---

## 🔧 트러블슈팅

### 1. 분산 환경에서 OAuth 로그인 실패

**증상**: EC2 두 대 분산 환경에서 네이버 로그인 후 에러 페이지로 이동

**원인**: Spring Security 기본 `HttpSessionOAuth2AuthorizationRequestRepository` 가 OAuth state 를 **JVM 메모리** 에 저장. nginx 로드밸런서가 콜백을 다른 EC2 로 보내면서 state 불일치

**해결**: `CookieOAuth2AuthorizationRequestRepository` 직접 구현 → state 를 **HttpOnly + Secure 쿠키** 에 base64 직렬화하여 저장. 어떤 EC2 가 콜백 받아도 쿠키에서 복원 가능

**결과**: `SessionCreationPolicy.STATELESS` 유지하면서 분산 환경 완전 호환

---

### 2. 분산 환경에서 채팅 메시지 격리

**증상**: 한 EC2 에 연결된 사용자가 보낸 메시지가 다른 EC2 의 사용자에게 전달되지 않음

**원인**: Spring 기본 `SimpleBroker` 는 JVM 인메모리 방식. 여러 인스턴스 간 메시지 공유 불가

**해결**: **RabbitMQ Fanout Exchange** 도입
- `RabbitTemplate.convertAndSend()` 로 메시지 발행
- 모든 Spring 인스턴스의 `@RabbitListener` 가 수신
- 각자 자기 `SimpMessagingTemplate` 으로 자기 WebSocket 구독자에게 broadcast

**결과**: 두 인스턴스가 같은 broker 를 거치므로 메시지 일관성 확보

---

### 3. 실시간 채팅 메시지가 새로고침해야 표시됨

**증상**: HTTPS 배포 후 채팅 실시간 미동작

**원인**:
- nginx 에 `/ws` 경로용 `Upgrade` 헤더 누락 → WebSocket 핸드셰이크 실패
- SockJS fallback 시 `proxy_buffering` 때문에 streaming transport 깨짐

**해결**: nginx `/ws` location 분리 + `proxy_buffering off` + `proxy_read_timeout 86400s` + Upgrade 헤더 추가

---

### 4. 메시지 50개 초과 시 표시 누락

**증상**: 채팅 메시지가 50개를 넘어가면 일부가 화면에 안 보임

**원인**: 매퍼가 `ORDER BY created_datetime ASC LIMIT 50` 으로 가장 오래된 50개만 가져왔고, 클라이언트도 페이지 0 만 호출

**해결**:
- 매퍼를 `ORDER BY ... DESC` 로 변경 → 최신 50개를 가져옴
- 서비스에서 `Collections.reverse()` 로 시간순 정렬
- 클라이언트에 **무한 스크롤 페이징** 구현 (scrollTop < 80 시 page++ prepend, 스크롤 위치 보존)

---

### 5. 결제 시 IP 차단 (BootPay APP_FIREWALL_BLOCKED)

**증상**: 결제 요청 시 `접근이 허가된 IP가 아닙니다` 401 응답

**해결**: BootPay 콘솔 → 결제설정 → 연동키 및 보안에서 EC2 Elastic IP 추가

---

### 6. HTTPS 적용 (Let's Encrypt)

**배경**: 초기 mkcert(self-signed) 사용 → 브라우저 신뢰 불가 + WebSocket 핸드셰이크 실패

**해결**: `certbot --nginx` 로 Let's Encrypt 정식 인증서 발급, nginx 자동 설정, 90일 자동 갱신 등록

---

## 🤝 협업

| 항목 | 내용 |
|---|---|
| 기간 | 2026.04 ~ 2026.05 |
| 인원 | Web X명 / AI 1명 (개인) |
| 도구 | GitHub, Notion, Figma, Discord |

---

## 🎓 회고

### 잘한 점
- **분산 환경 트러블슈팅을 정석적으로 해결**: OAuth state 는 쿠키로, 채팅 메시지는 RabbitMQ 로 옮겨 stateless 아키텍처 완성
- **CI/CD 자동화**: GitHub Actions 로 master push 시 두 EC2 에 자동 배포
- **실시간 안정화**: SockJS fallback 까지 고려한 nginx 설정

### 아쉬운 점
- **테스트 코드 부재**: 시간상 단위·통합 테스트를 충분히 작성하지 못함
- **로그 수집 시스템 미구축**: 운영 환경 에러 추적이 SSH 의존적. CloudWatch · ELK 같은 중앙 로그 필요

### 배운 점
- **무상태성은 분산 환경의 시작점**: 세션·캐시·state·메시지 모두 외부 저장소로 옮겨야 한다는 점을 실전으로 학습
- **인프라가 곧 애플리케이션 품질**: nginx 한 줄 설정이 백엔드 코드보다 큰 영향을 줄 수 있다

---

🔗 **AI 서버 리포지토리**: [bideo-ai](https://github.com/your-org/bideo-ai)
