# TRD — 명지대학교 규정 AI 챗봇

| 항목 | 내용 |
|---|---|
| 문서 | 기술 요구사항 정의서 (Technical Requirements Document) |
| 버전 | v1.0 (구현 반영) · 2026-11-14 |
| 이전 버전 | v0.1 초안 — [명지대규정챗봇_TRD.md](../명지대규정챗봇_TRD.md) |
| 관련 문서 | [PRD.md](PRD.md) · [청킹전략.md](청킹전략.md) (원본 설계 [명지대규정_청킹전략.md](../명지대규정_청킹전략.md)) · [README](../README.md) |

v0.1 초안을 구현하면서 바뀐 설계와 측정값을 반영했습니다. 초안과 달라진 곳은 **[변경]** 으로 표시합니다.

## 1. 기술 스택

| 영역 | 선택 | 비고 |
|---|---|---|
| Frontend | Next.js 16 (App Router) · React 19 · TypeScript · Tailwind CSS 4 · TanStack Query 5 | SSE는 `fetch` + `ReadableStream` |
| Backend | Python 3.12+ · FastAPI · Pydantic v2 · Uvicorn | 로컬 검증은 Python 3.14 |
| LLM | Upstage `solar-pro4` (OpenAI 호환 엔드포인트 `https://api.upstage.ai/v1`) | 질의 분석, 답변 생성, 평가 채점. 일반·JSON 구조화·스트리밍 호출 확인 |
| 임베딩 | `dragonkue/BGE-m3-ko` (1024차원, 정규화, 최대 512토큰) | 프로세스 내 상주 |
| 리랭커 | `BAAI/bge-reranker-v2-m3` | **[변경]** GPU가 있을 때만 사용. CPU에서는 경량 재정렬(5.2 ④) |
| 저장소 | Supabase Postgres + `pgvector`(HNSW) + `pgroonga` / **로컬 모드** SQLite + numpy | **[변경]** 같은 인터페이스(`store/base.py`)의 두 구현 |
| 인증 | Supabase Auth (익명 + 이메일) / 로컬 모드: 백엔드 발급 익명 토큰 + 관리자 비밀번호 | |
| 오케스트레이션 | 순수 Python 상태 객체(`RagState`) + 비동기 제너레이터 | LangGraph 미사용 |

**초안에서 "구현 시 확인할 것"으로 남긴 항목**

| 항목 | 결과 |
|---|---|
| `solar-pro4` 모델 ID·엔드포인트 | 확인. ID `solar-pro4`, OpenAI 호환 `/v1/chat/completions` |
| `solar-pro4` 응답 속도 | 질의 분석 약 1.4초, 답변 생성 약 0.7~2초. 전체 응답 p50 2.7초, p95 4.8초 (97문항) |
| `pgroonga` 사용 가능 여부 | **미확인.** Supabase 프로젝트를 만들지 못해 로컬 모드로 개발했습니다 |
| CPU 리랭크 속도 | 쌍 하나 약 1초. 후보 20개면 약 20초 → 실시간 경로에서 사용 불가 |

## 2. 시스템 구조

```
[브라우저]
   │ HTTPS · SSE
   ▼
[Next.js]  ── (Supabase 모드) Supabase Auth
   │ Authorization: Bearer <JWT>
   ▼
[FastAPI]
   ├─ RAG 파이프라인 ── solar-pro4 (Upstage API)
   ├─ 임베딩 BGE-m3-ko · 리랭커(GPU일 때)
   ├─ 색인 (PDF → 청크 → 임베딩 → 항 보조 벡터)
   └─ 평가 러너 (Golden Set 97문항)
   │
   ▼
[Store]  Supabase: Postgres(pgvector, pgroonga) · Auth
         로컬:    backend/data/local.db (SQLite) + 메모리 벡터 행렬 + BM25 역색인
```

