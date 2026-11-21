# Cypher Studio — 기술 요구사항 정의서 (TRD)

| 항목 | 내용 |
|---|---|
| 문서 버전 | 1.0 (현재 구현 기준) |
| 작성일 | 2026-11-21 |
| 관련 문서 | [PRD.md](PRD.md) (제품 요구사항), [../README.md](../README.md) (실행 방법) |

---

## 1. 시스템 구성

```
 브라우저 (localhost:3000)                 백엔드 (127.0.0.1:8000)                   외부
┌──────────────────────────┐  HTTP/JSON  ┌──────────────────────────────┐  Bolt/TLS  ┌──────────────┐
│ Next.js 16 · React 19     │ ──────────▶ │ FastAPI                       │ ─────────▶ │ Neo4j / Aura │
│  - 연결·스키마 사이드바    │             │  main.py   라우팅·오류 변환     │            └──────────────┘
│  - Cypher 에디터(색상·정렬)│ ◀────────── │  graph.py  드라이버·실행·스키마 │  HTTPS     ┌──────────────┐
│  - 그래프(SVG)/표/JSON     │             │  llm.py    자연어→Cypher        │ ─────────▶ │ Upstage /    │
│  - localStorage: 기록·선택 │             │  store.py  연결 정보 저장       │            │ Anthropic    │
└──────────────────────────┘             └──────────────┬───────────────┘            └──────────────┘
                                                        │
                                            backend/data/connections.json
```

- 브라우저는 Neo4j·LLM에 직접 접속하지 않는다. 모든 접속 정보와 API 키는 백엔드에만 있다.
- 백엔드는 `127.0.0.1`에만 바인딩한다(로컬 단일 사용자 도구).

### 1.1 기술 스택

| 영역 | 기술 | 버전 |
|---|---|---|
| Frontend | Next.js (App Router, Turbopack), React, Tailwind CSS, TypeScript | 16.4.6, 19.3.0, 4.3.3, 5 |
| Backend | Python, FastAPI, Uvicorn, Pydantic, neo4j Python driver, httpx, certifi | 3.14, 0.162.4, 0.54.0, 2.14.3, 6.4.1, 0.28.1, 2026.11.20 |
| LLM | Upstage `solar-pro4`(기본) / Anthropic Claude(`anthropic` SDK 2.8.0) | |
| DB | Neo4j 5.x / 2025.x, Neo4j Aura | 검증: Aura 5.27 Enterprise |

그래프 시각화, 문법 색상, 포매터는 외부 라이브러리 없이 직접 구현했다.

## 2. 디렉터리 구조

```
backend/
  app/main.py        FastAPI 앱, 엔드포인트, 예외 → HTTP 응답 변환, 스키마 캐시
  app/graph.py       드라이버 캐시, TLS 신뢰 설정, 문장 분리·판별, 실행, 결과 변환, 스키마 조회
  app/llm.py         LLM 제공자 선택, Upstage/Claude 호출, Cypher 생성 프롬프트
  app/store.py       연결 정보 JSON 저장소(URL 정규화)
  app/models.py      요청·응답 Pydantic 모델
  sample_data.cypher 실습 07 한글 요리 그래프
  requirements.txt, .env.example
frontend/src/
  app/page.tsx       화면 골격, 연결·상태·스키마 로딩
  components/
    Sidebar.tsx        연결 목록, 스키마 브라우저
    ConnectionModal.tsx 연결 등록·수정·테스트
    CypherStudio.tsx   자연어 입력, 에디터, 실행, 결과 탭, 기록
    CypherCode.tsx     문법 색상 에디터(CypherEditor), 읽기 전용 블록
    GraphView.tsx      SVG force 레이아웃 그래프
    ResultTable.tsx    결과 표, 변경 건수 표시
    ui.tsx             버튼·모달·토스트·변경 확인 창
  lib/
    api.ts             API 클라이언트·타입
    cypher.ts          Cypher 토크나이저, 포매터
    graph.ts           레이블 색, 노드 캡션, 그래프 병합, 값 표시
start-all.bat, start-backend.bat, start-frontend.bat
```

## 3. 백엔드 API

모든 응답은 JSON. 오류 응답 형식: `{"detail": str, "code"?: str, "hint"?: str}`.

