---
name: questions
description: 수학비서 문항 탐색 — "이차함수 문항 찾아줘", "비슷한 문제·숫자변형 찾아줘", "난이도 낮은/높은 참고 문항", "단원 자료 전부". 메타데이터 조회, "읽어줘" 이미지 판독, "보여줘" 보기 링크.
allowed-tools: mcp__plugin_mathsecr_api__curriculum_list, mcp__plugin_mathsecr_api__chapter_tree, mcp__plugin_mathsecr_api__cell_tag_search, mcp__plugin_mathsecr_api__cell_search, mcp__plugin_mathsecr_api__cell_count, mcp__plugin_mathsecr_api__cell_detail, mcp__plugin_mathsecr_api__similar_search_setting, mcp__plugin_mathsecr_api__similar_cells, mcp__plugin_mathsecr_api__question_quota, mcp__plugin_mathsecr_api__cell_image_link, mcp__plugin_mathsecr_api__cell_image, mcp__plugin_mathsecr_api__whoami
---

# 수학비서 문항 탐색

수학비서 문제은행과 회원 본인의 나만의 DB 에서 조건에 맞는 문항을 찾아 정리한다. 모든 도구는 읽기 전용이다.

아래 요청은 도구를 부르기 전에 해당 레시피 전문을 먼저 읽고 그 순서·상한을 따른다. 읽을 수 없으면(권한 거부·파일 없음) 멈추지 않는다. 답 첫머리에 「레시피를 읽지 못해 기본 규칙으로 진행(제한 모드)」이라고 밝히고, 이 문서의 규칙과 아래 「제한 모드」 상한으로 진행한다. 레시피에만 있는 단계는 하지 않고 「레시피 미적용 — 미확인」으로 적는다.

| 요청 | 레시피 |
|---|---|
| 기준 문항보다 난이도가 낮은·높은 참고 문항, 난이도 사다리 | [difficulty-ladder.md](references/difficulty-ladder.md) |
| 단원 자료를 한꺼번에(분포·ID 목록·이미지) | [chapter-material.md](references/chapter-material.md) |
| 수학비서에 없는 새 문제의 난이도 추정·적정성 | `analysis` 스킬의 difficulty-estimate 레시피 |

제한 모드(레시피를 읽지 못했을 때, 도구 호출 합계 8회·이미지 0장):
- 난이도 사다리: `cell_detail` 1 → `chapter_tree(chapterIds)` 1 → `cell_search`(`minDifficulty = maxDifficulty` 를 d-1, d+1 로, `limit: 5`; 비면 d±2 한 번씩) 최대 4. `relations` 의 PC·NC 와 `hasPermission` 이 true 가 아니거나 `isUnavailable`·`isProblemBankRemoved` 가 true 인 문항(문제은행 구독 필요·빠짐·권한 없음·사용 불가)은 후보에서 뺀다. 단계별 `totalCount` 를 `include` 로 적으면 `whoami` 1회를 더한다(아래 「문제은행 구독」, 합계 7).
- 단원 자료: `curriculum_list`·`chapter_tree` 합계 최대 2 → `whoami` 1 → 단원마다 `cell_count`(`breakdown: "difficulty"`, 기본 기준, 아래 4번의 분포 규칙) 1회, 최대 3단원, 한 번에 하나씩(합계 2+1+3 = 6) → 비용 안내. ID 목록·하위 단원 수는 받지 않는다. 센 단원이 둘 이상이면(별칭의 원본이 다른 단원 등) 단원마다 한 행으로 적고, 세 단원을 넘으면 나머지는 「세지 않음」으로 적는다. 도구 목록에 `cell_count` 가 없으면 첫 단원만 `cell_search` `limit: 1` 5회(전체·`minDifficulty` 1·3·5·7)로 세고 다른 단원은 「세지 않음」이다(합계 2+1+5 = 8).
- 새 문제 난이도 추정은 `analysis` 스킬의 제한 모드를 따른다.

## 순서

