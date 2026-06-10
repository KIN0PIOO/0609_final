# Multi-Agent DB Migration System

> Oracle 레거시 DB의 오브젝트 이관 및 SQL 변환·튜닝·포맷팅을 LLM 기반 멀티 에이전트로 자동화하는 시스템

---

## 프로젝트 개요

| 항목 | 값 |
|---|---|
| **언어** | Python 3.11 |
| **AI Framework** | LangChain 0.2.0 / LangGraph 0.2.34 |
| **빌드 도구** | pip / requirements.txt |
| **데이터베이스** | Oracle Database |
| **DB 드라이버** | oracledb 2.1.0 |
| **LLM** | OpenAI Compatible API (GLM-5.1 / GPT), Anthropic (Claude) |
| **Frontend** | Streamlit ≥ 1.35.0 |
| **로깅** | Python logging (migration_agent logger) |
| **환경 설정** | python-dotenv (.env) |

---

## 기술 스택

| 구분 | 기술 | 설명 |
|---|---|---|
| **Frontend** | Streamlit | 대시보드·챗봇·모니터링 UI |
| | Plotly | 작업 현황 차트 시각화 |
| **AI / LLM** | LangGraph (ReAct) | 멀티 에이전트 오케스트레이션 및 Tool 루프 |
| | LangChain | LLM 추상화, 프롬프트 관리, Tool 바인딩 |
| | OpenAI API | SQL 변환·튜닝·이관 추론, 챗봇 Function Calling |
| | Anthropic API | LLM 폴백 |
| **RAG** | FAISS | 벡터 유사도 검색 |
| | BAAI/bge-m3 | 텍스트 임베딩 모델 |
| **Database** | Oracle Database | 작업 큐·변환 결과·실행 로그 저장 |
| | oracledb | Python Oracle 드라이버 |
| **Infra** | subprocess / PID file | 에이전트 프로세스 제어 (시작·정지·일시정지) |
| | python-dotenv | 환경변수 기반 설정 관리 |

---

## 아키텍처

```text
graph TB
  subgraph Client["Client"]
    Web["웹 대시보드 (Streamlit)\n모니터링 / 챗봇 / 설정"]
  end

  subgraph Backend["Backend (Python)"]
    AgentControl["Agent Control\n(시작 / 정지 / 일시정지)"]

    subgraph SupervisorLayer["Supervisor (LangGraph ReAct)"]
      Supervisor["Supervisor Agent\n- poll_jobs\n- flush_cycle_metrics\n- request_wait"]
    end

    subgraph Agents["실행 에이전트"]
      MigAgent["Migration Agent\n(DB 오브젝트 이관)"]
      ConvAgent["SQL Conversion Agent\n(레거시 → TO-BE SQL)"]
      TuneAgent["SQL Tuning Agent\n(RAG 기반 최적화)"]
      FmtAgent["SQL Formatting Agent\n(SQL 포맷 정리)"]
    end

    subgraph RAG["RAG 엔진"]
      FAISS["FAISS Vector DB"]
      Embed["임베딩 모델\n(BAAI/bge-m3)"]
    end

    subgraph LLM["LLM"]
      OpenAI["OpenAI Compatible API\n(GLM-5.1 / GPT)"]
      Anthropic["Anthropic API\n(Claude)"]
    end

    subgraph DataLayer["데이터 접근 계층"]
      Repo["Repository\n(oracledb)"]
    end
  end

  subgraph DB["Database"]
    Oracle[("Oracle DB\nNEXT_MIG_INFO\nNEXT_SQL_INFO\nNEXT_MIG_LOG\nAG_AGENT_RUN_METRICS")]
  end

  Web --> AgentControl
  Web --> Supervisor
  AgentControl --> Supervisor
  Supervisor --> MigAgent
  Supervisor --> ConvAgent
  Supervisor --> TuneAgent
  Supervisor --> FmtAgent

  ConvAgent --> FAISS
  TuneAgent --> FAISS
  FAISS --> Embed

  MigAgent --> LLM
  ConvAgent --> LLM
  TuneAgent --> LLM
  FmtAgent --> LLM

  MigAgent --> Repo
  ConvAgent --> Repo
  TuneAgent --> Repo
  FmtAgent --> Repo
  Supervisor --> Repo

  Repo --> Oracle
```

---

## 패키지 구조

