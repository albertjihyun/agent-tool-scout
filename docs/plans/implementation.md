# agent-tool-scout 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 매주 월요일 아침 AI 에이전트 도구 리포트를 GitHub 이슈로 보내는 3단계 파이프라인(수집 → 분석 → 발행)을 만든다.

**Architecture:** 수집과 발행은 Python 패키지 `scout`의 결정적 로직이고, 분석만 `anthropics/claude-code-action`이 구독 토큰으로 수행한다. 워크플로우는 main의 코드를 실행하고, 봇 기록은 고아 브랜치 `data`를 `_data/`에 체크아웃해 읽고 쓴다. 수집 단계를 먼저 완성해 부트스트랩 스냅샷을 쌓기 시작한 뒤 발행과 분석을 붙인다.

**Tech Stack:** Python 3.12, requests, PyYAML, pytest, responses(HTTP 목), GitHub Actions, anthropics/claude-code-action@v1

**Spec:** `docs/design.md`

## Global Constraints

- 문서, 주석, 프롬프트, 커밋 메시지, PR은 한국어. 코드 식별자는 영어.
- Anthropic API 키를 쓰지 않는다. Claude는 `CLAUDE_CODE_OAUTH_TOKEN`으로만 호출한다.
- GitHub 호출은 Actions의 `GITHUB_TOKEN`만 쓴다. 개인 토큰 secret을 추가하지 않는다.
- 봇은 main에 커밋하지 않는다. 쓰기는 `data` 브랜치(`_data/` 체크아웃)에만 한다.
- 개인 값(키워드, 기준값, 리포트 언어, 이슈 제목 형식, 멘션 대상)은 `config.yml`에만 둔다. cron 시각만 워크플로우 파일에 둔다.
- 7일 기간 = 실행일 00:00 UTC 직전까지의 완료된 7일 `[run_date - 7일, run_date)`.
- 스타 히스토리 API 헤더: `X-GitHub-Api-Version: 2026-03-10`.
- 신규 후보 스타 히스토리 조회 상한 600회, 에이전트 전달 후보 30개, Trendshift 최소 파싱 개수 15개.
- 섹션 1 최대 5개, 별첨 최대 3개.
- `pull_request_target`을 쓰지 않는다.
- 이 레포의 커밋·PR 규칙은 로컬 `CLAUDE.md`를 따른다(Conventional Commits, 작업마다 브랜치와 PR).

## Review Focus

1. **레포가 이름이 바뀌거나 삭제·비공개 전환됨**: 스타 히스토리나 트리 조회가 404/451이어도 그 레포만 건너뛰고 수집은 계속돼야 한다. (Task 7 테스트)
2. **같은 주에 정식 실행이 두 번 돔(cron 지연 후 수동 `full` 재실행 등)**: 같은 ISO 주 리포트 파일은 덮어쓰고, `reported.json`에 중복 항목이 생기지 않으며, 기준 스냅샷은 7일 전 근처의 것을 고른다. (Task 4, Task 10 테스트)
3. **검색 도중 2차 레이트 리밋(403/429)**: 대기 후 재시도해서 계속하고, 재시도를 다 써도 실패하면 예외로 워크플로우를 실패시켜 메일이 가게 한다. 조용히 빈 결과로 넘어가면 안 된다. (Task 3 테스트)
4. **에이전트가 형식을 어김**: 마커 누락이면 대체 리포트로, 섹션 1이 5개를 넘거나 후보에 없는 레포(지어낸 레포)가 있으면 그 항목을 버린다. (Task 9 테스트)
5. **빈 주**: 릴리스 0건, 제외 후 후보 0개여도 리포트와 이슈는 나가고 "해당 없음" 줄이 들어간다. (Task 7, Task 10 테스트)

---

## 파일 구조

```
pyproject.toml                    패키지와 의존성, pytest 설정
config.yml                        개인 설정 (Task 1에서 스키마 고정)
profile.md                        개인화 정보
scout/__init__.py
scout/config.py                   config.yml 로딩과 검증
scout/github.py                   GitHub REST 클라이언트 (레이트 리밋, 페이지네이션)
scout/window.py                   7일 기간 계산
scout/stars.py                    스타 히스토리 합산, 스냅샷 비교, 기준 스냅샷 선택
scout/search.py                   검색 쿼리 열거 (1,000개 제한 분할)
scout/trendshift.py               Trendshift 페이지 파싱
scout/candidates.py               증가량 결정, 제외, 순위, 보강, 분류 힌트, 부풀리기 표시
scout/collect.py                  1단계 CLI: 풀 → 스냅샷 + candidates.json
scout/report.py                   리포트 마커, 검증, 정리
scout/publish.py                  3단계 CLI: 검증 → 리포트 저장 → 이력 → 이슈
prompts/weekly-report.md          에이전트 임무
prompts/report-template.md        리포트 형식 (마커 포함)
.claude/settings.json             에이전트 허용 도구
.github/workflows/ci.yml          PR마다 pytest
.github/workflows/risk-check.yml  1회성 위험 확인
.github/workflows/weekly.yml      주간 파이프라인
tests/...                         모듈별 테스트, tests/fixtures/ 에 응답 샘플
```