1. **교육과정 확정.** `curriculum_list` 로 교육과정 ID 를 찾는다. 사용자가 "고1", "공통수학1" 처럼 말하면 목록에서 맞는 것을 고르고, 둘 이상이면 어느 것인지 묻는다. 기본 교육과정(`is_default`)이 있으면 그것을 먼저 제안한다.
2. **단원 확정.** `chapter_tree` 를 `curriculumId` 와 `keyword`(예: "이차함수")로 부른다. 기본은 4단계이며, 가장 깊은 5단계 단원이 필요하면 부모(4단계) 단원 제목을 `keyword` 로 `depth: 5` 를 준다.
   - 요청이 상위 단원의 일부라면 해당 하위 단원만 고른다. 예를 들어 "이차함수" 요청에 "이차방정식과 이차함수" 전체를 넣지 말고 이차함수 하위 단원의 tree_id를 사용한다.
   - 응답의 `tree_id` 와 `chapter_id` 는 다를 수 있다(별칭 단원은 다르다). 문항 검색 필터 `chapterTreeIds` 에는 **tree_id** 를 넣는다. `isAlias: true` 인 별칭 단원은 문항이 원본 단원에 있으므로 `originTreeId` 를 넣는다.
   - `truncated` 가 true 면 일부만 온 것이다. `keyword` 를 좁히거나 `depth` 를 줄여 다시 부르고, 전체 트리라고 말하지 않는다.
   - 이미 아는 문항의 단원을 찾을 때는 그 문항의 `chapters[].id`(chapter_id)를 `chapter_tree(chapterIds: [...])` 로 넘긴다. 그 단원과 상위 경로, depth(기본 5단계)까지의 하위가 오고, depth 가 얕아도 그 단원은 온다(서버 v0.8.1 이상). 제목 대조가 필요 없다. `keyword` 와 함께 쓰지 않는다.
   - 같은 이름 단원이 둘 이상이면(예: 중3과 공통수학1) `nodePath`·`tree_id` 로 구분해 표로 보여 주고, 결과도 단원별로 나눠 센다. 하나만 고를 근거가 없으면 둘 다 검색하고 밝힌다.