| Method | Path | 요청 | 응답 |
|---|---|---|---|
| GET | `/api/health` | | `{ok, ai_enabled, ai_provider, ai_model}` |
| GET | `/api/connections` | | `Connection[]` |
| POST | `/api/connections` | `ConnectionIn` | `Connection` |
| PUT | `/api/connections/{id}` | `ConnectionIn` (password `null`이면 유지) | `Connection` |
| DELETE | `/api/connections/{id}` | | `{deleted: true}` |
| POST | `/api/connections/test` | `ConnectionIn` + `id?` | `ServerInfo` |
| GET | `/api/connections/{id}/status` | | `ServerInfo` |
| GET | `/api/connections/{id}/schema?refresh=bool` | | `GraphSchema` |
| POST | `/api/connections/{id}/execute` | `{cypher, allow_write}` | `ExecuteResult` |
| POST | `/api/connections/{id}/ai/cypher` | `{prompt, history: [{prompt, cypher}]}` | `{cypher, explanation, kind}` |

### 3.1 주요 타입

```ts
Connection   { id, name, uri, username, database: string|null, has_password: boolean }   // 비밀번호 없음
ServerInfo   { agent, version, edition, database, address }
GraphSchema  { labels: SchemaItem[], relationship_types: SchemaItem[],
               patterns: {from, type, to}[], indexes[], constraints[],
               total_nodes, total_relationships, server_version }
SchemaItem   { name, count, properties: {name, types: string[]}[] }
ExecuteResult{ results: ResultSet[], graph: GraphData, query_types: string[] }
ResultSet    { statement, columns, rows: Json[][], row_count, truncated, query_type,
               counters: Record<string, number>, notifications: {title, description}[], elapsed_ms }
GraphData    { nodes: GNode[], relationships: GRel[], truncated }
GNode        { _type: "node", id /*elementId*/, labels, properties }
GRel         { _type: "relationship", id, type, start, end, properties }
```

결과 값 직렬화: Node/Relationship/Path는 위 객체로, 시간 타입은 ISO 문자열, 공간 타입은 문자열, `NaN/Infinity`는 문자열, 바이트는 `0x…` 16진수.

### 3.2 HTTP 상태 코드

| 코드 | 상황 |
|---|---|
| 400 | Cypher 문법·실행 오류(`code`=Neo4j 오류 코드), 잘못된 URL, 빈 입력 |
| 401 | Neo4j 인증 실패(힌트: Aura 사용자 이름은 인스턴스 ID일 수 있음) |
| 404 | 등록되지 않은 연결 ID |
| 409 | `code: "WRITE_CONFIRM"` — 변경 쿼리인데 `allow_write=false` |
| 429 | Neo4j 로그인 시도 과다로 일시 차단 |
| 502 | LLM 오류(키 없음·인증·한도·응답 해석 실패), 드라이버 오류 |
| 503 | Neo4j 접속 불가(DNS 실패는 별도 안내) |

## 4. 핵심 설계

### 4.1 연결과 드라이버

- `store.py`가 연결을 `backend/data/connections.json`에 저장한다(임시 파일에 쓴 뒤 교체, 스레드 락). 경로는 `CONNECTIONS_FILE` 환경 변수로 바꿀 수 있다.
- 드라이버는 연결 ID별로 1개를 캐시한다(`connection_timeout=10s`, 풀 10). 연결을 수정·삭제하면 드라이버와 스키마 캐시를 함께 버린다.
- **TLS 신뢰**: `+s` 스킴은 기본 스킴 + `encrypted=True` + 사용자 지정 `ssl_context`로 바꿔 연결한다. 이 컨텍스트는 OS 인증서 저장소에 certifi(Mozilla) 루트를 더한다.
  - 이유: Windows는 일부 공인 루트를 필요할 때만 내려받는다. 그래서 Aura가 쓰는 `SSL.com TLS RSA Root CA 2022`가 Python에 보이지 않아 "Unable to retrieve routing information"으로 실패했다.

### 4.2 안전 실행 (`graph.execute`)