`data` 브랜치: `data/snapshots/YYYY-MM-DD.json`, `data/reported.json`, `data/candidates.json`, `reports/YYYY-Www.md`.

---

### Task 1: 프로젝트 뼈대와 설정 로딩

**Files:**
- Create: `pyproject.toml`, `config.yml`, `profile.md`, `scout/__init__.py`, `scout/config.py`, `.github/workflows/ci.yml`, `.gitignore`
- Test: `tests/test_config.py`

**Interfaces:**
- Produces: `load_config(path: str | Path) -> Config`. `Config`는 frozen dataclass이고 필드는 아래 `config.yml`의 키와 같다(중첩은 하위 dataclass: `SearchConfig`, `LimitsConfig`, `InflationConfig`, `ReportConfig`). 누락·잘못된 값은 `ConfigError`.

`config.yml` 초기값 (값은 이대로 고정):

```yaml
owner_login: albertjihyun
mention_owner: false
report:
  language: ko
  issue_title_format: "[주간 리포트] {year}년 {month}월 {week_of_month}주차"
  analysis_failed_prefix: "[분석 실패]"
  label: weekly-report
search:
  keyword_queries:
    - "claude OR skill OR skills OR mcp OR codex OR agentic in:name,description"
    - "agent OR agents OR cursor OR copilot OR opencode OR harness in:name,description"
  topics: [claude-code, agent-skills, mcp, mcp-server, ai-agents, codex, claude-skills]
  star_floor: 200
  created_within_days: 365
  pushed_within_days: 7
  relevance_pattern: "claude|anthropic|\\bmcp\\b|skill|agent|codex|cursor|copilot|llm|opencode"
limits:
  star_history_cap: 600
  candidates_for_agent: 30
  trendshift_min_entries: 15
  baseline_skew_days: 3
  max_tools: 5
  max_appendix: 3
inflation:
  min_weekly_gain: 1000
  max_fork_ratio: 0.01
  max_open_issues: 5
```

- [ ] **Step 1: 실패하는 테스트 작성** — `test_load_config_reads_defaults`(위 파일을 읽어 `cfg.limits.star_history_cap == 600`, `cfg.report.language == "ko"`, `len(cfg.search.keyword_queries) == 2`), `test_missing_key_raises`(`limits` 삭제한 임시 파일 → `ConfigError`), `test_invalid_language_raises`(`language: ""` → `ConfigError`).
- [ ] **Step 2: 실패 확인** — Run: `pytest tests/test_config.py -v` / Expected: FAIL (`scout.config` 없음)
- [ ] **Step 3: `pyproject.toml`(의존성 requests, PyYAML / dev: pytest, responses), `scout/config.py`, `config.yml`, `profile.md`(설계 7장의 초기 내용), `.gitignore`(`_data/`, `out/`, `reports/draft.md`, `__pycache__/`, `.venv/`) 작성**
- [ ] **Step 4: 통과 확인** — Run: `pip install -e ".[dev]" && pytest -v` / Expected: 3 passed
- [ ] **Step 5: `ci.yml` 작성** — `pull_request`와 main `push`에서 Python 3.12로 `pytest` 실행.
- [ ] **Step 6: 커밋** — `git commit -m "build: 프로젝트 뼈대와 설정 로딩 추가"`

### Task 2: 위험 확인 워크플로우 (설계 12장)

코드 테스트가 아니라 실제 환경 확인이다. 결과에 따라 설계가 바뀔 수 있으므로 Task 3보다 먼저 머지하고 결과를 기록한다.

**Files:**
- Create: `.github/workflows/risk-check.yml`, `docs/research/risk-checks.md`

- [ ] **Step 1: 사용자 작업 안내** — 사용자가 로컬에서 `claude setup-token` 실행 → 레포 Settings > Secrets > Actions에 `CLAUDE_CODE_OAUTH_TOKEN` 등록. 완료를 확인받고 진행.
- [ ] **Step 2: `risk-check.yml` 작성** — `workflow_dispatch` 전용, `permissions: issues: write`. 세 job:
  - `notify`: `GITHUB_TOKEN`으로 `POST /repos/{repo}/issues` (제목 `위험 확인: 알림 메일 테스트`).
  - `trendshift`: `curl -sS -o page.html -w "%{http_code}" https://trendshift.io/weekly`, 상태 코드와 크기, `"initialData"` 포함 여부를 job summary에 기록.
  - `claude`: `anthropics/claude-code-action@v1`, `claude_code_oauth_token`, prompt `"reports/draft.md 파일에 ok 라고만 쓰고 끝내라"`, `claude_args: --max-turns 3 --model sonnet`. 이어서 `cat reports/draft.md`.