| 모드 | 조건 | 저장·검색 | 인증 |
|---|---|---|---|
| 로컬 | `SUPABASE_DB_URL` 비움 | SQLite. 활성 청크와 항 보조 벡터를 메모리 행렬로 올려 numpy로 코사인 계산. 키워드는 자체 BM25 | `/v1/auth/anonymous`가 익명 JWT 발급, 관리자는 `/v1/auth/admin` 비밀번호 |
| Supabase | `SUPABASE_DB_URL` 설정 | Postgres + `hybrid_search` SQL 함수 | Supabase JWT 검증(HS256 또는 JWKS), 관리자는 `app_metadata.role = 'admin'` |

파이프라인·API·화면은 두 모드가 같습니다. **Supabase 모드 코드는 작성했지만 실제 프로젝트에서 실행하지 않았습니다.**

## 3. 데이터 모델

마이그레이션: `supabase/migrations/20261114000001_init.sql`, `20261114000002_chunk_parts.sql`. 로컬 모드는 `store/local.py`의 `SCHEMA`.

### 3.1 테이블

| 테이블 | 용도 | 초안 대비 |
|---|---|---|
| `regulations` | 규정(PDF 1개), `audience text[]` | `updated_at` 추가 |
| `regulation_versions` | 개정·롤백 단위, `content_hash`, `status` | `error` 추가 |
| `chunks` | 검색 단위(조문). `content`(규정명 접두, 임베딩용) · `body`(표시용) · `embedding vector(1024)` · `is_active` | 같음 |
| `chunk_parts` | **[변경]** 항 단위 보조 벡터 `(chunk_id, seq, content, embedding)` | 신규 ([청킹전략.md](청킹전략.md) 5장) |
| `conversations` · `messages` · `message_citations` · `feedback` | 대화·인용·피드백 | `messages.verified`, `messages.meta` 추가 |
| `query_logs` | 질문별 분석·후보·지연·토큰 | `user_id`, `status`, `answer` 추가 |
| `rate_events` | **[변경]** 사용자별 속도 제한 기록 | 신규 |
| `golden_items` · `eval_runs` · `eval_results` | 평가 | `eval_runs.status` 추가 |

함수: `hybrid_search`, `activate_version(version_id)`(버전 전환을 한 트랜잭션에서), `purge_old_logs(days)`(질의 로그 보존 기간).

### 3.2 행 수준 보안

초안과 같습니다. 모든 테이블에 RLS를 켜고, 사용자 데이터는 본인만, 규정·청크는 누구나 읽기, 관리 테이블은 관리자만 읽기입니다. `chunk_parts`는 정책 없이 RLS만 켜서 서버(service role)만 접근합니다.

### 3.3 하이브리드 검색 함수 **[변경]**

```sql
hybrid_search(query_text, query_embedding, match_count = 20, candidate_count = 30,
              filter_regulation_ids = null, rrf_k = 60, keyword_weight = 1.0)
  returns (chunk_id, dense_rank, keyword_rank, dense_score, rrf_score)
```

- **Dense 점수 = max(조문 벡터, 항 보조 벡터)**: `dense_chunk`와 `dense_part`를 합쳐 청크별 최댓값으로 순위를 매깁니다.
- `keyword_weight`: RRF에서 키워드 순위 가중치. 0이면 Dense 단독. 평가 결과 기본값은 리랭크를 할 때 1.0, 안 할 때 0.0입니다(5.2 ③).
- `dense_score`를 함께 돌려줘 리랭크 없이도 거부 판정(⑤)과 경량 재정렬(④)에 씁니다.
- `query_text`는 핵심어를 OR로 묶은 문자열입니다.

## 4. 색인 파이프라인

[청킹전략.md](청킹전략.md)에 단계별 처리, 실험 근거, 통계가 있습니다. 요약:

1. `pypdf` 추출 → 정제 → 조·부칙·별표 분리 → 시행일만 있는 부칙 제거
2. 1조 = 1청크, 500자 초과만 항 → 문장 → 공백 순 분할(겹침 50자), `[규정명]` 접두
3. 임베딩 후 새 버전으로 저장
4. **[변경]** 항이 3개 이상인 긴 조문은 항마다 보조 벡터 생성
5. `activate_version`으로 무중단 전환. `content_hash`가 같으면 건너뜀