```
.
├── main.py                          # 진입점 — SupervisorAgent 실행
├── requirements.txt
│
├── app/                             # Frontend (Streamlit)
│   ├── app.py                       # 메인 앱, 사이드바 (에이전트 선택·제어)
│   ├── pages/
│   │   ├── dashboard.py             # 메인 대시보드 + Supervisor 챗봇
│   │   ├── mig_monitor.py           # Migration 작업 모니터링
│   │   ├── sql_monitor.py           # SQL Conversion 모니터링
│   │   ├── tuning_monitor.py        # SQL Tuning 모니터링
│   │   ├── agent_metrics.py         # 에이전트 실행 메트릭 대시보드
│   │   ├── job_detail.py            # 작업 상세 조회
│   │   ├── rag_manager_page.py      # RAG 데이터 관리 UI
│   │   ├── settings_page.py         # 환경설정 UI
│   │   ├── system_health.py         # 시스템 상태 모니터링
│   │   └── xml_export.py            # XML 내보내기
│   └── utils/
│       ├── agent_control.py         # 에이전트 프로세스 제어 (start/stop/pause/resume)
│       ├── db.py                    # 프론트용 DB 조회·재실행·폴링 함수
│       ├── env_manager.py           # .env 읽기/쓰기
│       ├── rag_db.py                # RAG 벡터 DB 조회
│       └── rag_manager.py           # RAG 인덱스 관리
│
├── server/                          # Backend
│   ├── agents/
│   │   ├── supervisor/              # Supervisor (LangGraph ReAct 오케스트레이터)
│   │   │   ├── agent.py             # SupervisorAgent 클래스, 사이클 루프
│   │   │   ├── graph.py             # LangGraph 그래프 빌드
│   │   │   ├── prompts.py           # Supervisor 시스템 프롬프트
│   │   │   └── state.py             # SupervisorState 타입 정의
│   │   ├── migration/               # Migration Agent
│   │   │   ├── orchestrator.py      # MigrationOrchestrator (작업 진입점)
│   │   │   ├── executor.py          # SQL 실행 엔진
│   │   │   ├── graph.py             # Migration LangGraph 그래프
│   │   │   ├── scheduler.py         # 배치 스케줄링
│   │   │   ├── sql_utils.py         # SQL 유틸리티
│   │   │   ├── state.py             # 상태 타입
│   │   │   └── verifier.py          # 실행 결과 검증
│   │   ├── sql_conversion/
│   │   │   └── agent.py             # SqlConversionAgent
│   │   ├── sql_tuning/
│   │   │   └── agent.py             # SqlTuningAgent
│   │   └── sql_formatting/
│   │       └── agent.py             # SqlFormattingAgent
│   │
│   ├── tools/                       # Supervisor LangGraph Tool 정의
│   │   ├── __init__.py              # 외부 공개 Tool 목록
│   │   ├── context.py               # 공유 상태·레지스트리·메트릭·정지 플래그
│   │   ├── poll.py                  # poll_jobs Tool (DB 폴링 + 레지스트리 갱신)
│   │   ├── cycle.py                 # flush_cycle_metrics, request_wait Tool
│   │   ├── migration.py             # run_data_migration Tool
│   │   ├── sql_conversion.py        # run_sql_conversion Tool
│   │   ├── sql_tuning.py            # run_sql_tuning Tool
│   │   └── sql_formatting.py        # run_sql_formatting Tool
│   │
│   ├── services/
│   │   ├── migration/               # Migration 비즈니스 로직
│   │   │   ├── llm_client.py        # LLM 호출 클라이언트
│   │   │   ├── prompt_service.py    # 프롬프트 생성
│   │   │   └── domain_models.py     # 도메인 모델
│   │   └── sql/                     # SQL 변환·튜닝 비즈니스 로직
│   │       ├── agents.py            # SQL 에이전트 서비스
│   │       ├── llm_service.py       # LLM 호출
│   │       ├── prompt_service.py    # 프롬프트 생성
│   │       ├── correct_sql_rag_service.py  # Correct SQL RAG 검색
│   │       ├── tobe_sql_tuning_service.py  # TO-BE SQL 튜닝
│   │       ├── sql_formatting_service.py   # SQL 포맷팅
│   │       ├── binding_service.py   # 바인드 변수 처리
│   │       ├── validation_service.py # 결과 검증
│   │       ├── db_runtime.py        # DB 런타임 실행
│   │       ├── xml_parser_service.py # XML 파싱
│   │       ├── domain_models.py     # 도메인 모델
│   │       └── workflow/            # SQL LangGraph 워크플로우
│   │           ├── graph.py
│   │           └── state.py
│   │
│   ├── repositories/                # DB 접근 계층 (oracledb)
│   │   ├── migration/
│   │   │   ├── repository.py        # Migration 작업 조회·갱신
│   │   │   └── history_repository.py # 이관 이력
│   │   ├── sql/
│   │   │   ├── result_repository.py  # SQL 변환·튜닝 결과 저장
│   │   │   ├── mapper_repository.py  # 단순 매핑 룰 조회
│   │   │   ├── complex_mapper_repository.py # 복잡 매핑 룰 조회
│   │   │   └── log_repository.py    # SQL 실행 로그
│   │   └── supervisor/
│   │       └── metrics_repository.py # 에이전트 실행 메트릭 저장
│   │
│   └── core/                        # 공통 유틸리티 및 공유 컴포넌트
│       ├── db.py                    # DB 커넥션 팩토리
│       ├── db_migration.py          # DB 스키마 마이그레이션 유틸
│       ├── llm.py                   # LLM 클라이언트 팩토리
│       ├── llm_fallback.py          # LLM 폴백 처리
│       ├── logger.py                # 로거 설정
│       └── exceptions.py            # 공통 예외 정의
│
├── scripts/                         # 운영·초기화 스크립트
│   ├── init_db.py                   # DB 연결·테이블 존재 여부 점검
│   ├── seed_mig_rules.py            # Migration 룰 시드 데이터 삽입
│   ├── create_sql_rules_table.py    # SQL 룰 테이블 생성
│   ├── create_sql_log_table.py      # SQL 로그 테이블 생성
│   └── create_sql_complex_map_table.py # 복잡 매핑 테이블 생성
│
├── tests/
│   └── test_xml_parser_service.py
│
└── runtime/                         # 런타임 상태 파일 (자동 생성)
    # agent.pid         — 실행 중인 에이전트 PID
    # agent.pause       — 일시정지 플래그 파일
    # agent.wake        — 즉시 실행 신호 파일
    # target_job.json   — 챗봇 재실행 대상 job (1회성)
    # chats/            — 챗봇 대화 기록
```

