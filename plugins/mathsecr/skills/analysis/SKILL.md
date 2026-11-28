---
name: analysis
description: 수학비서 데이터 분석 — "문제지 진단해줘", "두 문항 비교해줘", "대체 문항 추천해줘", "내가 만든 문제 난이도 몇이야". 저장 정보와 필요한 이미지 판독으로 근거를 정리한다.
allowed-tools: mcp__plugin_mathsecr_api__curriculum_list, mcp__plugin_mathsecr_api__chapter_tree, mcp__plugin_mathsecr_api__cell_tag_search, mcp__plugin_mathsecr_api__cell_search, mcp__plugin_mathsecr_api__cell_count, mcp__plugin_mathsecr_api__cell_detail, mcp__plugin_mathsecr_api__similar_search_setting, mcp__plugin_mathsecr_api__similar_cells, mcp__plugin_mathsecr_api__mypaper_list, mcp__plugin_mathsecr_api__mypaper_detail, mcp__plugin_mathsecr_api__mypaper_cells, mcp__plugin_mathsecr_api__mypaper_scores, mcp__plugin_mathsecr_api__cell_image, mcp__plugin_mathsecr_api__cell_image_link, mcp__plugin_mathsecr_api__whoami
---

# 수학비서 근거 기반 분석

회원 본인의 자료를 읽어 분석한다. 저장된 분석지 조회는 `myanalysis`를 쓴다. 아래에서 필요한 레시피 하나만 읽는다. 본문 도구 약칭의 실제 이름은 `mcp__plugin_mathsecr_api__<도구명>`이다.

**도구 호출 전에 선택한 레시피 전문을 먼저 읽는다.** 읽을 수 없으면(권한 거부·파일 없음) 멈추지 않는다. 답 첫머리에 「레시피를 읽지 못해 기본 규칙으로 진행(제한 모드)」이라고 밝히고, 이 문서의 규칙과 아래 「제한 모드」 상한으로 진행한다. 레시피에만 있는 단계는 하지 않고 「레시피 미적용 — 미확인」으로 적는다. `paperId`가 있으면 `mypaper_detail` 응답을 기다리고, 완료 문제지의 `mypaper_cells`에서 저장 교육과정을 확인한 뒤에만 문항 상세·단원·검색을 시작한다. 이 단계들을 병렬 호출하지 않는다. 편집 중이거나 저장 교육과정 미상이면 종속 조회를 멈춘다.

완료 문제지인데 `mypaper_cells` 에 문항이 오지 않고 `lockedProblemBankCount` 가 1 이상이거나 그 필드가 없거나 null 이면, 문제은행 문항이 잠겨(또는 구독을 확인하지 못해) 문항 목록이 오지 않은 것이다 — 빈 문제지가 아니다. 이때 응답의 `curriculumId` 는 요청값을 되돌려 준 것이라 저장 교육과정으로 보지 않는다(저장 교육과정 미상). 문항 수는 `mypaper_detail` 로 적고 0문항 진단을 내지 않는다. 문항에 기대는 조회(문항 상세·단원·검색·유사·이미지)는 멈추고, 대체 후보는 모두 「기존 문항 중복 미확인」이다. 확인은 수학비서 웹에서 하도록 안내한다(구독을 권하지 않는다).

| 요청 | 레시피 |
|---|---|
| 문제지 진단 | [paper-diagnosis.md](references/paper-diagnosis.md) |
| 두 문항 비교 | [question-compare.md](references/question-compare.md) |
| 대체 문항 추천 | [replacement-recommend.md](references/replacement-recommend.md) |
| 새로 만든 문제(수학비서에 없는 문제)의 난이도 추정·목표 난이도 적정성 | [difficulty-estimate.md](references/difficulty-estimate.md) |

수학비서 문항 ID 를 기준으로 쉬운·어려운 참고 문항을 찾는 요청은 `questions` 스킬의 난이도 사다리 레시피다.

## 제한 모드 (레시피를 읽지 못했을 때)

도구 호출은 합계 15회 이내. 이미지는 새 문제 난이도 추정만 최대 2장(question), 나머지는 0장.

