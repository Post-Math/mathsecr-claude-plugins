---
name: questions
description: 수학비서 문항 탐색 — "이차함수 문항 찾아줘", "비슷한 문제 찾아줘". 메타데이터 조회, "읽어줘" 이미지 판독, "보여줘" 보기 링크.
allowed-tools: mcp__plugin_mathsecr_api__curriculum_list, mcp__plugin_mathsecr_api__chapter_tree, mcp__plugin_mathsecr_api__cell_tag_search, mcp__plugin_mathsecr_api__cell_search, mcp__plugin_mathsecr_api__cell_detail, mcp__plugin_mathsecr_api__similar_search_setting, mcp__plugin_mathsecr_api__similar_cells, mcp__plugin_mathsecr_api__question_quota, mcp__plugin_mathsecr_api__cell_image_link, mcp__plugin_mathsecr_api__cell_image
---

# 수학비서 문항 탐색

수학비서 문제은행과 회원 본인의 나만의 DB 에서 조건에 맞는 문항을 찾아 정리한다. 모든 도구는 읽기 전용이다.

## 순서

1. **교육과정 확정.** `curriculum_list` 로 교육과정 ID 를 찾는다. 사용자가 "고1", "공통수학1" 처럼 말하면 목록에서 맞는 것을 고르고, 둘 이상이면 어느 것인지 묻는다. 기본 교육과정(`is_default`)이 있으면 그것을 먼저 제안한다.
2. **단원 확정.** `chapter_tree` 를 `curriculumId` 와 `keyword`(예: "이차함수")로 부른다.
   - 요청이 상위 단원의 일부라면 해당 하위 단원만 고른다. 예를 들어 "이차함수" 요청에 "이차방정식과 이차함수" 전체를 넣지 말고 이차함수 하위 단원의 tree_id를 사용한다.
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
   - 태그 사전과 검색 필터의 근거만 제시하고 「문항별 부착 태그 확인 불가」를 적는다. 문항별 태그를 추측하지 않는다.
6. **한도.** "몇 문항까지 출제할 수 있어?" 는 `question_quota` 로 답한다. 분석 잔액이 아니다.

## 결과 정리

- 문항은 **ID·단원 경로·난이도·정답 유형·문제은행 여부**로 표를 만든다. 내용 확인이 필요하면 아래 이미지 경로를 쓴다.
- `hasPermission` 이 false 거나 `isUnavailable`·`isProblemBankRemoved` 가 true 인 문항은 표에 표시해 구분한다.
- `sourceless: true` 인 문항은 출처에서 빠진 문항이다(출처가 삭제·해제됨). 출처 칸에 "출처 없음"으로 적는다. 출처 ID 0 도 같은 뜻이다. 출처 정보가 누락됐다거나 오류라고 말하지 않는다.
- 문제은행 문항(`isProblemBank: true`)은 원래 출처 목록이 비어 있다(`sources: []`, `sourceless` 없음). 출처 칸에 "문제은행"으로 적는다.
- `totalCount` 와 실제로 받은 개수(`returnedCount`)를 함께 말한다. `truncated`·`nestedTruncated` 가 true 면 일부라고 밝힌다.
- 후보가 많으면 상위 몇 개만 보이고 기준(난이도·단원 분포)을 요약한다.

## 메타데이터·이미지 판독·보기 링크

표의 각 행에 근거 라벨·실제 필드·ID를 적고 생략하지 않는다. 보기 링크의 포함 항목·거절·만료·조회 횟수도 `「수학비서 데이터」 cell_image_link.entries/denied/expiresAt/maxViews + cellId`로 표시한다. 이미지 사용량은 `cell_image.returned/budgetRemainingToday` 또는 사용자 제공값이라고 출처를 붙인다. 오류·권한 차이·잔액 감소의 내부 원인은 추정하지 않는다.