---

## Resource 구조

```
.env                        # 환경변수 설정 파일 (DB, LLM, RAG 등)
.env.example                # 환경변수 템플릿
│
# ── DB 접속 정보 ──────────────────────────────────────────
# DB_USER / DB_PASS / DB_HOST / DB_PORT / DB_SID
# ORACLE_CLIENT_PATH        — Thick 모드 Oracle Client 경로
│
# ── LLM 설정 ─────────────────────────────────────────────
# LLM_PROVIDER              — openai | anthropic
# LLM_API_KEY / OPEN_API_KEY
# LLM_BASE_URL              — OpenAI Compatible 엔드포인트
# LLM_MODEL                 — 사용할 모델명 (예: GLM-5.1)
# LLM_MAX_TOKENS
│
# ── RAG / 임베딩 설정 ─────────────────────────────────────
# RAG_EMBED_BASE_URL        — 임베딩 API 엔드포인트
# RAG_EMBED_API_KEY
# RAG_EMBED_MODEL           — 기본값: BAAI/bge-m3
# TOBE_SQL_TUNING_TOP_K     — RAG 검색 상위 K개 (기본값: 3)
│
# ── DB 테이블명 설정 ──────────────────────────────────────
# MAPPING_RULE_TABLE        — 기본값: NEXT_MIG_INFO
# MAPPING_RULE_DETAIL_TABLE — 기본값: NEXT_MIG_INFO_DTL
# RESULT_TABLE              — 기본값: NEXT_SQL_INFO
# AGENT_METRICS_TABLE       — 기본값: AG_AGENT_RUN_METRICS
│
# ── 에이전트 선택 (프론트 사이드바 연동) ─────────────────
# DB_MIGRATION_ONLY         — true: Migration만 실행
# SQL_CONVERSION_ONLY       — true: SQL 변환만 실행
# SQL_TUNING_ONLY           — true: SQL 튜닝만 실행
# SQL_FORMATTING_ONLY       — true: SQL 포맷팅만 실행
# SUPERVISOR_MODE           — true: 챗봇 Supervisor 모드 활성화
# (모두 false이면 전체 에이전트 실행)
│
# ── Supervisor 설정 ───────────────────────────────────────
# SUPERVISOR_RECURSION_LIMIT — LangGraph 재귀 한도 (기본값: 10000)
```

---

## 실행 방법

```bash
# 1. 환경 설정
cp .env.example .env
# .env 에 DB_USER, DB_PASS, DB_HOST, LLM_API_KEY 등 입력

# 2. 패키지 설치
pip install -r requirements.txt

# 3. DB 연결 확인
python scripts/init_db.py

# 4. Backend (Supervisor Agent) 실행
python main.py

# 5. Frontend (Streamlit) 실행
streamlit run app/app.py
```