- [ ] **Step 3: PR 머지 후 수동 실행** — Run: `gh workflow run risk-check.yml` 후 `gh run watch` / Expected: 세 job 성공.
- [ ] **Step 4: 결과 확인** — 사용자에게 네이버 메일 도착 여부 확인. claude.ai 설정 > 사용량에서 실행 전후 주간 사용량 변화 확인.
- [ ] **Step 5: `docs/research/risk-checks.md`에 결과 기록** — 세 항목의 성공 여부, Trendshift 응답 코드, 사용량 변화. 실패 항목이 있으면 설계 12장의 대안을 적용할지 사용자와 정하고 기록.
- [ ] **Step 6: 테스트 이슈 닫기, 커밋** — `git commit -m "docs: 위험 확인 결과 기록"`

### Task 3: GitHub 클라이언트

**Files:**
- Create: `scout/github.py`
- Test: `tests/test_github.py`

**Interfaces:**
- Produces:
  - `class GitHubClient(token: str | None, session: requests.Session | None = None, sleep: Callable[[float], None] = time.sleep, max_retries: int = 5)`
  - `.get(path: str, params: dict | None = None, headers: dict | None = None) -> requests.Response` (404/451은 예외 없이 응답 반환)
  - `.search_repos(query: str, sort: str = "stars", max_pages: int = 10) -> tuple[list[dict], int]` (items, total_count)
  - `.star_history(full_name: str) -> list[dict] | None` (주 단위 배열, 404면 None)
  - `.repo(full_name) -> dict | None`, `.tree_paths(full_name, ref) -> list[str] | None`, `.releases(full_name, per_page=30) -> list[dict]`, `.create_issue(repo, title, body, labels) -> dict`, `.ensure_label(repo, name) -> None`
  - `.calls: Counter[str]` (`search`, `core` 별 호출 수)
  - `class RateLimitExhausted(Exception)`

- [ ] **Step 1: 실패하는 테스트 작성** (`responses` 사용, `sleep`은 기록만 하는 가짜)
  - `test_search_paginates_until_items_exhausted`: 1페이지 100개, 2페이지 40개 → items 140개, total_count 140, `calls["search"] == 2`.
  - `test_waits_on_secondary_rate_limit_then_succeeds`: 첫 응답 403 + `retry-after: 30`, 두 번째 200 → 성공, sleep 기록 `[30]`.
  - `test_waits_until_reset_when_remaining_zero`: 403 + `x-ratelimit-remaining: 0`, `x-ratelimit-reset`=현재+10초 → sleep 약 10초 후 재시도.
  - `test_raises_after_max_retries`: 403 6회 → `RateLimitExhausted`.
  - `test_star_history_sends_version_header_and_returns_none_on_404`: 요청 헤더에 `X-GitHub-Api-Version: 2026-03-10`, 404면 None.
- [ ] **Step 2: 실패 확인** — Run: `pytest tests/test_github.py -v` / Expected: FAIL
- [ ] **Step 3: `scout/github.py` 구현** — 재시도 대기 시간은 `retry-after` 우선, 없으면 `x-ratelimit-reset - now`, 둘 다 없으면 2^n초. 검색은 `per_page=100`.
- [ ] **Step 4: 통과 확인** — Run: `pytest tests/test_github.py -v` / Expected: 5 passed
- [ ] **Step 5: 커밋** — `git commit -m "feat(collect): 레이트 리밋 재시도를 갖춘 GitHub 클라이언트 추가"`

### Task 4: 기간과 스타 증가량 계산

**Files:**
- Create: `scout/window.py`, `scout/stars.py`
- Test: `tests/test_window.py`, `tests/test_stars.py`

**Interfaces:**
- Produces:
  - `@dataclass(frozen=True) class Window: start: date; end: date` (`end` 미포함), `window_for(run_date: date) -> Window` (`start = run_date - 7일`, `end = run_date`), `Window.iso_week_label() -> str` (`end - 1일`의 ISO 주, 예 `"2026-W45"`)
  - `gain_from_history(weeks: list[dict], window: Window) -> int`
  - `pick_baseline(snapshot_dates: list[date], run_date: date, skew_days: int) -> date | None`
  - `gain_from_snapshots(current: int, previous: int, interval_days: int) -> int` (7일 기준으로 환산, 반올림, 음수는 0)

`gain_from_history` 알고리즘: 각 주 객체의 `week`(unix 초, UTC)를 그 주 일요일 날짜로 보고, `days[i]`를 `week_date + i일`에 배정한다. 날짜가 `window.start <= d < window.end`인 값만 합한다.