```
입력 Cypher
  └─ split_statements: 문자열·백틱·주석 밖의 ; 로 분리
      └─ 문장별 _classify: "EXPLAIN <문장>"을 READ 세션에서 실행 → summary.query_type
           r(읽기) · rw/w(쓰기) · s(스키마) · ?(판별 불가: SHOW, CYPHER/USE 접두어 등)
      ├─ 쓰기/스키마가 있고 allow_write=false → 409 WRITE_CONFIRM (실행하지 않음)
      └─ 실행
           allow_write=false → READ 액세스 모드 세션
               (판별 불가 문장이 쓰기를 시도하면 서버가 AccessMode 오류 → 409로 변환)
           allow_write=true  → WRITE 세션
           ├─ IN TRANSACTIONS 포함   → 문장별 auto-commit
           ├─ 스키마 문장 + 여러 문장 → 문장별 트랜잭션(스키마·데이터 혼합 금지 제약)
           └─ 그 외                  → 전체 단일 트랜잭션(하나 실패 시 전체 롤백)
           트랜잭션 timeout 30s, 문장당 1,000행
  └─ _connect_nodes: 결과 노드 간 관계 보충, 속성 없는 끝 노드 채우기 (그래프 300노드 이하)
```

- `:param` 같은 Neo4j Browser 전용 명령은 실행 전에 400으로 거부한다.
- 문법 오류(`SyntaxError`)와 파라미터 누락은 판별 단계에서 그대로 보고한다.
- 쓰기 실행 후에는 해당 연결의 스키마 캐시를 버린다.

### 4.3 결과 변환 (`GraphCollector`)

- 모든 Node/Relationship/Path를 재귀적으로 JSON으로 바꾸면서 그래프용 노드·관계를 모은다(노드 최대 1,000개, 넘으면 `truncated`).
- 관계의 끝 노드는 드라이버가 속성 없이 주는 경우가 있다. 이때는 "빈 노드"로 넣어 두고, 나중에 같은 노드가 레이블·속성과 함께 오면 교체한다.
- **주의**: neo4j `Node`는 `Mapping`이라 속성이 없으면 falsy다. 존재 여부는 반드시 `is not None`으로 검사한다. 이 실수로 관계 끝 ID가 비어 선이 그려지지 않던 버그가 있었다.
- `_connect_nodes`: 결과 노드 ID 목록으로 `MATCH (a)-[r]->(b) WHERE elementId(a) IN $ids AND elementId(b) IN $ids RETURN r LIMIT 2000`을 실행해 관계를 보충한다(Neo4j Browser의 "connect result nodes"와 같은 동작). 빈 노드는 `MATCH (n) WHERE elementId(n) IN $ids RETURN n`으로 채운다. 표·JSON 결과에는 영향이 없다.

### 4.4 스키마 조회와 캐시

| 정보 | 쿼리 |
|---|---|
| 레이블·관계 타입 | `db.labels()`, `db.relationshipTypes()` (각 최대 200개) |
| 개수 | `MATCH (n:L) RETURN count(n)` 등 — count store 사용으로 빠름 |
| 속성·타입 | `db.schema.nodeTypeProperties()`, `db.schema.relTypeProperties()` |
| 관계 패턴 | 관계 타입별 500건 샘플에서 `(시작 레이블, 타입, 끝 레이블)` 추출 |
| 인덱스·제약조건 | `SHOW INDEXES`, `SHOW CONSTRAINTS` |
| 서버 버전 | `driver.get_server_info()` |

- 결과는 연결별로 5분 캐시한다(`SCHEMA_TTL=300`). `?refresh=true` 또는 쓰기 실행 시 무효화한다.
- 같은 결과를 AI 프롬프트용 텍스트(`schema_text`)로도 만든다. 데이터 값은 넣지 않는다.

### 4.5 자연어 → Cypher (`POST …/ai/cypher`)

```
cached_schema → schema_text + server_version
  └─ llm.generate_cypher(prompt, schema_text, version, history)   # 프런트엔드가 직전 3개 질문·쿼리를 보냄
        구조화 출력: {cypher, explanation, kind}
  └─ graph.check_generated: 문장별 EXPLAIN
        ├─ 오류 → 오류 메시지를 붙여 1회 재생성(repair) → 다시 판별
        └─ kind = 플래너 판별값(schema > write > read), 판별 불가 시 모델 값
```