| 항목 | 값 |
|---|---|
| 규정 / 청크 / 보조 벡터 | 239 / 7,571 / 2,370 |
| CPU 전체 색인 시간 | 약 75분 (다운로드·추출·임베딩). 보조 벡터는 추가 약 14분 |
| 일괄 색인 | `python -m scripts.ingest_all` (바뀐 규정만, `--force` 전체) |
| 보조 벡터 채우기 | `python -m scripts.build_parts` |
| 대상 과정(`audience`) | 규정명 규칙으로 초기 지정(`store/base.guess_audience`) 후 관리자 화면에서 수정. 현재 공통 183, 교직원 43, 학부 5, 대학원 4, 특수대학원 2, 전문대학원 1, 학부·대학원 1 |

## 5. RAG 파이프라인

### 5.1 흐름

```
질문 + 대화 이력(최근 3턴) + 화면에서 고른 과정
  ▼ ① 질의 분석 (solar-pro4 JSON 1회)  의도 · 독립 질문 · 하위 질문 · 대상 과정 · 핵심어
  │    잡담으로 분류됐어도 질문 형태면 규정 질문으로 되돌림
  ├─ 잡담 ───────────────────────► 안내 문구
  ▼ ①-b 기관명 해석 [변경]  규정에 없는 '○○대학원' → 규정상 대학원·전공
  ▼ ② 범위 필터 [변경]  질문에 드러난 과정 > 화면 선택 > 미상(필터 없음)
  ▼ ③ 검색  하위 질문마다 Dense 30 (+ Keyword 30) → RRF → 20
  ▼ ④ 재정렬 [변경]  GPU: 리랭커 / CPU: 경량 재정렬 → 하위 질문별 1위부터 교차 선택
  ▼ ⑤ 판정  최고 점수 < 임계값 → Step-back + 필터 해제로 1회 재검색 → 그래도 없으면 거부
  ▼ ⑥ 컨텍스트  같은 조 조각 병합 · 번호 부여
  ▼ ⑦ 생성 (solar-pro4 스트리밍, temperature 0)  [n] 인용 · [용어 해석] 반영
  │    범위를 좁혔는데 "찾을 수 없음" → reset 이벤트, 필터 해제 후 ③부터 1회 재시도 [변경]
  ▼ ⑧ 인용 검증 (규칙)  <n>→[n] 정규화 · [n] 범위 · 수치 대조
  ▼ 저장 (메시지 · 인용 · 질의 로그)
```

LLM 호출은 질문당 기본 2회(분석 + 생성), 재시도 경로에서 최대 3회입니다. 97문항 평균 2.02회.

### 5.2 단계별 명세

**① 질의 분석** (`rag/analyze.py`)

- 출력 `QueryAnalysis(intent, standalone_question, sub_questions, audience, keywords)`.
- 대상 과정은 질문·대화에 드러난 것만 씁니다. 화면에서 고른 과정을 그대로 옮기지 않도록 프롬프트에 명시했습니다. 교원·직원의 인사·복무·출장·외부강의·행동강령은 `교직원`입니다.
- 규정 질문을 잡담으로 오분류하는 경우가 있어(예: 숙박비 상한 질문), 질문 형태(`?`, 몇·얼마·언제·기준·학점 등)이거나 25자를 넘으면 규정 질문으로 처리합니다.

**①-b 기관명 해석 [변경]** (`rag/entities.py`)

사용자가 쓴 이름이 규정 명칭과 다르면 LLM이 근거 조문을 받고도 "그런 대학원은 없다"고 거부합니다. 예: "AI정보과학대학원" → 규정상 "기록정보과학전문대학원 AI정보과학전공"(학칙 별표3·7).