- [ ] **Step 1: 실패하는 테스트 작성**
  - `test_window_for_monday`: `window_for(date(2026,11,9))` → `Window(date(2026,11,2), date(2026,11,9))`, `iso_week_label() == "2026-W45"`.
  - `test_gain_from_history_spans_two_weeks`: 일요일 2026-11-01 주 `days=[1,2,3,4,5,6,7]`, 2026-11-08 주 `days=[10,0,0,0,0,0,0]`, 기간 11/2~11/9 → `2+3+4+5+6+7+10 == 37`.
  - `test_gain_from_history_new_repo_single_week`: 주 하나만 있어도 기간 내 값만 합산.
  - `test_pick_baseline_prefers_closest_to_seven_days`: 후보 `[11/1, 11/2, 11/5]`, run 11/9, skew 3 → `11/2`.
  - `test_pick_baseline_none_when_out_of_skew`: 후보 `[10/20]` → None.
  - `test_pick_baseline_ignores_same_week_rerun`: 후보에 `11/8`(1일 전) 포함돼도 `11/2`를 고른다. (Review Focus 2)
  - `test_gain_from_snapshots_scales_interval`: `(1100, 1000, 8)` → `88`. `(900, 1000, 7)` → `0`.
- [ ] **Step 2: 실패 확인** — Run: `pytest tests/test_window.py tests/test_stars.py -v` / Expected: FAIL
- [ ] **Step 3: 구현**
- [ ] **Step 4: 통과 확인** — Expected: 7 passed
- [ ] **Step 5: 커밋** — `git commit -m "feat(collect): 7일 기간과 스타 증가량 계산 추가"`

### Task 5: 검색 풀 열거

**Files:**
- Create: `scout/search.py`
- Test: `tests/test_search.py`

**Interfaces:**
- Consumes: `GitHubClient.search_repos` (Task 3), `Config.search` (Task 1)
- Produces:
  - `enumerate_query(client: GitHubClient, base: str, star_floor: int) -> list[dict]` — `total_count > 1000`이면 스타 구간을 반으로 나눠 재귀 열거, 결과는 `full_name` 기준 중복 제거.
  - `build_queries(cfg: SearchConfig, run_date: date) -> list[str]` — 키워드 쿼리 각각과 `topic:<t>`에 `created:>=…`, `pushed:>=…` 조건을 붙인다.
  - `collect_search_pool(client, cfg, run_date) -> dict[str, dict]` (`full_name` → 검색 item)

- [ ] **Step 1: 실패하는 테스트 작성** (가짜 클라이언트로 `total_count` 제어)
  - `test_build_queries_adds_date_filters`: run 2026-11-09 → 첫 쿼리가 `created:>=2025-11-09 pushed:>=2026-11-02 stars:>=200`을 포함, 토픽 7개 쿼리 포함, 총 9개.
  - `test_enumerate_splits_when_over_1000`: `stars:>=200` 전체 1,500 → `stars:200..X`와 `stars:>X` 두 구간으로 나뉘어 호출되고 결과 합집합 반환.
  - `test_enumerate_dedupes_across_ranges`: 경계에 걸린 레포가 한 번만 나온다.
- [ ] **Step 2: 실패 확인** — Run: `pytest tests/test_search.py -v` / Expected: FAIL
- [ ] **Step 3: 구현** — 분할 기준은 현재 구간의 스타 하한과 그 구간 첫 페이지 최대 스타의 중간값.
- [ ] **Step 4: 통과 확인** — Expected: 3 passed
- [ ] **Step 5: 커밋** — `git commit -m "feat(collect): 1,000개 제한을 나눠 넘는 검색 풀 열거 추가"`

### Task 6: Trendshift 파서

**Files:**
- Create: `scout/trendshift.py`, `tests/fixtures/trendshift_weekly.html`(Task 2에서 받은 실제 페이지 또는 조사 때 받은 페이지를 잘라 저장)
- Test: `tests/test_trendshift.py`

**Interfaces:**
- Produces: `@dataclass class TrendEntry: full_name: str; description: str; stars: int; stars_gained: int; created_at: str`, `parse_trendshift(html: str) -> list[TrendEntry]`, `fetch_trendshift(session, urls: list[str]) -> tuple[list[TrendEntry], str | None]` (entries, 실패 사유)

파싱: `self.__next_f.push([1,"..."])`의 문자열 인자를 모두 JSON 디코드해 이어 붙이고, `"initialData":` 뒤의 JSON 배열을 `json.JSONDecoder().raw_decode`로 읽는다. 필드 매핑은 `docs/research/global-top-sources.md` 참고.