3. **(선택) 태그.** 개념·유형으로 거르고 싶다고 하면 `cell_tag_search` 를 `keywords` 로 부른다. 필터 없이 부르면 속성 목록을 안내한다. 고른 태그 ID 는 `cell_search` 의 `adaptiveTagIds` 로 쓴다.
4. **검색.** `cell_search` 를 부른다. 필수는 `curriculumId` 다.
   - 난이도: `minDifficulty`·`maxDifficulty`. 정답 종류: `answerTypes`(`choice`·`answer`·`proof`·`boolean`).
   - 범위: 사용자가 나만의 DB 를 지정하면 `exclude`(나만의 DB 만), 수학비서·문제은행 문항을 말하거나 범위를 말하지 않고 단원을 탐색하면 `include` 로 검색한다. 문제은행만이면 `only`. 어떤 범위로 검색했는지 항상 밝힌다. `include` 는 구독 중일 때만 문제은행 문항을 더한다(아니면 `exclude` 와 같은 결과). `only` 는 구독이 없어도 오지만 잠긴 문항이고 `totalCount` 에 센다. 결과에 `isProblemBank: true` 가 없으면 범위를 '문제은행 포함(구독 중일 때만 반영)'이라고 적고 '문제은행에 없다'고 하지 않는다. 구독 상태를 읽었으면 「문제은행 포함(이 계정: 이용 중 아님 — 보유·공용만)」처럼 그 값을 함께 적는다.
   - 난이도 필터는 문항의 소문항 난이도 중 **최댓값**으로 걸린다. 소문항마다 회원 값, 없으면 수학비서 값을 쓴 최댓값이다. 난이도 없는 문항은 `minDifficulty` 를 주면 빠지고 `maxDifficulty` 만 주면 들어온다.
   - `include`·`only` 는 `myDbIds`·`sourceStartYear`·`sourceEndYear` 와 함께 쓸 수 없다. 도구가 거부하면 조건을 나눠 두 번 검색한다.
   - 더 필요하면 응답의 `nextCursor` 를 `cursor` 로 넘기고 **다른 필터는 그대로 둔다.**
   - `limit` 은 목적에 맞춘다: 개수만 → `1`(`totalCount`), 보여 줄 후보·미리보기 → `5`, 끝까지 세거나 ID 를 모두 모을 때(열거) → `20`. 열거는 첫 페이지부터 새로 시작하고(미리보기 검색을 이어 쓰지 않는다) 페이지를 넘길 때 `limit` 을 포함한 조건을 바꾸지 않는다. 시작 전에 필요한 페이지 수(전체 수 ÷ 20, 올림)를 알리고, 5페이지를 넘으면 먼저 묻는다. 이미 정한 범위(단원·난이도·관계)를 두고 사용자가 "전부"·"모두"를 말했으면 동의로 보고 25페이지까지 간다. 단원 자료처럼 범위를 아직 좁히지 않은 "전부"는 동의가 아니다(단원 자료 레시피를 따른다). 구독 중이고 '문제은행 포함'이 켜진 계정은 문제은행 문항까지 세므로 페이지 수가 다를 수 있다. `similar_cells` 도 같다.
   - `totalCount` 는 첫 페이지에만 온다. 난이도 분포는 한 페이지 결과로 내지 않고 `cell_count` 로 센다. `cell_count(breakdown: "difficulty")` 에 검색과 같은 `curriculumId`·`chapterTreeIds`·`problemBankScope`·필터를 넣고 `minDifficulty`·`maxDifficulty` 는 빼서 1회 부른다. 기본 기준 `minDifficulties` [1,3,5,7] 의 `ranges` 가 웹의 난이도 4단계(쉬움 1-2·보통 3-4·어려움 5-6·매우 어려움 7-9) 칸, `noDifficulty` 가 「난이도 없음」, `totalCount` 가 전체다. 한 난이도의 정확한 수가 필요할 때만 기준을 골라 한 번 더 부른다(오름차순 1~9, 4개까지; `[d, d+1]`(d 1~8, 9 는 `[9]`) 이면 `ranges` 첫 칸이 난이도 d, 1 을 넣어야 「난이도 없음」이 온다). `count` 가 null 인 칸은 「확인 실패(<reason>)」로 적고 채우거나 다른 칸에서 계산하지 않는다. `reason: "derived_zero"` 는 전체나 더 낮은 기준이 0이라 0으로 둔 값이다. `consistent` 가 false 면 그대로 알린다. `minDifficulty = maxDifficulty = d` 로 센 수를 더하지 않는다(같은문제 묶음이 두 칸에 들어갈 수 있다). `cell_count` 는 한 번에 하나씩 부르고(병렬 금지), "호출이 너무 잦습니다" 오류면 다시 부르지 않고 센 데까지만 알린다. `include` 로 센 수에는 이 계정 문제은행 상태(`whoami`)를 각주에 적는다(이용 중이 아니면 보유·공용 문항만). 도구 목록에 `cell_count` 가 없으면(예전 서버) `cell_search` `limit: 1` 로 전체 수와 `minDifficulty` 1·3·5·7(`maxDifficulty` 없이)의 `totalCount` 를 읽어 같은 칸을 차로 만든다(a~b = a 값 − b+1 값, 난이도 없음 = 전체 − 1 값).