| 요청 | 순서와 상한 |
|---|---|
| 문제지 진단 | `mypaper_detail` 1 → 출제 완료일 때만 `mypaper_cells`(limit 20) 최대 3페이지(문항 없이 잠겼으면 위 규칙대로 문항 단계를 멈춤) → `mypaper_scores` 1. 유사 관계 미검사, 「조회된 일부 기준」 |
| 두 문항 비교 | `cell_detail` 2 → `similar_cells`(NC·5depth, limit 5) 최대 2. 기준은 「문제은행 구독 필요」가 아닌 문항으로 하고, 둘 다 그렇다면 `similar_cells` 를 부르지 않는다 |
| 대체 문항 추천 | paperId 면 `mypaper_detail` 먼저(편집 중이면 멈춤) → 완료면 `mypaper_cells`(limit 20) 1페이지로 저장 교육과정 확인(문항 없이 잠겼으면 위 규칙대로 멈춤) → `cell_detail` 1 → `cell_search`(같은 단원·난이도, limit 10) 1페이지 → 후보 `cell_detail` 최대 3. 모든 후보 「기존 문항 중복 미확인」 |
| 새 문제 난이도 추정 | 먼저 문제를 보고 처음 추정 d0 을 정한다. `curriculum_list`·`chapter_tree` 합계 최대 2(`chapter_tree` 는 4단계 제목이나 개념어 `keyword` 와 `depth: 5` 로 불러 5단계 단원 하나를 고른다. 분포·앵커는 그 5단계 `tree_id` 로 센다) → `whoami` 1 → `cell_count`(`breakdown: "difficulty"`, `problemBankScope: "include"`, `minDifficulties` = d0-1·d0·d0+1·d0+2 중 1~9 안의 값) 1 → 앵커 `cell_search`(limit 5) 최대 3 → 앵커 `cell_detail` 최대 2 → 이미지 최대 2. 합계 2+1+1+3+2+2 = 11(`cell_count` 가 없으면 `cell_search` limit 1 로 전체와 같은 기준 누적 수 최대 5회, 합계 15). 아래 「새 문제 난이도 추정(제한 모드)」을 따른다 |

새 문제 난이도 추정(제한 모드):
- 분포와 앵커 검색은 모두 `problemBankScope: "include"` 다. 분포 칸은 `ranges` 그대로(d0-1·d0·d0+1·(d0+2)~9) 적고, 기준이 1 로 시작하지 않으면 전체 − 첫 기준 `atLeast` 를 「1~(d0-2)·난이도 없음」 한 칸으로 적는다(기준에 1 이 들면 `noDifficulty` 를 「난이도 없음」으로 따로 적는다). null 칸은 「확인 실패(<reason>)」다.
- 앵커는 d0-1·d0·d0+1 에서 `cell_search`(`minDifficulty = maxDifficulty`, limit 5, `excludeSubQuestions: true`, `excludePassageQuestions: true`) 로 찾아 난이도마다 `hasPermission: true`·`isUnavailable: false` 인 문항 하나씩, 3개를 쓴다. `cell_detail` 2회로 두 앵커의 `relations` 에 다른 앵커가 있는지 보고, 못 보면 「앵커 간 같은 묶음 여부 미확인」. 판독하지 못한 앵커는 저장 난이도만 쓰고 「판독 안 함」.
- 결론은 점 추정(1~9 정수)과 범위(기본 ±1, 1~9 안으로 자름). 앵커가 3개보다 적으면 ±2 로 넓히고 이유를 적는다.
- 목표를 주면 판정은 네 가지 중 하나를 글자 그대로 쓴다(「Claude 제안」): 점 추정 = 목표 「적정」 / 목표가 범위 안이지만 점 추정과 다르면 「경계(조정 권장)」(점 < 목표 "목표보다 조금 쉬움 — 한 단계 어렵게", 점 > 목표 "목표보다 조금 어려움 — 한 단계 쉽게") / 범위 전체가 목표보다 높으면 「목표보다 어려움」 / 낮으면 「목표보다 쉬움」. 「적정」이 아니면 같은 방향으로만 바꿀 점을 한두 가지 제안한다.
- 결과는 결론 줄 → 단원 분포 표(「수학비서 데이터」 cell_count.ranges·totalCount, tree_id, include) → 앵커 표 → 각주(include 로 센 수 · 이 계정 문제은행 상태(whoami, 조회 시점) · 앵커 묶음 확인 여부 · 정확도 미검증 · 「제한 모드라 레시피 모드보다 덜 정확할 수 있음」 · 이미지 장수/잔액) 순이다.