1. 규정 원문에 3회 이상 나오는 구체적인 대학원명 사전을 만듭니다(10분 캐시).
2. 질문의 `○○대학원`이 사전에 있으면 그 이름으로 과정만 정합니다(전문대학원 / 일반대학원 / 그 외 특수대학원).
3. 사전에 없으면 어간("AI정보과학")으로 `어간+전공|학과`를 원문에서 찾아, 바로 앞(160자 안)에 나오는 대학원명을 소속으로 투표합니다. 한 대학원으로 분명히 모일 때만 씁니다(2위 득표의 2배 이상).
4. 반영: 검색용 질문의 이름을 규정 명칭으로 바꾸고, 과정 일반 표현("전문대학원의 수업연한")으로 바꾼 하위 질문을 추가합니다(최대 4개). 질의 분석의 과정이 `미상` 또는 대학원 계열일 때만 과정을 바꿉니다(교직원 질문은 유지).
5. 생성 단계에는 사용자의 원래 표현과 `[용어 해석]`을 함께 넘기고, 답변 첫 문장에서 한 번만 밝히게 합니다.

학부에도 같은 이름이 많아(부동산, 심리치료 등) 사용자가 "대학원"이라고 쓴 경우에만 소속을 추정합니다.

**② 범위 필터 [변경]** (`rag/retrieve.apply_filter`)

| 질문에 드러난 과정 | 화면 선택 | 적용 범위 |
|---|---|---|
| 있음 | 무관 | 질문의 과정 + 공통 |
| 미상 | 있음 | 화면 선택 + 공통 |
| 미상 | 없음 | 필터 없음 |

화면 선택은 브라우저에 저장돼 다음 질문에도 남습니다. 초안처럼 화면 선택을 우선하면, `전문대학원`을 눌러 둔 채 "교직원이 외부강의를 하면 언제까지 신고?"를 묻는 경우 교직원 행동강령이 범위에서 빠져 거부됩니다(실제 발생).

**③ 검색** (`rag/retrieve.search`, `store.hybrid_search`)

- 하위 질문마다 따로 검색하고 결과를 따로 유지합니다.
- **키워드 결합 평가** (검색 평가, 리랭크 없음):

| 키워드 가중치 | 0 (Dense 단독) | 0.2 | 0.35 | 0.5 | 1.0 |
|---|---|---|---|---|---|
| Hit@4 | **0.979** | 0.957 | 0.957 | 0.947 | 0.947 |

"학부·학기·학생" 같은 흔한 단어가 키워드 점수를 끌어올려 손해를 봤습니다. 고유 명칭 문항(특수대학원 연구과정, 1차시당 등)은 Hybrid가 나았습니다. 리랭크 없이는 이득이 없어 CPU 모드 기본값은 0입니다.

**④ 재정렬 [변경]** (`rag/retrieve.rerank`)

| `RERANK_MODE` | 동작 | 용도 |
|---|---|---|
| `full` | 후보 20개 전체를 리랭커로 채점 | GPU |
| `light` | RRF 상위 5개만 리랭커 | 중간 |
| `off` | 리랭커 없이 경량 재정렬 | CPU |
| `auto` (기본) | CUDA가 있으면 `full`, 없으면 `off` | |

경량 재정렬 점수:

```
adjusted = dense_score
         + 0.04 × (질문 핵심어 중 조문에 들어 있는 비율)
         − 0.04 × (조문 제목이 특정 기관·사업용인데 질문에 그 기관이 없으면 1)
```

- 특정 기관·사업용 규정: 제목에 `대학`·`교육원`·`사업단`·`센터`·`계약학과`·`학점은행제`·`공개강좌`·`일학습병행` 등이 들어간 규정(예: 미래융합대학 학사처리 내규). 일반 질문에 그 기관의 예외 기준(출석 3분의 2, 원격수업 50%)이 끼어드는 것을 막습니다.
- 가중치는 검색 평가로 정했습니다. 판정(⑤)에는 원래 Dense 점수를 씁니다.

**선택 개수와 순서 [변경]**

| 상황 | 개수 |
|---|---|
| 단일 질문, 리랭크 off | 6 (`TOP_K_SINGLE_OFF`) — 순위 정밀도가 낮아 넓힘 |
| 단일 질문, 리랭크 on | 4 |
| 하위 질문 | 질문당 3, 합계 상한 8 |