- **제공자**: `LLM_PROVIDER`가 `upstage`/`anthropic`이면 그 값을 쓴다. 비어 있으면 `UPSTAGE_API_KEY`, 그다음 `ANTHROPIC_API_KEY` 순으로 고른다.
- **Upstage**: `/v1/chat/completions`, `response_format=json_schema(strict)`. 기본은 추론을 끈다(`reasoning_effort` 미전송). JSON 해석 실패 시 1회 재시도한다.
  - 측정(같은 질문): medium 19.9초 / low 13.4초 / 끔 1.5~2.0초. 숨은 추론 토큰이 1,150 / 808 / 0개였다.
- **Claude**: `beta.messages.parse`, 구조화 출력, `output_config.effort=CLAUDE_EFFORT`(기본 low), 서버 측 fallback beta 사용.
- **프롬프트 핵심 규칙**: 스키마에 있는 이름만 사용 / 관계 방향은 패턴 목록 그대로(예시 포함) / 그래프 요청은 `MATCH p = … RETURN p` / 대소문자 불확실하면 `toLower(…) CONTAINS` / 최신 Cypher 문법(`IS NOT NULL`, `elementId`, `COUNT {}`) / 열린 조회는 `LIMIT 25`(그래프)·`100`(표) / `$파라미터` 금지(입력 수단 없음) / 삭제는 `DETACH DELETE` + `WHERE` / 설명은 마크다운 없이 한두 문장.
- 프런트엔드는 생성 결과를 포매터로 정렬한 뒤 에디터에 넣고, `kind == read`이고 자동 실행이 켜져 있으면 실행한다. 실행 단계에서도 4.2의 판별이 다시 적용되므로, 모델이 종류를 틀려도 확인 없이 쓰기가 실행되지 않는다.

### 4.6 Cypher 토크나이저·포매터·색상 (`lib/cypher.ts`, `CypherCode.tsx`)

- **토크나이저**: 공백, 주석(`//`, `/* */`), 문자열(`'…'`, `"…"`, 이스케이프), 백틱 이름, `$파라미터`, 숫자, 유니코드 식별자(한글 포함), 기호. 닫히지 않은 문자열·주석도 예외 없이 끝까지 처리한다.
- **식별자 분류**:
  - `:`, `|`, `&` 뒤의 식별자는 `(`/`[` 안이면 레이블/관계 타입이다. 단 `{}` 안의 맵 값은 제외한다.
  - `.` 뒤(속성 접근)와 `:` 앞(맵 키)은 키워드로 보지 않는다. 예: `n.limit`, `{limit: 1}`
  - `(`가 이어지면 함수다.
- **포매터 규칙**:
  - 절 키워드(`MATCH`, `OPTIONAL MATCH`, `WHERE`, `WITH`, `RETURN`, `ORDER BY`, `SKIP`, `LIMIT`, `UNWIND`, `CREATE`, `MERGE`, `SET`, `DELETE`, `DETACH DELETE`, `REMOVE`, `CALL`, `UNION` …) 앞에서만 줄을 바꾼다.
  - `()`/`[]` 안이나 `CASE … END` 안에서는 줄을 바꾸지 않는다. `STARTS WITH`/`ENDS WITH`는 절로 보지 않는다.
  - `ON CREATE SET`/`ON MATCH SET`과 `{ }` 서브쿼리 본문은 들여쓴다.
  - 키워드는 대문자로 바꾸고, 문장 사이에는 빈 줄을 둔다.
  - 절 내부 간격은 원문을 유지하고, 연속 공백만 하나로 줄인다.
- **불변 조건**(테스트됨): 공백과 키워드 대소문자를 무시한 토큰 열이 정렬 전후 같다(의미 보존). 정렬을 두 번 해도 결과가 같다(멱등).
- **색상 에디터**: 투명 글자의 `<textarea>`를 색칠한 `<pre>` 위에 겹친다. 두 요소는 글꼴, 줄 간격, 안쪽 여백, 줄바꿈 규칙(`pre-wrap`, `overflow-wrap:anywhere`), `scrollbar-gutter:stable`이 같고 스크롤도 함께 움직인다. 높이는 내용에 맞춰 7~24줄로 늘어난다.

### 4.7 그래프 시각화 (`GraphView.tsx`)

