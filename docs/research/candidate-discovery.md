# 섹션 1 후보 찾기 조사 (2026-10-24, 중단된 중간 결과)

AI 코딩 도구 생태계 레포(스킬, 플러그인, MCP 서버, 에이전트 도구)를 GitHub 검색으로 얼마나 잡아낼 수 있는지 실험했다. 조사 도중 중단했고, 아래는 저장된 데이터에서 다시 계산한 결과다.

## 실험 방법

- 정답 집합: GitHub Trending(언어별 포함)과 이번 주 상위권에서 생태계 관련 레포 39개를 골라 기준으로 삼았다. 예: obra/superpowers, mattpocock/skills, cloudflare/security-audit-skill, anthropics/claude-plugins-official, github/github-mcp-server, anomalyco/opencode, wshobson/agents, expo/skills 등.
- 비교한 검색 전략 (모두 `created:>=2025-10-24`, 즉 최근 1년 생성 레포 대상):
  - topics: `topic:claude-code`, `agent-skills`, `mcp`, `mcp-server`, `ai-agents`, `codex`, `claude-skills`
  - K1: `claude OR skill OR skills OR mcp OR codex OR agentic in:name,description`
  - K2: `agent OR agents OR cursor OR copilot OR opencode OR harness in:name,description`
  - R1: `"claude code" OR "SKILL.md" OR "mcp server" OR "coding agent" OR "claude-code" in:readme`
  - 검색 결과가 1,000개 제한에 걸리면 스타 구간을 나눠 전부 열거했다.

## 결과 (스타 200개 이상 기준)

| 전략 | 후보 수 | 정답 39개 중 |
|---|--:|--:|
| topics만 | 3,226 | 27 |
| K1 (이름·설명 키워드) | 3,911 | 33 |
| R1 (README 키워드) | 6,043 | 34 |
| topics + K1 + K2 | 4,913 | 34 |
| 전부 합침 | 8,503 | 35 |

스타 1,000개 이상으로 올리면 후보는 절반 이하로 줄고 정답은 1~2개 더 놓친다.

## 관찰

- 토픽만으로는 부족하다. 많은 레포가 토픽을 달지 않는다(mattpocock/skills, anomalyco/opencode 등).
- 이름·설명 키워드(K1)가 가장 효율이 좋다.
- 검색으로 끝까지 못 잡은 레포: cloudflare/security-audit-skill, EpicGames/unreal-engine-skills-for-claude-code-plugin, HKUDS/CLI-Anything, SimoneAvogadro/android-reverse-engineering-skill. 이들은 Trendshift와 GitHub Trending에는 잡혔다. 즉 검색과 트렌딩 출처를 함께 써야 한다.
- 후보가 수천 개라서 전부 star-history를 조회하면 Actions 기본 토큰 한도(시간당 1,000회)를 넘는다. 조사 당시 5,005개 풀을 조회하다 1,250회 시점에서 중단했다.

## 후속 측정 (2026-10-25)

최근 7일 push 조건을 추가해 후보 수를 다시 쟀다(이름·설명 키워드, 최근 1년 생성, 최근 7일 push).

| 검색 | 스타 200+ | 500+ | 1,000+ |
|---|--:|--:|--:|
| K1 | 2,623 | 1,499 | 938 |
| K2 | 2,182 | 1,261 | 785 |
| K1, 생성일 제한 없음 | 3,940 | 2,471 | 1,685 |

스타 1,000개 이상으로 올려도 후보마다 star-history를 조회하면 한도를 넘고, 하한을 더 올리면 막 뜨기 시작한 작은 레포를 놓친다. 결정: 검색 결과에 현재 스타 수가 이미 들어 있으므로 주간 스냅샷 비교로 증가량을 구하고, star-history API는 새로 후보에 들어온 레포에만 쓴다(`docs/design.md` 6.1).

## 끝내지 못한 것

- 코드 검색(`filename:SKILL.md`, `.claude-plugin`)은 스크립트까지 만들었으나 결과 정리 전에 중단했다.
- 정확한 스타 하한과 push 조건은 구현하면서 조정한다.