- [ ] **Step 1: 실패하는 테스트 작성**
  - `test_parse_fixture_returns_25_entries`: 픽스처 → 25개, 첫 항목 `full_name`에 `/` 포함, `stars_gained > 0`.
  - `test_parse_returns_empty_on_unknown_markup`: `"<html></html>"` → `[]`.
  - `test_fetch_reports_failure_reason`: 500 응답 → `([], "HTTP 500")`.
- [ ] **Step 2: 실패 확인** — Run: `pytest tests/test_trendshift.py -v` / Expected: FAIL
- [ ] **Step 3: 구현**
- [ ] **Step 4: 통과 확인** — Expected: 3 passed
- [ ] **Step 5: 커밋** — `git commit -m "feat(collect): Trendshift 주간·일간 목록 파서 추가"`

### Task 7: 후보 조립과 수집 CLI

**Files:**
- Create: `scout/candidates.py`, `scout/collect.py`
- Test: `tests/test_candidates.py`, `tests/test_collect.py`

**Interfaces:**
- Consumes: Task 3~6의 함수 전부, `load_config`
- Produces:
  - `resolve_gains(pool: dict[str, dict], baseline: dict[str, int] | None, interval_days: int | None, window: Window, client: GitHubClient, cap: int, trend_gains: dict[str, int]) -> tuple[dict[str, int], dict]` (레포별 증가량, 메타: `history_lookups`, `cap_hit`, `skipped_404`)
  - `classify_hint(paths: list[str], topics: list[str]) -> str | None` (`"skill" | "plugin" | "mcp-server" | None`)
  - `is_inflation_suspect(gain: int, forks: int, open_issues: int, cfg: InflationConfig) -> bool`
  - `build_candidates(...) -> list[dict]` (상위 `candidates_for_agent`개, 각 항목 키: `full_name, url, description, topics, stars, gain_7d, forks, open_issues, pushed_at, created_at, license, homepage, category_hint, inflation_suspect`)
  - CLI: `python -m scout.collect --config config.yml --data-dir _data --run-date YYYY-MM-DD --mode {dry-run,bootstrap,full}`
    - 모든 모드: 풀 수집 후 `_data/data/snapshots/<run-date>.json` 기록 (`{"date", "repos": {full_name: {"stars", "created_at"}}}`).
    - `bootstrap`: 스냅샷만 쓰고 종료.
    - 그 외: `_data/data/candidates.json` 기록 (`{"window": {...}, "tools": [...], "claude_code_releases": [...], "meta": {...}}`).

`resolve_gains` 규칙: 기준 스냅샷에 있으면 스냅샷 비교. 없고 `created_at >= window.start`면 현재 스타 수. 그 외는 현재 스타 내림차순으로 스타 히스토리 조회(상한까지), 404면 건너뜀. 상한을 넘은 레포는 `trend_gains`에 값이 있으면 그 값, 없으면 순위에서 제외.

- [ ] **Step 1: 실패하는 테스트 작성** (`test_candidates.py`)
  - `test_resolve_uses_snapshot_diff_when_in_baseline`
  - `test_resolve_new_repo_created_in_window_uses_total_stars`
  - `test_resolve_caps_history_lookups_and_falls_back_to_trendshift`: cap=1, 기준 없는 레포 3개 → 스타 많은 1개만 조회, 1개는 trend 값, 1개 제외, `meta["cap_hit"] is True`.
  - `test_resolve_skips_404_repo`: 스타 히스토리 404 → 결과에 없음, `meta["skipped_404"] == ["gone/repo"]`. (Review Focus 1)
  - `test_build_excludes_reported_and_ranks_by_gain`: `reported`에 있는 레포 제외, 증가량 내림차순, 최대 30개.
  - `test_classify_hint`: `["skills/x/SKILL.md"]` → `"skill"`, `[".claude-plugin/plugin.json"]` → `"plugin"`, 토픽 `mcp-server` → `"mcp-server"`, 그 외 None.
  - `test_inflation_suspect`: `(5000, 10, 0)` → True, `(5000, 400, 30)` → False, `(500, 0, 0)` → False.
  - `test_trendshift_entries_filtered_by_relevance_pattern`: 설명에 키워드 없는 Trendshift 항목은 풀에 들어오지 않는다.
- [ ] **Step 2: 실패하는 테스트 작성** (`test_collect.py`, 클라이언트·Trendshift를 가짜로 주입)
  - `test_bootstrap_writes_snapshot_only`
  - `test_full_writes_candidates_and_snapshot`: `candidates.json`에 `window`, `tools`, `claude_code_releases`, `meta.trendshift_ok` 존재.
  - `test_empty_week_writes_empty_lists`: 풀이 비고 릴리스 0건 → `tools == []`, `claude_code_releases == []`, 예외 없음. (Review Focus 5)
  - `test_releases_filtered_to_window`: `published_at`이 기간 밖인 릴리스 제외.