5. **상세·유사.** 특정 문항은 `cell_detail`(`cellId`, `curriculumId`)로 단원·난이도·권한을 확인한다. 교육과정을 듣지 못한 문항 ID 는 기본 교육과정(`is_default`)으로 `cell_detail` 을 한 번 부른다. 권한·사용 가능 여부(`hasPermission`·`isUnavailable`·`isProblemBankRemoved`)는 교육과정과 상관없으므로 「사용 불가」·「권한 없음」·「문제은행 구독 필요」가 오거나 오류가 나면 다른 교육과정으로 다시 부르지 않는다. 열 수 없는 문항(사용 불가·권한 없음·문제은행 구독 필요)은 이미지 도구(`cell_image`·`cell_image_link`)를 부르지 않는다. 쓸 수 있는 문항인데 `chapters` 가 비었을 때만 사용자에게 교육과정을 묻는다. 비슷한 문항은 `similar_cells` 로 찾는다.
   - `relationType` 은 필수다. 요청에 맞춰 고른다: `PC` 같은문제, `NC` 숫자변형, `5depth` 유사문제, `4depth` 같은유형, `MT` 다른단원 유사문제.
   - 관계 목록은 서로 겹치지 않는다. `5depth` 유사문제에는 같은문제(PC)·숫자변형(NC) 문항이 들어 있지 않다.
   - "비슷한"·"유사"·"변형"·"숫자 바꾼" 요청은 관계 하나를 콕 집어 말하지 않는 한(예: "같은문제만") 같은문제(PC)·숫자변형(NC)·유사문제(5depth)를 모두 본다.
     1. **기준 문항.** `cell_detail` 로 기준 문항을 읽는다. 기준이 「문제은행 구독 필요」 문항(아래 「문제은행 구독」)이면 수학비서가 비슷한 문항 검색을 거절하므로 `similar_cells` 를 부르지 않고, 「문제은행 구독 필요」라고 알린 뒤 멈춘다. `relations` 는 「미리보기(보유·공용 문항, 문제은행 제외·설정 미적용, 종류별 ID 20개까지)」다. 전체 수라고 부르지 않는다. `relations` 에 없는 종류는 0개가 아니라 「미리보기 없음」이다(문제은행 문항만으로 된 관계일 수 있다).
     2. **호출.** `similar_cells` 를 PC·NC·5depth 에 한 번씩 부른다. 목록을 보여 줄 관계(5depth, 변형·숫자 바꾼 문제 요청이면 NC)는 `limit: 5`, 개수만 필요한 관계는 `limit: 1` 로 `totalCount` 만 읽는다. 첫 응답의 `appliedRelationType` 이 `TX` 면 나머지를 부르지 않는다. 한 관계 안의 문제은행 문항 수나 전체 ID 가 필요할 때만 그 관계를 `limit: 20` 으로 처음부터 열거한다(위 4번의 열거 규칙).
     3. **개수.** 관계별 개수(0 포함)는 `similar_cells.totalCount` 「설정 적용」으로 말한다. `relations[].count` 는 「미리보기(보유·공용)」 칸에 따로 적는다. 어느 수도 「전체」라고 부르지 않는다(목록은 「받은 n / 설정 적용 N」). 두 수를 전체·부분으로 부르지 않고, 한쪽에서 다른 쪽을 빼서 문제은행 수를 만들지 않는다. 문제은행 비중은 「받은 n개 중 isProblemBank m개」로만 적는다. 「문제은행 문항은 구독 중이고 유사 검색 설정의 '문제은행 포함'이 켜졌을 때만 이 수에 들어간다」를 각주에 한 번 적는다. MT 는 `similar_cells` 를 부르지 않았으면 「설정 적용 수 미확인」이다.
     4. **넓히기.** 5depth·NC·PC 의 `totalCount` 가 모두 0일 때만 `4depth`, `MT` 순으로 넓히고, 넓혔다고 밝힌다.
   - 결과에 문제은행 문항이 없어도 "문제은행에 관련 문항이 없다"고 말하지 않는다. 구독 상태는 `whoami` 값으로만 말하고, 문제은행 문항이 없는 원인(구독·유사 검색 설정·실제로 없음)을 하나로 단정하지 않는다. 사용자가 문제은행 문항을 기대했는데 하나도 없을 때만 `similar_search_setting` 을 1회 읽어 '문제은행 포함' 설정을 알린다.
   - 숫자변형·같은문제 문항의 난이도는 기준 문항에서 복사된 경우가 많다. 난이도 비교의 독립된 근거로 쓰지 않는다.
   - 결과 수는 회원의 유사 검색 설정(연도·난이도·DB 등)으로 좁혀진 값이다. 적거나 0이면 설정 때문일 수 있다고 밝힌다.
   - 회원의 검색 설정(`similar_search_setting` 의 `searchMethod`)이 `textSimilarity` 면 수학비서가 요청과 관계없이 텍스트 유사성(`TX`)으로 답한다. 응답의 `appliedRelationType` 이 `TX` 면 넓혀도 결과가 같으니 멈추고, 그렇게 검색됐다고 알린다.
   - 다음 페이지는 응답의 `appliedRelationType` 을 `relationType` 으로 쓰고 같은 필터와 `nextCursor` 를 넘긴다.
   - 태그 사전과 검색 필터의 근거만 제시하고 「문항별 부착 태그 확인 불가」를 적는다. 문항별 태그를 추측하지 않는다.
