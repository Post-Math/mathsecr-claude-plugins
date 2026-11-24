---
name: questions
description: 수학비서 문항 탐색 — 교육과정·단원·태그로 문항을 찾고 유사 문항을 본다. "이차함수 문항 찾아줘", "이 단원 중상 난이도 문항", "이 문항과 비슷한 문제", "어떤 태그가 있어?", "유사문제 검색 설정 보여줘" 같은 요청에 쓴다. 문항 메타데이터만 다루며 문제지에 담거나 바꾸지 않는다.
allowed-tools: mcp__plugin_mathsecr_api__curriculum_list, mcp__plugin_mathsecr_api__chapter_tree, mcp__plugin_mathsecr_api__cell_tag_search, mcp__plugin_mathsecr_api__cell_search, mcp__plugin_mathsecr_api__cell_detail, mcp__plugin_mathsecr_api__similar_search_setting, mcp__plugin_mathsecr_api__similar_cells, mcp__plugin_mathsecr_api__question_quota, mcp__plugin_mathsecr_api__cell_image_link
---

# 수학비서 문항 탐색

수학비서 문제은행과 회원 본인의 나만의 DB 에서 조건에 맞는 문항을 찾아 정리한다. 모든 도구는 읽기 전용이다.

## 순서

1. **교육과정 확정.** `curriculum_list` 로 교육과정 ID 를 찾는다. 사용자가 "고1", "공통수학1" 처럼 말하면 목록에서 맞는 것을 고르고, 둘 이상이면 어느 것인지 묻는다. 기본 교육과정(`is_default`)이 있으면 그것을 먼저 제안한다.
2. **단원 확정.** `chapter_tree` 를 `curriculumId` 와 `keyword`(예: "이차함수")로 부른다.
   - 응답의 `tree_id` 와 `chapter_id` 는 다른 값이다. 문항 검색 필터 `chapterTreeIds` 에는 **tree_id** 를 넣는다.
   - `truncated` 가 true 면 일부만 온 것이다. `keyword` 를 좁히거나 `depth` 를 줄여 다시 부르고, 전체 트리라고 말하지 않는다.
3. **(선택) 태그.** 개념·유형으로 거르고 싶다고 하면 `cell_tag_search` 를 `keywords` 로 부른다. 필터 없이 부르면 속성 목록을 안내한다. 고른 태그 ID 는 `cell_search` 의 `adaptiveTagIds` 로 쓴다.
4. **검색.** `cell_search` 를 부른다. 필수는 `curriculumId` 다.
   - 난이도: `minDifficulty`·`maxDifficulty`. 정답 종류: `answerTypes`(`choice`·`answer`·`proof`·`boolean`).
   - 범위: `problemBankScope` 기본값은 `exclude`(나만의 DB 만)다. 문제은행을 포함하려면 `include`, 문제은행만이면 `only`.
   - `include`·`only` 는 `myDbIds`·`sourceStartYear`·`sourceEndYear` 와 함께 쓸 수 없다. 도구가 거부하면 조건을 나눠 두 번 검색한다.
   - 더 필요하면 응답의 `nextCursor` 를 `cursor` 로 넘기고 **다른 필터는 그대로 둔다.**
5. **상세·유사.** 특정 문항은 `cell_detail`(`cellId`, `curriculumId`)로 단원·난이도·권한을 확인한다. 비슷한 문항은 `similar_cells` 로 찾는다.
   - `relationType` 은 필수다. 요청에 맞춰 고른다: `PC` 같은문제, `NC` 숫자변형, `5depth` 유사문제, `4depth` 같은유형, `MT` 다른단원 유사문제.
   - 사용자가 정하지 않았으면 `5depth` 로 시작해 결과가 없을 때만 `4depth`, `MT` 순으로 넓히고, 넓혔다고 밝힌다.
   - 회원의 검색 설정(`similar_search_setting` 의 `searchMethod`)이 `textSimilarity` 면 수학비서가 요청과 관계없이 텍스트 유사성(`TX`)으로 답한다. 응답의 `appliedRelationType` 이 `TX` 면 넓혀도 결과가 같으니 멈추고, 그렇게 검색됐다고 알린다.
   - 다음 페이지는 응답의 `appliedRelationType` 을 `relationType` 으로 쓰고 같은 필터와 `nextCursor` 를 넘긴다.
6. **한도.** "몇 문항까지 출제할 수 있어?" 는 `question_quota` 로 답한다. 분석 잔액이 아니다.

## 결과 정리

- 문항은 **ID·단원 경로·난이도·정답 유형·문제은행 여부**로 표를 만든다. 문항 본문·정답·해설은 도구가 주지 않는다. 지어내지 말고 "문항 내용은 수학비서 웹에서 확인해 주세요" 라고 안내한다.
- 문항 이미지를 보여 달라고 하면 `cell_image_link`(`cellId`, `curriculumId`)로 **보기 링크**를 받아 그대로 건넨다. 링크는 5분 동안 5번까지 열린다(`expiresAt`, `maxViews`). 링크를 직접 열거나 내려받지 않는다(WebFetch·브라우저 자동화 금지). 이미지는 사용자 브라우저에서만 보인다. 이용 권한이 없거나 이용할 수 없는 문항은 링크가 나오지 않는다. 정답·해설·본문 텍스트는 여전히 도구가 주지 않는다.
- `hasPermission` 이 false 거나 `isUnavailable`·`isProblemBankRemoved` 가 true 인 문항은 표에 표시해 구분한다.
- `sourceless: true` 인 문항은 출처에서 빠진 문항이다(출처가 삭제·해제됨). 출처 칸에 "출처 없음"으로 적는다. 출처 ID 0 도 같은 뜻이다. 출처 정보가 누락됐다거나 오류라고 말하지 않는다.
- 문제은행 문항(`isProblemBank: true`)은 원래 출처 목록이 비어 있다(`sources: []`, `sourceless` 없음). 출처 칸에 "문제은행"으로 적는다.
- `totalCount` 와 실제로 받은 개수(`returnedCount`)를 함께 말한다. `truncated`·`nestedTruncated` 가 true 면 일부라고 밝힌다.
- 후보가 많으면 상위 몇 개만 보이고 기준(난이도·단원 분포)을 요약한다.

## 하지 않는 것

- 문제지에 담기·교체·출제, 문항 구매는 할 수 없다. 요청하면 고른 문항 ID 목록을 정리해 주고 수학비서 웹에서 진행하라고 안내한다.
- 학생 정보(이름·답안·점수)는 다루지 않는다.
- 도구가 "다시 연결" 류 오류를 내면 `/mathsecr:status` 의 안내대로 재인증하게 한다.
- 웹에서만 되는 일(내려받기·구매·출제 등)을 브라우저 자동화(Chrome 등)로 대신하겠다고 제안하지 않는다. 수학비서 웹에서 직접 하도록 안내만 한다.
- 도구 정의에 있는 입력인데 "입력 형식" 오류가 나거나, "output schema" 불일치 오류가 나면 수학비서 서버가 업데이트돼 도구 정의가 바뀐 것이다. 같은 호출을 되풀이하지 말고 `/mathsecr:status` 안내대로 다시 연결하게 한다. 다시 연결한 뒤에도 같은 대화에서는 앞서 읽은 예전 정의를 따르기 쉬우니, 그 입력은 더 쓰지 않는다.

Claude Code 에서는 여러 번 검색해 후보를 추리는 일을 `question-scout` 에이전트에 맡기면 대화가 짧게 유지된다.