- [ ] **Step 3: 실패 확인** — Run: `pytest tests/test_candidates.py tests/test_collect.py -v` / Expected: FAIL
- [ ] **Step 4: 구현** — 보강은 검색 item 필드를 쓰고, Trendshift에서만 온 레포는 `client.repo()`로 채운다. 분류 힌트는 최종 후보에 대해서만 `tree_paths`를 호출한다.
- [ ] **Step 5: 통과 확인** — Run: `pytest -v` / Expected: 전체 통과
- [ ] **Step 6: 로컬 실측** — Run: `GITHUB_TOKEN=... python -m scout.collect --config config.yml --data-dir /tmp/scout-data --run-date 2026-11-09 --mode dry-run` / Expected: 후보 30개 이하, `meta`에 호출 수 기록, core 호출 1,000회 미만. 풀 크기와 호출 수를 PR 본문에 적는다.
- [ ] **Step 7: 커밋** — `git commit -m "feat(collect): 후보 조립과 수집 CLI 추가"`

### Task 8: data 브랜치와 수집 워크플로우 (부트스트랩 시작)

**Files:**
- Create: `.github/workflows/weekly.yml` (이 Task에서는 수집과 data 커밋까지만)

**Interfaces:**
- Consumes: `python -m scout.collect` (Task 7)
- Produces: `weekly.yml`의 job 구조 `collect → analyze → publish` 중 `collect` job과 `commit-data` 스텝. Task 11이 나머지를 채운다.

- [ ] **Step 1: `data` 고아 브랜치 생성** — 로컬에서 `git switch --orphan data && git rm -rf . && mkdir -p data/snapshots reports && echo "{}" > data/reported.json` 후 `.gitkeep` 추가, 커밋 `chore: data 브랜치 초기화`, `git push -u origin data`, `git switch main`. (사용자 승인을 받은 1회성 작업. 이후 사람은 `data`에 커밋하지 않는다.)
- [ ] **Step 2: `data` 브랜치 보호 규칙 추가** — `gh api -X POST repos/albertjihyun/agent-tool-scout/rulesets`로 이름 `protect-data`, 대상 `refs/heads/data`, 규칙 `non_fast_forward`, `deletion`.
- [ ] **Step 3: `weekly.yml` 작성**
  - 트리거: `schedule: cron "17 23 * * 0"`, `workflow_dispatch` 입력 `mode`(choice: `dry-run` 기본, `bootstrap`, `full`), `skip_analysis`(boolean, 기본 false).
  - 실제 모드: 스케줄이면 `full`, 아니면 입력값.
  - `permissions: contents: write, issues: write`. `concurrency: weekly` (동시 실행 방지).
  - 스텝: main 체크아웃 → `data` 브랜치를 `_data`에 체크아웃 → Python 설치 → `pip install .` → `run-date`는 실행 시점 UTC 날짜 → `python -m scout.collect ...`.
  - `commit-data` 스텝: 모드가 `bootstrap` 또는 `full`일 때만 `_data`에서 `github-actions[bot]`으로 커밋 `chore(data): <run-date> 스냅샷 저장` 후 push.
  - `dry-run`이면 `_data/data/`를 아티팩트로 업로드.
- [ ] **Step 4: PR 머지 후 dry-run 확인** — Run: `gh workflow run weekly.yml -f mode=dry-run` / Expected: 성공, 아티팩트에 `candidates.json`과 스냅샷, `data` 브랜치 변화 없음.
- [ ] **Step 5: 부트스트랩 실행** — Run: `gh workflow run weekly.yml -f mode=bootstrap` / Expected: `data` 브랜치에 `data/snapshots/<오늘>.json` 커밋 1개. 이 날짜가 첫 정식 리포트(7일 ± 3일 뒤 월요일)의 기준이 된다.
- [ ] **Step 6: 커밋** — `git commit -m "ci(workflow): 주간 수집 워크플로우와 data 브랜치 기록 추가"`

### Task 9: 리포트 마커와 검증

**Files:**
- Create: `scout/report.py`, `prompts/report-template.md`
- Test: `tests/test_report.py`

**Interfaces:**
- Produces:
  - 마커 상수: `SECTION_TOOLS = "<!-- section:tools -->"`, `SECTION_CLAUDE = "<!-- section:claude-code -->"`, `SECTION_APPENDIX = "<!-- section:appendix -->"`, 항목 마커 `<!-- repo:{full_name} -->`.
  - `@dataclass class ParsedReport: text: str; tools: list[str]; appendix: list[str]; problems: list[str]`
  - `parse_report(text: str, allowed: set[str], max_tools: int, max_appendix: int) -> ParsedReport | None` (섹션 마커 하나라도 없으면 None)