6. **한도.** "몇 문항까지 출제할 수 있어?" 는 `question_quota` 로 답한다. 분석 잔액이 아니다.

## 결과 정리

- **답 모양.** 맨 앞 1~2줄에 결론(바로 쓸 수·판정·추천)을 쓰고, 다음에 표, 마지막에 번호 각주를 둔다. 근거 라벨은 행마다 표의 근거 칸에 쓴다. 구독·유사 검색 설정·미리보기·태그처럼 되풀이되는 단서는 각주에 한 번씩만 쓰고 본문과 표마다 다시 쓰지 않는다. 에이전트 보고를 옮길 때도 이 모양을 지키고 다시 늘리지 않는다.
- 문항은 **ID·단원 경로·난이도·정답 유형·문제은행 여부**로 표를 만든다. 내용 확인이 필요하면 아래 이미지 경로를 쓴다.
- 모든 표와 집계에 근거를 붙인다: `「수학비서 데이터」 도구.필드 + ID`(예: 「수학비서 데이터」 `cell_search.totalCount`, tree_id 7110). 레시피 결과도 같다.
- **문제은행 구독.** 문제은행 문항은 회원의 구독 상태에 따라 열 수 있는지가 달라진다. 이 계정의 결과가 다른 계정에도 같다고 말하지 않는다. 구독 상태는 `whoami` 의 `problemBankSubscription.status` 로만 말한다. 답이 구독에 기대는 말을 할 때(`include`·`only` 로 센 수·`cell_count`, 잠긴 문항, `lockedProblemBankCount`, 유사 결과의 문제은행 비중)만, 이 대화에서 아직 읽지 않았으면 한 번 불러 「수학비서 데이터」 `whoami.problemBankSubscription` 으로 각주에 적는다(subscribed 「이용 중」 / not_subscribed 「이용 중 아님」 / unknown 「확인 불가」, 조회 시점 값). 문항 결과와 다르면 결과를 따르고 원인을 추정하지 않는다. 확인 불가거나 그 필드가 없으면(예전 서버) 「구독 상태에 따라 다름」 단서를 그대로 쓴다. 구독을 권하지 않고 확인·변경은 수학비서 웹으로 안내한다.
  - `isProblemBank: true`·`hasPermission: false`·`isProblemBankRemoved` 가 true 아님 = 「문제은행 구독 필요」. 후보·앵커·난이도 단계·대체 추천에 쓰지 않고 `cell_image`·`cell_image_link` 를 부르지 않는다. 단원·난이도는 「잠긴 문항의 저장값」으로만 적는다.
  - `isProblemBankRemoved: true` = 「문제은행에서 빠진 문항」. 구독과 상관없이 쓰지 않는다.
  - 그 밖의 `hasPermission: false` = 「권한 없음」, `isUnavailable: true` = 「사용 불가」.
  - 개수는 권한 필드를 읽은 결과만 「쓸 수 있음 N / 열 수 없음 M(사유별)」으로 나눈다. `similar_cells` 결과는 조회 시점에 열 수 있는 문항만 오므로 받은 n개를 모두 쓸 수 있음으로 적고(TX 결과도 세기만 할 때는 확인하지 않는다), 문제은행 비중은 「받은 n개 중 isProblemBank m개」로 적는다(구독 중이고 '문제은행 포함'이 켜졌을 때만 들어온다). 이 결과로 구독 여부를 추론하지 않는다(구독 상태는 `whoami` 값으로만 말한다). `similar_cells.totalCount` 는 「설정 적용」 수로만 쓴다.
  - 「잠긴 문항 포함 여부 미확인」은 `only` 검색처럼 잠긴 문제은행 문항이 섞일 수 있는 결과를 나눠 세지 못했을 때만 쓴다. `similar_cells` 와 `include` 검색 결과에는 붙이지 않는다.
