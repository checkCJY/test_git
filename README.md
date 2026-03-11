graph TB
    subgraph "🧑‍💻 Client Layer (Browser)"
        UI[UI Components<br/>리뷰 조회 / 분석 버튼]
        API[api.js<br/>Axios / JWT 자동 부착]
    end

    subgraph "⚙️ Django + DRF Layer"
        MW[Middleware<br/>JWT Auth / Pagination]
        VIEWS[API Views<br/>todo, accounts, reviews]
    end

    subgraph "🔄 Async Task Layer (Celery + Redis)"
        REDIS[/(Redis Docker)/<br/>Broker DB:0 / Backend DB:1]
        WORKER[Celery Worker<br/>-P solo / 백그라운드 대기]
    end

    subgraph "🤗 AI Model"
        HF[HuggingFace Model<br/>finetuned-nsmc-sentiment]
    end

    subgraph "🐘 Data Layer"
        DB[(PostgreSQL Docker)<br/>todo, accounts, stg_movie_reviews]
        JUPYTER[Jupyter Crawler<br/>영화 리뷰 스크래핑]
    end

    %% 연결선 및 흐름
    UI -->|1. 감정 분석 클릭| API
    API -->|2. POST /sentiment-async/| MW
    MW --> VIEWS
    
    VIEWS <-->|DB 읽기/쓰기| DB
    JUPYTER -->|크롤링 데이터 적재| DB
    
    VIEWS -->|3. Task 큐 등록| REDIS
    VIEWS -.->|4. 202 Accepted<br/>task_id 즉시 반환| UI
    
    REDIS -->|5. Task 전달| WORKER
    WORKER -->|6. 텍스트 전달| HF
    HF -->|7. 긍정/부정 추론| WORKER
    WORKER -->|8. 결과 저장| REDIS
    
    UI -.->|9. 800ms 폴링<br/>GET /sentiment-result/| VIEWS
    VIEWS -.->|10. 결과 조회| REDIS