- **렌더링**: SVG. 노드는 반지름 32px 원, 캡션 12px, 관계 타입 11.5px. 같은 노드 쌍의 여러 관계는 ±44px씩 휜 곡선으로, 자기 자신으로의 관계는 루프로 그린다.
- **레이아웃**: 자체 force simulation(`requestAnimationFrame`).

  | 힘 | 값 |
  |---|---|
  | 반발 | 800·α/d² (900px 밖은 무시) |
  | 스프링 | 링크 길이 130px, 강도 0.2 |
  | 중심 인력 | 0.06, 가로 방향은 화면 비율(높이/너비, 0.35~1)만큼 약하게 |
  | 감쇠 | 0.55, α 감소 0.985/tick, α<0.015에서 정지 |

  처음 그릴 때 120 tick(400노드 초과 시 40)을 미리 계산한다. 위 값은 노드 10~31개 샘플 그래프로 오프라인 비교해 정했다: 노드 간 최소 거리가 지름 이상이면서 레이아웃이 가장 촘촘한 조합.
- **카메라**:
  - 레이아웃이 안정될 때까지 화면에 자동으로 맞춘다. 자동 맞춤은 55% 미만으로 축소하지 않는다.
  - 사용자가 확대, 이동, 드래그를 하면 자동 맞춤을 멈춘다.
  - 수동 확대/축소 범위는 10~500%다.
- **증분 갱신**: 새 결과의 노드가 모두 화면에 있으면 위치를 유지한다. 일부만 새 노드면(이웃 펼치기) 이웃 근처에 배치한다. 겹치는 노드가 없으면 새로 배치한다.
- **조작**: 노드 드래그(놓은 자리에 고정), 배경 드래그(이동), 휠(커서 기준 확대). 클릭하면 속성 패널, 더블클릭하면 이웃 펼치기(`MATCH (n)-[r]-(m) WHERE elementId(n)=… LIMIT 50`). 전체 화면, 단축키 `+ - 0 F Esc`.

### 4.8 클라이언트 상태

| 키 (localStorage) | 내용 |
|---|---|
| `cypherstudio.active` | 마지막 선택 연결 ID |
| `cypherstudio.history` | `{conn, prompt, cypher, at}[]` 최대 100개, 연결별 표시 |

`useSyncExternalStore`로 SSR과 하이드레이션 결과를 맞추고, localStorage 접근 실패는 무시한다.

## 5. 설정

### `backend/.env`

| 변수 | 기본값 | 설명 |
|---|---|---|
| `LLM_PROVIDER` | (자동) | `upstage` 또는 `anthropic` |
| `UPSTAGE_API_KEY` | | Upstage 키 |
| `UPSTAGE_MODEL` | `solar-pro4` | |
| `UPSTAGE_REASONING_EFFORT` | (비움 = 끔) | `low`/`medium`/`high`로 추론 켜기. 10~20초로 느려짐 |
| `UPSTAGE_BASE_URL` | `https://api.upstage.ai/v1` | |
| `ANTHROPIC_API_KEY` | | Claude 키 |
| `CLAUDE_MODEL` | `claude-opus-5-5` | |
| `CLAUDE_EFFORT` | `low` | |
| `CORS_ORIGINS` | `http://localhost:3000` | 쉼표 구분 |
| `CONNECTIONS_FILE` | `backend/data/connections.json` | 연결 저장 위치 |

### `frontend/.env.local`

| 변수 | 기본값 |
|---|---|
| `NEXT_PUBLIC_API_URL` | `http://localhost:8000` |

### 코드 상수

| 상수 | 값 | 위치 |
|---|---|---|
| `TX_TIMEOUT` | 30초 | graph.py |
| `MAX_RESULT_ROWS` | 1,000 | graph.py |
| `MAX_GRAPH_NODES` | 1,000 | graph.py |
| `CONNECT_LIMIT` | 300 | graph.py |
| `SCHEMA_LIMIT` | 200 | graph.py |
| `SCHEMA_TTL` | 300초 | main.py |

## 6. 보안