규칙: 항목은 다음 항목 마커나 다음 섹션 마커 직전까지의 블록이다. `allowed`에 없는 레포 블록은 삭제, 섹션 1이 `max_tools`를 넘으면 뒤쪽 블록 삭제, 별첨에서 섹션 1과 겹치는 레포 블록 삭제, 별첨이 `max_appendix`를 넘으면 뒤쪽 삭제. 삭제할 때마다 `problems`에 사유를 남긴다.

- [ ] **Step 1: 실패하는 테스트 작성**
  - `test_parse_valid_report`: 섹션 3개, 도구 2개, 별첨 1개 → `tools`, `appendix` 순서 그대로.
  - `test_missing_section_returns_none`
  - `test_drops_repo_not_in_candidates`: 후보에 없는 `fake/repo` 블록이 `text`에서 사라지고 `problems`에 기록. (Review Focus 4)
  - `test_truncates_tools_over_max`: 도구 6개 → 5개.
  - `test_removes_appendix_duplicate_of_tools`
- [ ] **Step 2: 실패 확인** — Run: `pytest tests/test_report.py -v` / Expected: FAIL
- [ ] **Step 3: `scout/report.py` 구현, `prompts/report-template.md` 작성** — 템플릿은 설계 4장의 항목 형식과 위 마커를 그대로 쓴 마크다운 골격. 섹션 제목 문구는 에이전트가 리포트 언어로 쓰고, 마커는 그대로 둔다고 템플릿 안에 주석으로 적는다.
- [ ] **Step 4: 통과 확인** — Expected: 5 passed
- [ ] **Step 5: 커밋** — `git commit -m "feat(publish): 리포트 마커 파싱과 검증 추가"`

### Task 10: 발행 CLI

**Files:**
- Create: `scout/publish.py`
- Test: `tests/test_publish.py`

**Interfaces:**
- Consumes: `parse_report` (Task 9), `GitHubClient.create_issue`, `ensure_label` (Task 3), `Window` (Task 4), `load_config`
- Produces:
  - `issue_title(fmt: str, run_date: date, failed: bool, failed_prefix: str) -> str` (`week_of_month = (run_date.day - 1) // 7 + 1`)
  - `render_fallback(candidates: dict, language: str) -> str` (후보 목록과 릴리스 제목만, 섹션 마커 포함. 문구는 `ko`, `en` 두 벌을 코드 안 사전에 두고 그 외 언어는 `en`)
  - `status_footer(meta: dict, language: str) -> str` (후보 수, Trendshift 정상 여부, 스타 히스토리 조회 수, 상한 도달 여부)
  - `update_reported(reported: dict, tools: list[str], appendix: list[str], week: str) -> dict` (같은 레포·같은 주는 한 번만)
  - CLI: `python -m scout.publish --config config.yml --data-dir _data --run-date YYYY-MM-DD --mode {dry-run,full} --draft reports/draft.md --repo owner/name`
    - 초안 검증 성공 → 정리된 본문 + 푸터, 실패 또는 초안 없음 → `render_fallback` + 푸터, 제목에 실패 접두어.
    - 결과를 `_data/reports/<iso_week_label>.md`에 덮어쓰기.
    - 분석 성공일 때만 `_data/data/reported.json` 갱신.
    - `full`이면 라벨 확인 후 이슈 생성. `mention_owner`가 true면 본문 끝에 `cc @{owner_login}`. `dry-run`이면 이슈 생성 안 함.

- [ ] **Step 1: 실패하는 테스트 작성**
  - `test_issue_title_korean_format`: `2026-11-09` → `"[주간 리포트] 2026년 11월 2주차"`, 실패면 `"[분석 실패] [주간 리포트] 2026년 11월 2주차"`.
  - `test_fallback_used_when_draft_missing`: 초안 없음 → 이슈 본문에 후보 이름들, `reported.json` 변화 없음.
  - `test_valid_draft_updates_reported_once_per_week`: 같은 run-date로 두 번 실행해도 `reported.json`의 해당 레포 항목은 1개. (Review Focus 2)
  - `test_rerun_overwrites_same_week_report_file`
  - `test_dry_run_does_not_create_issue`: 가짜 클라이언트 `create_issue` 호출 0회.
  - `test_empty_week_still_publishes`: 후보 0개·릴리스 0건 + 초안 없음 → 이슈 1개, 본문에 "해당 없음" 문구. (Review Focus 5)
- [ ] **Step 2: 실패 확인** — Run: `pytest tests/test_publish.py -v` / Expected: FAIL
- [ ] **Step 3: 구현**
- [ ] **Step 4: 통과 확인** — Run: `pytest -v` / Expected: 전체 통과
- [ ] **Step 5: 커밋** — `git commit -m "feat(publish): 리포트 검증·저장·이슈 생성 CLI 추가"`

### Task 11: 분석 단계와 워크플로우 연결

**Files:**
- Create: `prompts/weekly-report.md`, `.claude/settings.json`
- Modify: `.github/workflows/weekly.yml` (Task 8)