하위 질문 결과는 **하위 질문별 1위 → 2위 → … 순서로 교차**해서 담습니다. 초안처럼 질문 순서대로 이어 붙여 상한에서 자르면, 마지막 하위 질문의 1위 근거가 잘립니다(실제 발생: "전문대학원 수업연한" 하위 질문의 1위 학칙 제11조가 빠짐).

**⑤ 판정과 재검색**

- 리랭크 off: 최고 Dense 코사인 유사도 < `DENSE_REJECT_THRESHOLD`(0.50).
- 리랭크 on: 최고 리랭크 점수 < `REJECT_THRESHOLD`(0.02). **리랭크 평가를 끝내지 못해 이 값은 보정하지 않았습니다.**
- 임계값 보정 근거(항 보조 벡터 적용 후): 정답 문항의 최고 유사도 최솟값 0.506, 하위 5% 0.600, 정답없음 문항 최댓값 0.565. 두 분포가 겹치므로 정답 문항을 거부하지 않는 0.50으로 두고, 정답없음은 LLM의 "규정에서 찾을 수 없습니다" 응답으로 거부합니다(평가에서 정답없음 3문항 모두 거부).
- 재검색: Step-back 질문과 핵심어를 LLM으로 다시 만들고, 범위 필터를 해제해 1회.

**⑥ 컨텍스트** — 초안과 같습니다. 같은 조의 분할 조각을 `seq` 순으로 병합(겹침 제거), 1,500자를 넘으면 선택 조각과 앞뒤 1개. 형식 `<n> [규정명] 제N조(제목) 본문`.

**⑦ 생성** (`rag/prompts.ANSWER_SYSTEM`) — 초안 규칙에 아래를 추가했습니다.

| 추가 규칙 | 해결한 문제 |
|---|---|
| `<1>` 꺾쇠가 아니라 `[1]`로 인용 | 모델이 컨텍스트 표기를 그대로 써 인용률 0.914 |
| 과정이 미상이고 학생 학사 질문이면 학부 기준, 다른 과정 차이는 한 문장만 | 과정별 나열로 채점 오답 |
| 교직원·위원회·도서관 질문은 과정 언급 안 함 | 불필요한 과정 안내 |
| 질문 대상의 전용 규정이 있으면 그 규정으로 답하고 다른 규정의 과정별 기준을 나열하지 않음 | 외국인학생 입학시기에 학칙의 학사과정 기준을 섞음 |
| 일반 기준을 먼저, 예외는 물을 때만. 특정 기관 기준을 일반 기준처럼 쓰지 않음 | 출석 기준을 미래융합대학 기준(3분의 2)으로 답함 |
| `[용어 해석]`을 따르고, 명칭이 다르다는 이유로 거부하지 않음. 해석 문장은 답변에 옮기지 않음 | AI정보과학대학원 거부 |

입력에는 대상 과정과 그 출처(사용자 선택 / 질문에서 추정 / 미상)를 함께 넣습니다.

**생성 후 범위 확장 재시도 [변경]** — 범위 필터를 적용한 상태에서 LLM이 "찾을 수 없음"으로 답하고 재시도를 쓰지 않았으면, `reset` 이벤트를 보내고 필터를 해제해 ③부터 다시 검색·생성합니다. 범위 밖 규정도 Dense 유사도가 임계값을 넘으면 ⑤에서 걸러지지 않기 때문입니다.

**⑧ 인용 검증** (`rag/verify.py`)

- `<n>` → `[n]` 정규화.
- 거부 판정: 인용 없이 "찾을 수 없" 류로 답하면 거부(표현이 조금 달라도).
- `[n]` 범위, 답변 속 숫자+단위(학점·개월·일·%·원·분의 등)가 인용 조문에 있는지 대조. 실패하면 "근거 확인 필요" 표시.

### 5.3 채택하지 않은 기법

| 기법 | 판단 | 근거 |
|---|---|---|
| CPU 실시간 리랭크 | 미채택 | 쌍당 약 1초, 후보 20개에 약 20초 |
| CPU에서 키워드 결합(Hybrid) | 미채택 (가중치 0) | 모든 가중치에서 Dense 단독보다 Hit@4 낮음 |
| 청크 크기 확대, Parent-Child | 미채택 | 청킹 실험에서 이득 없음 |
| HyDE / HyQE / LLM 정련 | 보류 | LLM 호출 증가. 남은 오답 2건은 검색이 아니라 생성·채점 쪽 문제 |
| 실시간 LLM 근거 채점 | 미채택 | 실험에서 Faithfulness가 1.000으로 포화 |

