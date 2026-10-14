# law-api — 6주차 실습: 법제처 법령정보 Open API 수집

법제처 국가법령정보 Open API로 **검색(법령ID) → 본문조회 → 재사용 함수 → FastAPI 서버 → SQLite 캐시** 파이프라인을 만든 실습 결과.

| 파일 | 역할 |
|---|---|
| `step1_search.py` | 목록조회 `lawSearch.do`로 "전자금융거래법" 검색 → 법령명·**법령ID** 출력 |
| `step2_detail.py` | 본문조회 `lawService.do`에 법령ID(`010199`)를 넘겨 현행 조문 출력 |
| `law_client.py` | 위 두 호출을 재사용 함수 `search_law` · `get_law_detail`로 정리. 인증키는 `.env`의 `LAW_OC`에서 읽음 |
| `explore.py` | 응답 JSON을 `ensure_ascii=False, indent=2`로 통째로 찍어 구조 확인 |
| `main.py` | FastAPI 서버 — `/search`, `/detail/{law_id}` (+ lifespan에서 `init_db`, 상세조회 캐시) |
| `db.py` | SQLite `law.db`의 `laws`·`articles` 테이블 생성 · UPSERT 저장 · 조회 |

실행 환경: Python 3.13, requests · fastapi · uvicorn[standard] · python-dotenv

```bash
python -m venv .venv && source .venv/bin/activate
pip install requests fastapi "uvicorn[standard]" python-dotenv
echo "LAW_OC=본인OC" > .env          # 커밋하지 않음 (.gitignore)
python law_client.py                 # 함수 자체 테스트
python db.py                         # law.db 테이블 생성
uvicorn main:app --reload            # http://127.0.0.1:8000/docs
```

> `step1_search.py` · `step2_detail.py` · `explore.py`는 `.env`를 읽지 않으므로 실행 전 파일 안의 `"본인OC"`를 실제 OC로 바꿔야 한다(커밋본은 자리표시자로 되돌려 둠).

## ① 결과 확인

- **단계 1** `python step1_search.py` → `법령명: 전자금융거래법 / 법령ID: 010199`, `법령명: 전자금융거래법 시행령 / 법령ID: 010366`
- **직접 깨뜨려 보기** OC를 `wrong-oc-xyz`로 바꾸면 0건으로 조용히 끝나지 않고 `호출 실패: 사용자 정보 검증에 실패하였습니다. → OC 값·요청 IP 등록·승인 상태를 확인`으로 멈춤
- **단계 2** `python step2_detail.py` → 주소에 `ID=010199`, 출력은 `제1장 총칙` → `제1조(목적) 이 법은 전자금융거래의 …` → `제2조(정의) …` 순
- **단계 3** `python law_client.py` → 검색 결과 리스트 + 앞 3개 조문. `.env`의 값을 비우면 `.env에 LAW_OC=내OC 한 줄을 넣어주세요 (5.3절)`만 출력하고 멈춤
- **단계 4** `/search?query=전자금융거래법` → `{"count":2,"results":[…]}`, `/detail/010199` → `article_count: 80`, `/docs`에 `GET /search` · `GET /detail/{law_id}` 두 엔드포인트
- **단계 5** `/detail/010199`를 두 번 호출
  - 1회차: `"source": "api(law.go.kr)"`, `article_count: 80`
  - 2회차: `"source": "db(cache)"`, `name: ""`, `fetched_at: 2026-10-14T20:15:06…` (본인 OC로 재실행) → 응답 키가 바뀐 것이 DB에서 왔다는 증거
  - `law.db`: `laws` 1행, `articles` 80행

## ② 핵심 개념 되새김