## 실행과 근거

- **답 모양.** 맨 앞 1~2줄에 결론(바로 쓸 수·판정·추천)을 쓰고, 다음에 표, 마지막에 번호 각주를 둔다. 근거 라벨은 행마다 표의 근거 칸에 쓴다. 구독·유사 검색 설정·미리보기·태그처럼 되풀이되는 단서는 각주에 한 번씩만 쓰고 본문과 표마다 다시 쓰지 않는다. 에이전트 보고를 옮길 때도 이 모양을 지키고 다시 늘리지 않는다.
- Desktop Chat에서는 스킬이 레시피를 직접 실행한다. Claude Code에서는 `math-analyst`에 선택한 **레시피 전문**, 요청·ID·교육과정·이미 읽은 필드와 출처·잘림 상태·이미지 사용 장수와 잔액·상한을 전달한다. 에이전트가 없으면 직접 실행한다. 레시피를 읽지 못했으면 위임하지 않고 직접 제한 모드로 실행한다. 에이전트에 파일 읽기 도구가 있다고 가정하지 않는다. 에이전트 보고를 사용자에게 옮길 때 레시피 결과 틀의 칸(판정 라벨·분포 표·각주)을 빼거나 바꾸지 않는다.
- 모든 사실·집계에 `「수학비서 데이터」 도구.필드, paperId/cellId/단원 ID`를 붙인다. 집계는 분모·범위·계산 방식을 적고, 권장·평가에는 **Claude 제안**이라고 명시하여 저장된 평가로 오인시키지 않는다. 새 문제의 난이도 추정은 「Claude 추정 — 수학비서 난이도 아님」으로 적는다. 이미지 판단은 `「이미지 판독(Claude 해석)」 cellId, part, 판독 주체`로 구분한다. 판독하지 않은 내용은 미확인이다.
- 태그 사전과 `cell_search.adaptiveTagIds` 검색 조건만 근거로 쓴다. **「문항별 부착 태그 확인 불가」**를 표시한다. `similar_cells` 결과는 열 수 있는 문항만 오므로 권한 확인용 `cell_detail` 을 부르지 않는다(대체 추천 레시피의 추천 직전 확인은 예외). 단 `appliedRelationType` 이 TX 인 결과는 최대 5분 캐시라 그사이 권한이 바뀐 문항이 남을 수 있으므로, 추천·채택·이미지 전에 그 문항을 `cell_detail` 로 확인한다(개수만 셀 때는 확인하지 않는다). 다른 값이 필요해 부른 `cell_detail` 에서 `hasPermission` 이 true 가 아니거나 `isUnavailable`·`isProblemBankRemoved` 가 true 면 그 문항을 빼고, 사유 라벨(「문제은행 구독 필요」·「문제은행에서 빠진 문항」·「권한 없음」·「사용 불가」)과 「유사 결과와 상세 조회 결과가 다름」을 적는다. 원인은 추정하지 않는다.
- 숫자변형(NC)·같은문제(PC) 문항의 난이도는 기준에서 복사된 경우가 많아 독립된 근거가 아니다. `similar_cells.totalCount` 는 「설정 적용」 수이며 「전체」라고 부르지 않는다(목록은 「받은 n / 설정 적용 N」).
- 문제은행 문항의 빈 `sources`는 "문제은행"으로 적는다. `sourceless: true` 또는 출처 ID 0만 "출처 없음"이다. 누락·null 필드는 추정하지 않는다.
- 난이도 분포는 `cell_count(breakdown: "difficulty")` 에 검색과 같은 `curriculumId`·`chapterTreeIds`·`problemBankScope`·필터를 넣고 `minDifficulty`·`maxDifficulty` 는 빼서 1회 부른다. 기본 기준 `minDifficulties` [1,3,5,7] 의 `ranges` 가 웹의 난이도 4단계(쉬움 1-2·보통 3-4·어려움 5-6·매우 어려움 7-9) 칸, `noDifficulty` 가 「난이도 없음」, `totalCount` 가 전체다. 한 난이도의 정확한 수가 필요할 때만 기준을 골라 한 번 더 부른다(오름차순 1~9, 4개까지; `[d, d+1]`(d 1~8, 9 는 `[9]`) 이면 `ranges` 첫 칸이 난이도 d, 1 을 넣어야 「난이도 없음」이 온다). `count` 가 null 인 칸은 「확인 실패(<reason>)」로 적고 채우거나 다른 칸에서 계산하지 않는다. `reason: "derived_zero"` 는 전체나 더 낮은 기준이 0이라 0으로 둔 값이다. `consistent` 가 false 면 그대로 알린다. `minDifficulty = maxDifficulty = d` 로 센 수를 더하지 않는다(같은문제 묶음이 두 칸에 들어갈 수 있다). `cell_count` 는 한 번에 하나씩 부르고(병렬 금지), "호출이 너무 잦습니다" 오류면 다시 부르지 않고 센 데까지만 알린다. `include` 로 센 수에는 이 계정 문제은행 상태(`whoami`)를 각주에 적는다(이용 중이 아니면 보유·공용 문항만). 도구 목록에 `cell_count` 가 없으면(예전 서버) `cell_search` `limit: 1` 로 전체 수와 `minDifficulty` 1·3·5·7(`maxDifficulty` 없이)의 `totalCount` 를 읽어 같은 칸을 차로 만든다(a~b = a 값 − b+1 값, 난이도 없음 = 전체 − 1 값).
- **문제은행 구독.** 문제은행 문항은 회원의 구독 상태에 따라 열 수 있는지가 달라진다. 이 계정의 결과가 다른 계정에도 같다고 말하지 않는다. 구독 상태는 `whoami` 의 `problemBankSubscription.status` 로만 말한다. 답이 구독에 기대는 말을 할 때(`include`·`only` 로 센 수·`cell_count`, 잠긴 문항, `lockedProblemBankCount`, 유사 결과의 문제은행 비중)만, 이 대화에서 아직 읽지 않았으면 한 번 불러 「수학비서 데이터」 `whoami.problemBankSubscription` 으로 각주에 적는다(subscribed 「이용 중」 / not_subscribed 「이용 중 아님」 / unknown 「확인 불가」, 조회 시점 값). 문항 결과와 다르면 결과를 따르고 원인을 추정하지 않는다. 확인 불가거나 그 필드가 없으면(예전 서버) 「구독 상태에 따라 다름」 단서를 그대로 쓴다. 구독을 권하지 않고 확인·변경은 수학비서 웹으로 안내한다.
  - `isProblemBank: true`·`hasPermission: false`·`isProblemBankRemoved` 가 true 아님 = 「문제은행 구독 필요」. 후보·앵커·난이도 단계·대체 추천에 쓰지 않고 `cell_image`·`cell_image_link` 를 부르지 않는다. 단원·난이도는 「잠긴 문항의 저장값」으로만 적는다.
  - 기준 문항이 「문제은행 구독 필요」 문항이면 수학비서가 비슷한 문항 검색을 거절하므로 그 문항으로 `similar_cells` 를 부르지 않고 「문제은행 구독 필요」라고 적는다.
  - `isProblemBankRemoved: true` = 「문제은행에서 빠진 문항」. 구독과 상관없이 쓰지 않는다.
  - 그 밖의 `hasPermission: false` = 「권한 없음」, `isUnavailable: true` = 「사용 불가」.
  - 개수는 권한 필드를 읽은 결과만 「쓸 수 있음 N / 열 수 없음 M(사유별)」으로 나눈다. `similar_cells` 결과는 조회 시점에 열 수 있는 문항만 오므로 받은 n개를 모두 쓸 수 있음으로 적고(TX 결과도 세기만 할 때는 확인하지 않는다), 문제은행 비중은 「받은 n개 중 isProblemBank m개」로 적는다(구독 중이고 '문제은행 포함'이 켜졌을 때만 들어온다). 이 결과로 구독 여부를 추론하지 않는다(구독 상태는 `whoami` 값으로만 말한다). `similar_cells.totalCount` 는 「설정 적용」 수로만 쓴다.
  - 「잠긴 문항 포함 여부 미확인」은 `only` 검색처럼 잠긴 문제은행 문항이 섞일 수 있는 결과를 나눠 세지 못했을 때만 쓴다. `similar_cells` 와 `include` 검색 결과에는 붙이지 않는다.