## 6. API

모든 엔드포인트는 `Authorization: Bearer <JWT>`가 필요합니다(익명 포함). `/v1/admin/*`은 관리자만.

| 메서드 | 경로 | 설명 |
|---|---|---|
| POST | `/v1/chat` | SSE 스트리밍 답변 |
| GET · DELETE | `/v1/conversations[/{id}]` | 내 대화 목록·상세·삭제 |
| POST | `/v1/messages/{id}/feedback` | 👍/👎, 사유, 의견 |
| GET | `/v1/regulations[/{id}]` | 규정 목록, 조 단위 열람 |
| GET | `/v1/chunks/{id}` | 출처 조문 원문 |
| GET | `/v1/auth/config` · `/v1/me` | 인증 모드, 내 정보 |
| POST | `/v1/auth/anonymous` · `/v1/auth/admin` | 로컬 모드 전용 |
| GET · POST · PATCH | `/v1/admin/regulations[/{id}]` | 목록, PDF 업로드→색인, 대상 과정 수정 |
| GET · POST | `/v1/admin/regulations/{id}/versions`, `/v1/admin/versions/{id}[/activate]` | 버전 목록·상태·활성화(롤백) |
| GET · POST | `/v1/admin/eval-runs[/{id}]` | 회귀 평가 실행·결과·비교 |
| GET · POST | `/v1/admin/golden` | 평가셋 조회, 질의 로그 사례를 평가셋에 추가 |
| GET | `/v1/admin/query-logs`, `/v1/admin/dashboard` | 질의 로그, 대시보드 |
| GET | `/healthz` | 모델 로드, 리랭크 방식, 색인 상태 |

### `/v1/chat` 이벤트

| 이벤트 | 데이터 | 비고 |
|---|---|---|
| `meta` | `{conversation_id, message_id}` | |
| `status` | `{stage, loop}` — analyzing · searching · reranking · generating | |
| `citations` | `[{idx, chunk_id, regulation_title, reg_id, article, article_title, kind, preview, score, indexed_at}]` | 토큰보다 먼저 |
| `token` | `{text}` | |
| `reset` | `{reason: "widen_scope"}` | **[변경]** 클라이언트는 지금까지 받은 답변과 인용을 지움 |
| `done` | `{message_id, status, verified, issues, audience, audience_note, index_date, suggestions, timings_ms}` | `suggestions`는 빈 목록(F-08 미구현) |
| `error` | `{code, message}` | `llm_unavailable` · `internal` |

클라이언트 연결이 끊겨도 저장은 끝까지 수행합니다.

## 7. Frontend

```
app/        page.tsx(새 대화) · c/[id] · regulations · regulations/[id] · admin · login
components/ chat/   ChatView · MessageBubble · AnswerMarkdown · CitationCards · CitationPanel
                    AudienceChips · Composer · ConversationList · FeedbackButtons · Welcome
            admin/  Dashboard · RegulationAdmin · EvalPanel · QueryLogs
lib/        api.ts(SSE 파서 포함) · auth.ts · types.ts
```

- 답변의 `[n]`을 버튼으로 바꿔 출처 패널(데스크톱 우측, 모바일 하단 시트)을 엽니다. 마크다운은 HTML 비허용.
- 대상 과정 칩: 자동 / 학부 / 대학원 / 전문대학원 / 특수대학원 / 교직원. 선택은 `localStorage`에 저장(질문에 과정이 있으면 서버가 질문을 따름).
- 규정 상세 화면에서 "이 조문에 대해 질문하기"로 채팅을 시작합니다(`/?q=`).
- 관리자 평가 화면은 Dense · 경량 재정렬 · Hybrid · ReRank 단계별 Hit/MRR과 응답 지표를 직전 실행과 비교합니다.