- 단원·난이도·배점·상태 질문은 메타데이터로 답한다. "보여줘"는 `cell_image_link`, "읽어/분석/비교"는 내용이 필요할 때 `cell_image`를 쓴다. 메타데이터 응답에 문항 본문은 없지만 허용된 이미지는 직접 판독할 수 있다.
- 보기 링크: `curriculumId`와 단일 `cellId` **또는** 중복 없는 `cellIds`(최대 10개), `parts: ["question"]` 또는 `["question", "explanation"]`을 보낸다. 한 이미지는 5분·5회, 여러 이미지는 10분·항목당 3회다. 실제 `url`, `expiresAt`, `maxViews`, `entries`를 전달한다. 링크를 직접 열거나 내려받지 않는다(WebFetch·브라우저 자동화 금지).
- 직접 판독: `cell_image(cellId, curriculumId, parts)`는 허용된 문항·지문·해설 이미지를 호출당 최대 4장 반환한다. parts 입력은 `question`(기본)·`explanation`만, `explanationIds`는 확인된 해설 ID만 쓴다. 해설은 정답과 전체 풀이를 포함하므로 필요한 분석에만 쓰고, 요청 없이 정답을 노출하지 않는다. 문항·풀이·LaTeX 전사는 하지 않는다.
- 회원 공용 일일 이미지 예산은 **300장**. 요청 누적 기본 **4장**, 필요한 비교는 이유를 알리고 **8장 이내**, 8장 초과 전에는 사용자에게 묻는다. 위임 전 사용량을 차감하며 순차 호출한다. 호출 전 여유는 이번 호출이 돌려줄 수 있는 최대 장수로 계산한다(question 1장, 지문이 있으면 +1, 해설은 지정한 해설 ID 수, 기본 1장), `returned` 장수로 정산한다. `budgetRemainingToday`가 이번 호출 예상 장수보다 작으면 멈춘다. 이미지 호출은 병렬로 하지 않는다(동시 호출 중에는 다른 호출의 예약분이 잔액에서 잠시 빠져 보인다). 잔액은 가장 최근 응답의 `budgetRemainingToday`를 쓰고, 도구 오류로 잔액이 오지 않으면 직전 잔액에서 요청 장수를 뺀 값으로 이어 간다. 직전 잔액이 없으면 멈춘다. 실패한 부분은 차감되지 않으며 `denied`에 사유가 온다. 응답에 `fallback: "cell_image_link"`가 있으면 읽지 못한 그 부분은 사용자에게 보기 링크를 제안한다.
- 두 이미지 도구 모두 `denied`에 거절된 문항·부분·사유가 온다. 이를 보고하고 자동 재시도하지 않는다. `cell_image`의 "이미지를 가져올 수 없습니다" 오류도 재시도하지 않고 `cell_image_link` 보기를 제안하며 수학비서 데이터만으로 계속한다.
- 모든 분석 근거를 `「수학비서 데이터」 도구.필드 + ID` 또는 `「이미지 판독(Claude 해석)」 cellId + part + 판독자`로 구분한다. 해석을 저장 정보라고 쓰지 않는다. `similar_cells` 결과에는 `hasPermission`/`isUnavailable`이 없으므로 활용 전 `cell_detail`로 확인한다.
- 깊은 진단·비교·교체 추천은 `analysis` 스킬로 이어 간다. Code에서는 선택한 레시피 전문·메타데이터 출처·이미지 사용량/잔액을 `math-analyst`에 전달한다. 최종 분석자가 근거 이미지를 직접 가져온다. 앞선 판독은 판독자·출처를 보존하고 재판독하지 않았다고 표시한다.

## 하지 않는 것

- 문제지에 담기·교체·출제, 문항 구매는 할 수 없다. 요청하면 고른 문항 ID 목록을 정리해 주고 수학비서 웹에서 진행하라고 안내한다.
- 학생 정보(이름·답안·점수)는 다루지 않는다.
- 도구가 "다시 연결" 류 오류를 내면 `/mathsecr:status` 의 안내대로 재인증하게 한다.
- 웹에서만 되는 일(내려받기·구매·출제 등)을 브라우저 자동화(Chrome 등)로 대신하겠다고 제안하지 않는다. 수학비서 웹에서 직접 하도록 안내만 한다.
- 도구 정의에 있는 입력인데 "입력 형식" 오류가 나거나, "output schema" 불일치 오류가 나면 수학비서 서버가 업데이트돼 도구 정의가 바뀐 것이다. 같은 호출을 되풀이하지 말고 `/mathsecr:status` 안내대로 다시 연결하게 한다. 다시 연결한 뒤에도 같은 대화에서는 앞서 읽은 예전 정의를 따르기 쉬우니, 그 입력은 더 쓰지 않는다.

Claude Code 에서는 여러 번 검색해 후보를 추리는 일을 `question-scout` 에이전트에 맡기면 대화가 짧게 유지된다.