**Interfaces:**
- Consumes: `_data/data/candidates.json` (Task 7), `prompts/report-template.md` 마커 (Task 9), publish CLI (Task 10)

- [ ] **Step 1: `prompts/weekly-report.md` 작성** — 담을 지시:
  - 입력 파일 경로(`_data/data/candidates.json`, `profile.md`, `prompts/report-template.md`)와 리포트 언어(`config.yml`의 `report.language`)를 읽는다.
  - 섹션 1: `tools` 후보 중 범위(스킬, 플러그인, MCP 서버, 개발 도구)에 맞는 최대 5개. `category_hint`가 있으면 그 분류를 바꾸지 않는다. `inflation_suspect`가 true면 "부풀리기 의심"을 표시한다. 항목 형식은 설계 4장. 설치 명령어는 레포 README에서 확인한 것만 쓰고, 확인 못 하면 "README 참고"라고 쓴다.
  - 섹션 2: `claude_code_releases`에서 새 기능과 사용 방식 변화만. 없으면 "이번 주 주요 변경 없음".
  - 별첨: 섹션 1 범위 밖이지만 흐름으로 알아 둘 만한 0~3개, 한두 줄, 깊이 조사하지 않는다.
  - 후보 목록에 없는 레포를 쓰지 않는다. 마커를 지운다거나 바꾸지 않는다.
  - 결과는 `reports/draft.md`에만 쓴다.
- [ ] **Step 2: `.claude/settings.json` 작성** — `permissions.allow`: `Read`, `Write(reports/draft.md)`, `WebSearch`, `WebFetch`. `permissions.deny`: `Bash`, `Edit`.
- [ ] **Step 3: `weekly.yml`에 analyze, publish 스텝 추가**
  - analyze: `skip_analysis`가 false이고 모드가 `bootstrap`이 아닐 때. `anthropics/claude-code-action@v1`, `claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}`, prompt는 `prompts/weekly-report.md` 내용, `claude_args: --max-turns 40 --model sonnet`, `continue-on-error: true`. 실행 후 소요 시간과 결과(`steps.analyze.outcome`)를 job summary에 기록.
  - publish: 모드가 `bootstrap`이 아닐 때 항상 실행(`if: always()` 포함). `GITHUB_TOKEN` 전달.
  - commit-data: `full`일 때 publish 이후 실행하도록 순서 조정, 커밋 메시지 `chore(data): <iso_week_label> 리포트와 스냅샷 저장`.
  - `dry-run`이면 `_data/reports/`와 `reports/draft.md`도 아티팩트에 포함.
- [ ] **Step 4: PR 머지 후 skip-analysis dry-run** — Run: `gh workflow run weekly.yml -f mode=dry-run -f skip_analysis=true` / Expected: 아티팩트에 대체 리포트(`[분석 실패]` 제목 형식), 이슈 없음.
- [ ] **Step 5: 분석 포함 dry-run** — Run: `gh workflow run weekly.yml -f mode=dry-run` / Expected: 아티팩트 리포트에 섹션 3개, 도구 5개 이하, 마커 유지. 실행 전후 주간 사용량을 사용자에게 확인받아 `docs/research/risk-checks.md`에 기록. 5%를 크게 넘으면 `--max-turns`와 후보 수를 줄이는 안을 사용자와 정한다.
- [ ] **Step 6: 커밋** — `git commit -m "feat(analyze): 분석 프롬프트와 주간 워크플로우 연결 추가"`

### Task 12: 운영 문서와 첫 정식 실행

**Files:**
- Modify: `README.md`

- [ ] **Step 1: README에 "직접 돌려 보기" 섹션 추가** — `claude setup-token`과 secret 등록, 토큰 1년 만료와 갱신 방법, `config.yml`과 `profile.md` 수정, cron 시각 위치(`.github/workflows/weekly.yml`), 실행 모드 세 가지, `data` 브랜치 설명. README 상단 "현재 상태"를 "운영 중"으로 바꾼다.
- [ ] **Step 2: 커밋, PR** — `git commit -m "docs: 운영 방법과 실행 모드 안내 추가"`
- [ ] **Step 3: 첫 정식 실행 확인** — 부트스트랩(Task 8 Step 5) 후 첫 월요일 cron 실행을 기다리거나, 사용자 승인 후 `gh workflow run weekly.yml -f mode=full`. Expected: 이슈 1개와 메일 도착, `data` 브랜치에 리포트 사본·스냅샷·`reported.json` 커밋.
- [ ] **Step 4: 다음 주 확인** — 두 번째 정식 실행에서 첫 주에 소개한 레포가 다시 나오지 않는지, 증가량이 스냅샷 비교로 계산됐는지(`meta.history_lookups`가 첫 주보다 작은지) 확인하고 결과를 사용자에게 보고.
