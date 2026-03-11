## 🏗️ System Architecture

**DRF TodoList × 영화 리뷰 감정분석 프로젝트**의 전체적인 요청 흐름과 기술 스택 구조입니다.

### 📍 Request Flow (비동기 감정분석 흐름)

| Step | 과정 | 발신 (From) | 수신 (To) | 상세 설명 |
| :---: | :--- | :--- | :--- | :--- |
| **1** | **Todo/리뷰 요청** | 사용자 (Browser) | `DRF Views` | Todo 작성 및 크롤링된 리뷰 목록(`stg_movie_reviews`) 조회 |
| **2** | **JWT 인증** | `api.js` (Interceptor) | `DRF Middleware` | 모든 API 요청 시 localStorage의 Access Token을 Header에 자동 부착 |
| **3** | **감정분석 요청** | 리뷰 분석 버튼 클릭 | `POST /sentiment-async/` | 선택한 리뷰 텍스트를 비동기 엔드포인트로 전송 |
| **4** | **Task Queue 등록**| `DRF View` | `202 Accepted` | Celery 태스크 큐에 작업을 등록하고 즉시 `task_id` 반환 (서버 블로킹 방지) |
| **5** | **AI 모델 추론** | `Celery Worker` | `predict_sentiment()`| Redis(DB:0)에서 태스크 수신 → HuggingFace 모델로 추론 → Redis(DB:1)에 결과 저장 |
| **6** | **결과 폴링 (Polling)**| `pollResult()` (UI) | `GET /sentiment-result/` | 프론트엔드가 800ms 간격으로 결과 조회. 완료 시 긍정/부정(Score) 화면 표시 |

<br/>

### ⚙️ System Components

#### 🧑‍💻 Client Layer (Vanilla JS)
* **`api.js`**: 공통 axios 인스턴스, JWT Bearer 토큰 자동 부착, 401 에러 시 로그인 리다이렉트
* **`todo_list.js`**: Todo CRUD, 좋아요/북마크/댓글 이벤트 위임(Event Delegation) 처리
* **`reviews_page.js`**: 리뷰 목록 페이지네이션, 감정분석 버튼 이벤트 및 800ms 간격 결과 폴링

#### ⚙️ Django + DRF Layer
* **Apps**: `accounts` (JWT 로그인/회원가입), `todo` (할일 관리), `interaction` (좋아요/댓글), `reviews` (리뷰 조회 및 감정분석)
* **Middleware & Utils**: `JWTAuthentication`, `IsAuthenticated`, `CustomPageNumberPagination`, `Q()` 객체 필터링

#### 🔄 Background Processing (Celery + Redis)
* **Celery Worker**: `analyze_review_sentiment_by_id`, `analyze_sentiment_text` 비동기 실행 (WSL CUDA 충돌 방지를 위해 `-P solo` 옵션 사용)
* **Redis (Docker)**: Message Broker (DB:0) 및 Result Backend (DB:1) 역할 수행 (Port: 6379)

#### 🤗 AI & Data Pipeline
* **HuggingFace Model**: `blockenters/finetuned-nsmc-sentiment` (NSMC 파인튜닝 모델, 711MB). `_pipe` 전역 변수 캐싱으로 초기 1회만 로딩
* **Jupyter Crawling**: BeautifulSoup 활용 영화 리뷰 수집 → SHA1 해싱으로 중복 검사(Upsert) → `stg_movie_reviews` 테이블 적재
* **PostgreSQL (Docker)**: `accounts_user`, `todo_todo`, `interaction_*` 등 서비스 메인 DB

<br/>

### ⚠️ TO-DO (보완 필요 항목)
- [ ] **DB/Data**: 필드명 불일치 수정 (name→title), DBeaver 포트 통일, DB 인덱스 추가 (collected_at, user FK)
- [ ] **Security**: JWT Refresh Token 자동 갱신 (api.js), JWT httpOnly 쿠키 전환 검토 (XSS 방어)
- [ ] **Infra/Test**: 통합 `docker-compose.yml` 구성, `.env` 분리, 앱별 테스트 코드 보완