## 8. Backend

```
app/
  main.py         lifespan에서 저장소 열기, 모델 백그라운드 로드
  core/           config.py · auth.py
  api/            chat · conversations · regulations · admin · auth
  rag/            state · analyze · entities · retrieve · context · generate · verify · pipeline · prompts
  models/         embedder · reranker · llm
  ingest/         extract · structure · chunk · parts · index
  eval/           metrics · runner · golden.json
  store/          base · local · supabase
scripts/          ingest_all · build_parts · evaluate
tests/            27개 (가짜 임베딩·리랭커·LLM으로 로컬 모드 통합 테스트 포함)
```

- 모델 추론은 `run_in_threadpool`. Uvicorn 워커 1개.
- LLM: 타임아웃 30초, 재시도 2회.
- 속도 제한: 사용자당 분당 10회, 일 200회. 질문 최대 1,000자.

### 환경 변수 (주요)

| 이름 | 기본값 | 설명 |
|---|---|---|
| `UPSTAGE_API_KEY` | – | 필수 |
| `LLM_MODEL` | `solar-pro4` | |
| `SUPABASE_DB_URL` | – | 비우면 로컬 모드 |
| `RERANK_MODE` | `auto` | full · light · off · auto |
| `DENSE_REJECT_THRESHOLD` | 0.50 | 리랭크 off일 때 거부 임계값 |
| `REJECT_THRESHOLD` | 0.02 | 리랭크 on일 때 (미보정) |
| `KEYWORD_WEIGHT` | (자동) | 리랭크 on 1.0 / off 0.0 |
| `TOP_K_SINGLE` / `TOP_K_SINGLE_OFF` / `TOP_K_SUB` / `TOP_K_MAX` | 4 / 6 / 3 / 8 | |
| `LEXICAL_BOOST` / `SCOPE_PENALTY` | 0.04 / 0.04 | 경량 재정렬 |
| `ADMIN_PASSWORD` | `admin` | 로컬 모드 관리자 — 배포 전 반드시 변경 |
| `RATE_PER_MINUTE` / `RATE_PER_DAY` | 10 / 200 | |

## 9. 평가와 품질 관리

| 항목 | 내용 |
|---|---|
| 평가셋 | 97문항 (사실형 88, 종합형 6, 정답없음 3). 근거 문구(`reg_id`, `phrase`)로 검색 정답을 자동 표시 |
| 검색 평가 | `python -m scripts.evaluate retrieval` — Dense · 경량 재정렬 · Hybrid · ReRank 단계별 Hit/Recall/MRR/NDCG@1·3·4·10, 거부 임계값 힌트 |
| 응답 평가 | `python -m scripts.evaluate full` — Correctness(`solar-pro4` 채점), 거부율, 오거부율, 출처 표기율, 인용 검증, 컨텍스트 Hit, 지연, LLM 호출 수 |
| 기법 끄기 | `--no-analysis --no-filter --no-keyword --no-retry --no-rerank`, `--keyword-weight` |
| 회귀 기준 | 직전 같은 모드 대비 Hit@4 또는 Correctness 2%p 이상 하락이면 차단 표시. 문항 대응 부트스트랩 95% CI |

### 개선 이력 (응답 평가 97문항, CPU 모드)

| 실행 | Correctness | 정답없음 거부 | 오거부 | 출처 표기 | 인용 검증 | p50 / p95 |
|---|---|---|---|---|---|---|
| ① 첫 실행 | 0.938 | 0.667 | 0.011 | 0.914 | 0.903 | 3.5초 / 9.1초 |
| ② 프롬프트·인용 정규화·잡담 가드 | 0.959 | 1.000 | 0.021 | 1.000 | 0.989 | 2.7초 / 4.8초 |
| ③ 기관명 해석·교차 선택·범위 우선순위 | 0.969 | 1.000 | 0.000 | 1.000 | 0.989 | 2.7초 / 4.8초 |
| ④ 항 보조 벡터·경량 재정렬·답변 규칙 | **0.979** | 1.000 | 0.000 | 1.000 | 0.989 | 2.8초 / 4.5초 |