## 이미지 공통 규칙

표의 각 행에 근거 라벨·실제 필드·ID를 적고 생략하지 않는다. 보기 링크의 포함 항목·거절·만료·조회 횟수도 `「수학비서 데이터」 cell_image_link.entries/denied/expiresAt/maxViews + cellId`로 표시한다. 이미지 사용량은 `cell_image.returned/budgetRemainingToday` 또는 사용자 제공값이라고 출처를 붙인다. 오류·권한 차이·잔액 감소의 내부 원인은 추정하지 않는다.

- 메타데이터 질문은 저장 필드로 답한다. "보여줘"는 `cell_image_link`, "읽어/분석/비교"는 내용 확인이 꼭 필요할 때 `cell_image`다. 분석을 뒷받침할 소수 문항만 판독한다.
- 회원 공용 하루 300장. 요청 전체 기본 상한 **4장**, 꼭 필요한 비교는 이유를 밝히고 **8장 이내**로 확대할 수 있다. **8장 초과는 사용자에게 묻기 전 실행하지 않는다.** 위임 전 사용량도 차감한다. 호출 전 여유는 이번 호출이 돌려줄 수 있는 최대 장수로 계산한다(question 1장, 지문이 있으면 +1, 해설은 지정한 해설 ID 수, 기본 1장). 부족하면 중단하거나 위 한도 안에서 확대한다. 반환 `returned` 장수(지문·해설 포함)를 누적하며 **순차 호출**만 한다. 매 응답의 `budgetRemainingToday`를 읽는다. `budgetRemainingToday`가 이번 호출 예상 장수보다 작으면 멈춘다. 이미지 호출은 병렬로 하지 않는다(동시 호출 중에는 다른 호출의 예약분이 잔액에서 잠시 빠져 보인다). 잔액은 가장 최근 응답의 `budgetRemainingToday`를 쓰고, 도구 오류로 잔액이 오지 않으면 직전 잔액에서 요청 장수를 뺀 값으로 이어 간다. 직전 잔액이 없으면 멈춘다. 실패한 부분은 차감되지 않으며 `denied`에 사유가 온다(해설은 `explanationId` 포함; `not_found` = 그 문항의 해설 ID가 아님, `no_explanation` = 해설이 없는 문항, `permission_denied` = 권한 없음). 응답에 `fallback: "cell_image_link"`가 있으면 읽지 못한 그 부분은 사용자에게 보기 링크를 제안한다.
- `denied`의 문항·부분·사유를 알리고 자동 재시도하지 않는다. "이미지를 가져올 수 없습니다" 같은 오류도 재시도하지 않고 `cell_image_link`로 사용자가 볼 수 있다고 안내하며 **「수학비서 데이터」만으로 분석 계속**이라고 표시한다. 링크는 직접 열거나 다운로드하지 않는다.
- 최종 분석 주체(Code의 `math-analyst`, Chat의 스킬)가 근거로 쓸 이미지를 직접 가져온다. 앞선 판독을 전달받으면 판독자·출처를 보존하고 "전달받은 판독, 재판독하지 않음"이라 쓴다. 예산 부족이면 이를 새 판독으로 주장하지 않는다.
- `parts`는 `question`(기본)·`explanation`. 해설은 정답과 전체 풀이를 포함하므로 분석에 필요한 경우만 읽고, 사용자가 요청하지 않으면 정답 내용을 노출하지 않는다. 문항·정답·풀이의 전사와 LaTeX 변환은 하지 않는다.

읽기 전용이다. 문항 교체·출제·구매·파일 생성은 수학비서 웹에서 사용자가 한다. 학생 이름·답안·점수·명단은 요청하거나 다루지 않는다. 브라우저 자동화·WebFetch로 링크나 웹 작업을 대신하지 않는다. 인증 오류는 `/mathsecr:status` 재연결 안내를 따른다. 입력 형식/output schema 오류는 반복하지 않고 재연결을 안내하며 같은 대화에서 예전 정의의 입력을 더 쓰지 않는다.