| 항목 | 현재 구현 |
|---|---|
| 비밀 정보 위치 | Neo4j 비밀번호와 LLM 키는 백엔드에만 있다. API 응답에는 `has_password`만 포함한다. |
| 저장 | `connections.json`, `.env`, `.env.local`은 `.gitignore`로 제외한다. 비밀번호는 **평문 저장**이다(향후 OS 자격 증명 저장소 검토). |
| 네트워크 | 백엔드는 `127.0.0.1` 바인딩, CORS는 프런트엔드 주소만 허용한다. |
| 쿼리 안전 | 4.2의 판별 + READ 세션 + 확인 창으로 이중 방어한다. 사용자가 입력한 Cypher는 그대로 실행하므로(SQL 콘솔과 같은 성격) DB 계정 권한이 최종 경계다. |
| 주입 | 앱이 만드는 쿼리(관계 보충, 이웃 펼치기 등)는 파라미터 또는 이스케이프한 문자열을 쓴다. 레이블·타입 이름은 백틱으로 감싼다. |
| 외부 전송 | LLM에는 질문, 최근 3개 질문·쿼리, 스키마 요약(이름·타입·개수)만 보낸다. |

## 7. 실행과 배포

- **Windows**: `start-all.bat`이 백엔드·프론트엔드를 각각 새 창에서 띄우고 브라우저를 연다.
  - 배치 파일은 **UTF-8 + CRLF** 형식이다. LF만 쓰면 cmd에서 한글이 깨진다.
  - `if ( … )` 블록 안의 문구에는 괄호를 쓰지 않는다. cmd가 그 괄호를 블록 끝으로 해석한다.
- **백엔드**: `.venv`가 없으면 만들고 `requirements.txt`를 설치한다. `uvicorn --reload --reload-dir app`으로 실행하므로 코드가 바뀌면 자동으로 재시작한다.
- **프론트엔드**: `node_modules`가 없으면 `npm install`, 이어서 `npm run dev`를 실행한다.

## 8. 테스트와 검증

| 영역 | 방법 | 결과 (2026-11-21) |
|---|---|---|
| 정적 검사 | `npx eslint src`, `npx tsc --noEmit`, `npx next build` | 통과 |
| 문장 분리 | 문자열 안 `;`, 이스케이프, 백틱, 주석 케이스 | 통과 |
| 포매터 | 7개 케이스(CASE, STARTS WITH, 서브쿼리, ON CREATE, 다중 문장, `:Order` 레이블, 주석) — 의미 보존·멱등 | 통과 |
| 안전 실행 | 실제 Aura에서 `CREATE`/`SET`/`CREATE INDEX` 미확인 실행 → 409, 문법 오류 → 400 | 통과 |
| 그래프 결과 | 노드만 / 관계만 / 경로 반환 — 모든 관계 끝이 노드와 연결 | 통과 |
| 샘플 데이터 | `sample_data.cypher` 2회 실행(두 번째는 생성 0건), 실습 문서의 예상 결과(08-3, 08-4, 09-3, 09-5, Q5, Q6)와 비교 | 일치 |
| 생성 품질·속도 | 자연어 10문항(실습 Q1~Q6, 집계, 그래프 요청) × 2회 | 8~9/10, 평균 2.5~2.8초 |
| UI | Playwright: 연결 등록·실패 표시, 그래프·속성 패널·확대·전체 화면, 변경 확인 창, 색상 에디터 레이어 정렬(긴 줄 포함) | 확인 |

### 수동 회귀 점검 목록

1. 잘못된 URL / 비밀번호 / 없는 호스트로 연결 테스트 → 각각 다른 안내가 나오는가
2. `MATCH (p:사람)-[r:좋아함]->(f:요리) RETURN p, r, f` → 관계 선이 그려지는가
3. `CREATE (:임시)` 실행 → 확인 창이 뜨고, 취소하면 아무것도 생기지 않는가
4. 생성된 쿼리가 정렬·색상 표시되는가, Shift+Alt+F가 동작하는가
5. 백엔드 코드를 바꾸면 자동 재시작되는가

## 9. 알려진 제약

- 자동 생성은 추론을 끈 상태의 정확도 한계가 있다(관계 방향, 표/그래프 형태 선택). 필요하면 `UPSTAGE_REASONING_EFFORT`로 추론을 켠다.
- 스키마·데이터 혼합 스크립트는 문장별 커밋이라 원자적이지 않다.
- `elementId()`가 없는 Neo4j 4.x에서는 관계 보충과 이웃 펼치기가 동작하지 않는다. 오류는 무시하고 반환된 그래프만 그린다.
- 그래프 force 계산은 O(n²)이라 수백 노드 이상에서는 느려질 수 있다.
- 단일 사용자 로컬 도구다. 인증, 다중 사용자, 원격 배포를 고려하지 않았다.