④에서 ③의 오답 3건 중 q07(원격수업 20%)과 g26(외국인 입학시기)은 정답이 됐고, 컨텍스트 Recall은 0.957 → 0.979로 올랐습니다. 남은 오답 2건:
- **q22** 출석 기준: 일반 5분의 4는 맞혔으나, 군복무 휴학 기준을 "3분의 2 미만이면 불인정"으로 서술해 채점에서 틀림 처리. 의미는 같아 채점 엄격성 문제로 보입니다.
- **g07** 교수입학사정관 책임시간 감면: ③에서는 맞혔고 ④에서 다시 "찾을 수 없음". 실행마다 답이 달라지는 문항입니다.

### 검색 평가 (94문항, LLM 없음)

| 단계 | Hit@4 | Recall@4 | MRR@4 |
|---|---|---|---|
| Dense (조문 벡터) | 0.979 | 0.968 | 0.903 |
| Dense (조문 + 항 보조 벡터) | 0.989 | 0.989 | 0.920 |
| + 경량 재정렬 | **1.000** | **1.000** | **0.926** |
| ReRank (bge-reranker-v2-m3) | 미측정 | | |

주의할 점:
- 답변과 채점을 같은 모델이 합니다. 절대값보다 변경 전후 비교에 쓰고, 오답은 사람이 확인합니다.
- 평가셋 정답률이 천장(0.97~1.0)에 가까워 더 이상 차이를 구분하기 어렵습니다. 실제 사용 중 나온 오류(외부강의 신고 거부, AI정보과학대학원 거부)는 평가셋에 없던 유형입니다. 질의 로그의 👎·거부 사례를 평가셋에 추가해야 합니다(관리자 화면에서 가능).
- 경량 재정렬 가중치와 범위 패턴은 이 평가셋으로 정했으므로 과적합 가능성이 있습니다. 새 문항으로 재확인이 필요합니다.

## 10. 보안

초안과 같습니다(키는 서버에만, RLS + `user_id` 조건, 프롬프트 주입 방지 문구, 입력 1,000자 제한, 관리자만 업로드, 속도 제한, HTML 비허용 렌더링, CORS 제한, 로그 180일).

로컬 모드 추가 사항:
- 관리자 비밀번호 기본값 `admin`과 `LOCAL_JWT_SECRET` 기본값은 개발용입니다. 외부에 노출하기 전에 바꿉니다.
- `backend/.env`(Upstage 키 포함)는 `.gitignore`에 있습니다.

## 11. 배포와 운영

| 항목 | 현재 | 운영 전 할 일 |
|---|---|---|
| 실행 | `run.bat` / `stop.bat` (Windows 로컬) | 컨테이너 이미지 |
| 저장소 | 로컬 SQLite | Supabase 프로젝트 생성, 마이그레이션 2개 적용, `pgroonga` 확인, 색인 이관 |
| 추론 | CPU (리랭크 off) | GPU를 쓰면 `RERANK_MODE=full`로 리랭크 평가 후 `REJECT_THRESHOLD` 보정 |
| 모델 로드 | 서버 시작 후 백그라운드 (약 30초, `/healthz`의 `models.status`) | 이미지에 가중치 포함 |

## 12. 남은 과제

| 과제 | 이유 |
|---|---|
| Supabase 모드 실제 실행 (마이그레이션, `pgroonga`, `hybrid_search`의 항 벡터 결합, JWT) | 코드만 작성, 미검증 |
| GPU 환경에서 리랭크 평가와 임계값 보정 | CPU에서는 평가를 끝내지 못함 |
| 평가셋 확장 (실사용 오류 유형, 별표, 기관명, 후속 질문) | 현재 평가셋은 천장에 가까움 |
| 별표 표 구조 추출 | [청킹전략.md](청킹전략.md) 7장 |
| 후속 질문 제안(F-08) | `done.suggestions` 자리만 있음 |
| 기관명 해석의 Supabase 모드 성능 | 대학원명 사전을 만들 때 원문 최대 5,000건을 읽음 (10분 캐시) |