- `sourceless: true` 인 문항은 출처에서 빠진 문항이다(출처가 삭제·해제됨). 출처 칸에 "출처 없음"으로 적는다. 출처 ID 0 도 같은 뜻이다. 출처 정보가 누락됐다거나 오류라고 말하지 않는다.
- 문제은행 문항(`isProblemBank: true`)은 원래 출처 목록이 비어 있다(`sources: []`, `sourceless` 없음). 출처 칸에 "문제은행"으로 적는다.
- `totalCount` 와 실제로 받은 개수(`returnedCount`)를 함께 말한다(`similar_cells` 는 「받은 n / 설정 적용 N」). `truncated`·`nestedTruncated` 가 true 면 일부라고 밝힌다.
- 후보가 많으면 상위 몇 개만 보이고 기준(난이도·단원 분포)을 요약한다.

## 메타데이터·이미지 판독·보기 링크

표의 각 행에 근거 라벨·실제 필드·ID를 적고 생략하지 않는다. 보기 링크의 포함 항목·거절·만료·조회 횟수도 `「수학비서 데이터」 cell_image_link.entries/denied/expiresAt/maxViews + cellId`로 표시한다. 이미지 사용량은 `cell_image.returned/budgetRemainingToday` 또는 사용자 제공값이라고 출처를 붙인다. 오류·권한 차이·잔액 감소의 내부 원인은 추정하지 않는다.

