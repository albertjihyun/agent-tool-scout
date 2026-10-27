# GitHub 전체 상위 레포 데이터 출처 조사 (2026-10-24)

> 이 조사 당시에는 "GitHub 전체 최상위"를 섹션 3으로 둘 계획이었다. 이후 섹션 3은 폐지하고 별첨으로 흡수했다(`docs/design.md` 4장). 출처 비교, Trendshift 파싱, 스타 히스토리 API 계산법은 섹션 1 수집에 그대로 쓰인다.

GitHub 전체에서 지난 7일 스타 증가량 상위 레포를 어디서 가져올지 실제 호출로 비교한 결과.

## 결론

- 순위 기준: `GET https://api.github.com/repos/{owner}/{repo}/stargazers/history` (헤더 `X-GitHub-Api-Version: 2026-03-10`). 공개 레포는 인증 없이 동작. 호출 1회 = core 한도 1회.
  - 응답은 주 단위 배열, 최신 주가 먼저. 각 주의 `days`는 일요일부터 토요일 순 일별 증가량. 이번 주 마지막 칸은 진행 중인 값.
  - 지난 7일 = 주 배열을 일 단위로 펼쳐서 완료된 마지막 7일을 합산. 생성 1주 미만 레포는 이전 주가 없으므로 0으로 처리.
- 후보 출처 1순위: Trendshift
  - `https://trendshift.io/weekly` (이번 ISO 주 진행분), `https://trendshift.io/weekly/{YYYY}/{WW}` (완료된 주), `https://trendshift.io/` (일간). 페이지당 25개.
  - 파싱: Next.js 서버 렌더링. `self.__next_f.push([1,"..."])` 문자열을 JSON 디코드해 이어 붙인 뒤 `"initialData":` 뒤의 배열을 읽는다.
  - 필드: `rank, score, full_name, repository_description, repository_stars, repository_stars_gained, repository_created_at, date`.
  - `rank`는 스타가 아닌 복합 점수 기준이므로 증가량으로 재정렬.
  - robots.txt는 이 페이지들을 허용(`/api/`, `/apikeys/`, 로그인만 금지). 공식 API 없음. 1인 운영 사이트라 중단·구조 변경 위험.
- 후보 출처 2순위(GitHub만으로, Trendshift 실패 시 단독 사용):
  - 검색 `created:>={오늘-30일}` 스타순 100개
  - 검색 `stars:>20000 pushed:>={오늘-7일}` 스타순 300개
  - (참고) `github.com/trending` 일간은 robots.txt 허용. `?since=weekly`/`monthly`는 robots.txt 금지이므로 쓰지 않는다.
  - 이 조합(약 400개)만으로 실제 상위 10개 중 9개를 잡았다.
- 파이프라인: 두 출처 합집합(약 450~560개) → 전부 star-history 조회 → 7일 증가량 정렬 → 상위 후보 → `GET /repos/{r}`로 설명 보강. Trendshift 파싱 결과가 15개 미만이면 기록만 남기고 GitHub 경로로 계속.
- 한도: Actions `GITHUB_TOKEN`은 시간당 1,000회. 후보 500개 수준이면 충분.

## 출처별 결과 (실제 상위 10 대비)

| 출처 | 항목 수 | 실제 상위 10 중 | 비고 |
|---|--:|--:|---|
| Trendshift 주간 | 25 | 10 | 증가량 수치가 star-history와 0~8% 차이 |
| GitHub Trending 주간 | 15 | 2 | robots.txt 금지, 증가량순 아님 |
| 검색 created 30일 | 100 | 5 | |
| 검색 stars>20k pushed 7일 | 300 | 4 | 오래된 레포의 급상승을 잡는 유일한 검색 |
| OSS Insight | 0 | - | 2026-03부터 이벤트 수집 붕괴, 결과 비어 있음. 사용 불가 |
| GH Archive / BigQuery | - | - | 같은 원인으로 이벤트량이 예전의 약 2.5%. 유료. 사용 불가 |

## 이번 주 실제 상위 (2026-10-17 ~ 10-23 UTC, 후보 1,071개 검증)

1. morluto/rea 80,400 (3주에 약 37만, 부풀리기 의심)
2. Gathin23/cleanyourtv 28,755
3. deepseek-ai/DeepInfer 21,357
4. Ebony-Vinyl/dsh-our-free-model 20,765
5. google-research/harness2 17,994
6. ai-job-search-dev/ai-job-search 15,483
7. cloudflare/security-audit-skill 12,374
8. debpalash/VoiceStudio 12,201
9. KnifeLemon/Filee 9,898
10. DietrichGebert/ponytail 9,898

11~15: stablyai/orca, peach-mitao/peach, deepseek-ai/deepseek-harness, mattpocock/skills, boykopovar/AnyPS5.

관찰: 상위권에 기술과 무관한 레포와 부풀리기 의심 레포가 섞인다. 의미 있는 항목은 대부분 AI·에이전트 분야다. GitHub 전체 섹션을 폐지하고 별첨으로 흡수한 근거.

## 확인하지 못한 것

- GitHub Actions(Azure IP)에서 Trendshift 접속이 막히는지.
- Trendshift `/api/`의 이용 조건.
- 후보 풀 밖에 더 크게 오른 레포가 없다는 증명(공개 이벤트 스트림이 없어 불가. 풀을 564→1,071개로 늘려도 상위 25는 그대로였음).
- 세 출처의 주 경계가 다름(Trendshift 월~일 ISO, star-history 일~토, GitHub Trending 미상). 하나로 통일해야 함.