- **인증키(OC)** — 법제처가 "누가, 어디(등록 IP)에서, 승인된 범위 안에서" 부르는지 확인하는 신분증. 서버는 요청 사이의 상태를 기억하지 않으므로(무상태) 매 요청마다 붙여야 하고, 그래서 호출량 관리·차단도 키 단위로 이뤄진다.
- **목록조회 → 상세조회 2단 호출** — 목록조회는 키워드로 후보 법령과 그 **법령ID**만 돌려주고, 본문(조문)은 주지 않는다. 본문조회는 법령ID를 받아 한 법령의 **현행 조문 전체**를 돌려준다. 법령ID는 개정돼도 바뀌지 않는 법 자체의 번호이고, 법령일련번호(MST, 예: `290247`)는 개정본마다 새로 붙는 번호라 본문조회에는 법령ID를 넣는다.
- **DB 캐시** — 같은 법령을 다시 물을 때 법제처를 또 부르지 않아 응답이 빨라지고 호출량(특히 공유 `test` 계정·일일 한도)을 아낀다. 저장 시각(`fetched_at`)이 남아 언제 받은 데이터인지 이력도 쌓인다. 단, 캐시는 읽기(`load_law`)와 쓰기(`save_law`)가 한 짝이어야 동작한다.

### 워크북 기록 칸

- **단계 0** — OC: 처음엔 `test` 데모 계정으로 진행 → 본인 OC 승인 확인 후 `.env`의 `LAW_OC`를 **본인 값**으로 교체하고 단계 1~5를 다시 실행해 같은 결과 확인(검색 2건 · 조문 80개 · `api` → `db(cache)`). (신청 일자: ____)
- **단계 1** — 전자금융거래법 / 법령ID `010199`
- **단계 2** — 목록조회는 법령명·법령ID 같은 "목록 정보"만 주고 조문을 주지 않으므로, 조문을 얻으려면 그 ID로 본문조회를 따로 불러야 한다.
- **단계 3** — 법령ID는 법 자체의 고정 번호(`010199`), 법령일련번호는 개정본별 번호(`290247`). 본문조회에는 **법령ID**를 넣는다.
- **단계 4** — ① 검색·본문조회 호출, 오류 처리, 1건일 때 dict→list 처리를 다시 짤 필요가 없었다. ② FastAPI가 함수의 경로와 타입 힌트(`query: str`, `law_id: str`)를 읽어 OpenAPI 스키마를 만들고 그것으로 `/docs`를 생성했다.
- **단계 5** — ① 첫 요청 때 `save_law`로 조문이 `law.db`에 저장됐고, 두 번째 요청은 `load_law`가 DB에서 찾아 바로 반환했기 때문. ② 저장을 안 하면 다음 요청에서도 `load_law`가 `None`을 돌려줘 매번 API 갈래로 간다.

## ③ 자유 로그

- **응답 구조 확인(explore)** — `LawSearch.law[]` 아래 항목마다 16개 키. `법령일련번호`(290247)는 위쪽, `법령ID`(010199)는 열 줄쯤 아래에 떨어져 있고, `법령상세링크`의 `MST=290247`이 법령일련번호와 같은 값임을 확인. 본문은 `법령.조문.조문단위[]`의 `조문내용`이며, 장·절 제목 행(`제1장 총칙`)도 조문단위로 섞여 온다. 제3조처럼 항·호만 있는 조문은 제목 한 줄만 나온다.
- **막힌 곳** — 단계 5에서 서버를 새로 켰는데 응답에 `source`가 없었다. 원인은 단계 4 서버가 8000번 포트를 계속 잡고 있어 새 서버가 `address already in use`로 뜨지 못한 것. 이전 서버를 끄고 다시 켜니 해결.
- **AI 활용** — Claude Code에 워크북의 단계 1~5와 제출 준비를 수행하게 했다. 검증은 각 단계의 "확인/예상 결과"와 실제 출력을 대조(법령ID 010199, 80개 조문, source 전환, DB 행 수)하고, `git status`에서 `.env`가 빠졌는지 확인하는 방식으로 했다.
- (여기에 오늘 배운 것·아직 안 풀린 것을 본인 말로 추가)