- 단원·난이도·배점·상태 질문은 메타데이터로 답한다. "보여줘"는 `cell_image_link`, "읽어/분석/비교"는 내용이 필요할 때 `cell_image`를 쓴다. 메타데이터 응답에 문항 본문은 없지만 허용된 이미지는 직접 판독할 수 있다.
- 보기 링크: `curriculumId`와 단일 `cellId` **또는** 중복 없는 `cellIds`(최대 10개), `parts: ["question"]` 또는 `["question", "explanation"]`을 보낸다. 한 이미지는 5분·5회, 여러 이미지는 10분·항목당 3회다. 실제 `url`, `expiresAt`, `maxViews`, `entries`를 전달한다. 링크를 직접 열거나 내려받지 않는다(WebFetch·브라우저 자동화 금지).
- 직접 판독: `cell_image(cellId, curriculumId, parts)`는 허용된 문항·지문·해설 이미지를 호출당 최대 4장 반환한다. parts 입력은 `question`(기본)·`explanation`만, `explanationIds`는 확인된 해설 ID만 쓴다(`cell_detail.explanations[].id`, 기본 해설은 `isMain`·선택 해설은 `isChosen`). 해설은 정답과 전체 풀이를 포함하므로 필요한 분석에만 쓰고, 요청 없이 정답을 노출하지 않는다. 문항·풀이·LaTeX 전사는 하지 않는다.
- 회원 공용 일일 이미지 예산은 **300장**. 요청 누적 기본 **4장**, 필요한 비교는 이유를 알리고 **8장 이내**, 8장 초과 전에는 사용자에게 묻는다. 위임 전 사용량을 차감하며 순차 호출한다. 호출 전 여유는 이번 호출이 돌려줄 수 있는 최대 장수로 계산한다(question 1장, 지문이 있으면 +1, 해설은 지정한 해설 ID 수, 기본 1장), `returned` 장수로 정산한다. `budgetRemainingToday`가 이번 호출 예상 장수보다 작으면 멈춘다. 이미지 호출은 병렬로 하지 않는다(동시 호출 중에는 다른 호출의 예약분이 잔액에서 잠시 빠져 보인다). 잔액은 가장 최근 응답의 `budgetRemainingToday`를 쓰고, 도구 오류로 잔액이 오지 않으면 직전 잔액에서 요청 장수를 뺀 값으로 이어 간다. 직전 잔액이 없으면 멈춘다. 실패한 부분은 차감되지 않으며 `denied`에 사유가 온다(해설은 `explanationId` 포함; `not_found` = 그 문항의 해설 ID가 아님, `no_explanation` = 해설이 없는 문항, `permission_denied` = 권한 없음). 응답에 `fallback: "cell_image_link"`가 있으면 읽지 못한 그 부분은 사용자에게 보기 링크를 제안한다.
- 두 이미지 도구 모두 `denied`에 거절된 문항·부분·사유가 온다. 이를 보고하고 자동 재시도하지 않는다. `cell_image`의 "이미지를 가져올 수 없습니다" 오류도 재시도하지 않고 `cell_image_link` 보기를 제안하며 수학비서 데이터만으로 계속한다.
- 모든 분석 근거를 `「수학비서 데이터」 도구.필드 + ID` 또는 `「이미지 판독(Claude 해석)」 cellId + part + 판독자`로 구분한다. 해석을 저장 정보라고 쓰지 않는다. `similar_cells` 결과는 열 수 있는 문항만 오므로 권한 확인용 `cell_detail` 을 부르지 않는다. 단 `appliedRelationType` 이 TX 인 결과는 최대 5분 캐시라 그사이 권한이 바뀐 문항이 남을 수 있으므로, 추천·채택·이미지 전에 그 문항을 `cell_detail` 로 확인한다(개수만 셀 때는 확인하지 않는다). 다른 값이 필요해 부른 `cell_detail` 에서 `hasPermission` 이 true 가 아니거나 `isUnavailable`·`isProblemBankRemoved` 가 true 면 그 문항을 빼고, 사유 라벨(「문제은행 구독 필요」·「문제은행에서 빠진 문항」·「권한 없음」·「사용 불가」)과 「유사 결과와 상세 조회 결과가 다름」을 적는다. 원인은 추정하지 않는다.
- 깊은 진단·비교·교체 추천은 `analysis` 스킬로 이어 간다. Code에서는 선택한 레시피 전문·메타데이터 출처·이미지 사용량/잔액을 `math-analyst`에 전달한다. 최종 분석자가 근거 이미지를 직접 가져온다. 앞선 판독은 판독자·출처를 보존하고 재판독하지 않았다고 표시한다.

## 하지 않는 것

- 문제지에 담기·교체·출제, 문항 구매는 할 수 없다. 요청하면 고른 문항 ID 목록을 정리해 주고 수학비서 웹에서 진행하라고 안내한다.
- 학생 정보(이름·답안·점수)는 다루지 않는다.
- 도구가 "다시 연결" 류 오류를 내면 `/mathsecr:status` 의 안내대로 재인증하게 한다.
- 웹에서만 되는 일(내려받기·구매·출제 등)을 브라우저 자동화(Chrome 등)로 대신하겠다고 제안하지 않는다. 수학비서 웹에서 직접 하도록 안내만 한다.
- 도구 정의에 있는 입력인데 "입력 형식" 오류가 나거나, "output schema" 불일치 오류가 나면 수학비서 서버가 업데이트돼 도구 정의가 바뀐 것이다. 같은 호출을 되풀이하지 말고 `/mathsecr:status` 안내대로 다시 연결하게 한다. 다시 연결한 뒤에도 같은 대화에서는 앞서 읽은 예전 정의를 따르기 쉬우니, 그 입력은 더 쓰지 않는다.

Claude Code 에서는 여러 번 검색해 후보를 추리는 일, 난이도 사다리, 단원 ID 수집을 `question-scout` 에이전트에 맡기면 대화가 짧게 유지된다. 레시피를 쓸 때는 레시피 전문과 이미 읽은 조건·ID, 이미지 사용 장수·잔액·상한, 사용자가 동의한 페이지 상한을 함께 넘긴다. 레시피를 읽지 못했으면 맡기지 않고 직접 제한 모드로 실행한다.
