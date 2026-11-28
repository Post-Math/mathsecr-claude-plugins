# 난이도 사다리 (기준 문항보다 쉬운·어려운 참고 문항)

수학비서 문항 하나를 기준으로, 같은 단원에서 난이도가 한 칸씩 낮고 높은 참고 문항을 고른다. 변형 문제를 만들 때 난이도 단계를 잡는 자료다. 도구 호출은 약 7회 이내로 끝낸다.

1. `cell_detail(cellId, curriculumId)` 로 기준 문항을 읽는다.
   - 기준 난이도 `d` 는 `answerMetadata[].difficulty.mathSecr` 중 가장 큰 값이다(소문항이 여러 개면 최댓값). 회원 본인이 매긴 `difficulty.member` 가 있으면 따로 적는다. 검색 필터는 소문항마다 회원 값을 먼저 쓰고 그 최댓값으로 거른다.
   - 단원은 `chapters[]` 의 5단계 경로다. `hasPermission`·`isUnavailable` 도 확인한다.
2. 기준 문항의 `chapters[].id`(chapter_id)로 `chapter_tree(curriculumId, chapterIds: [id])` 를 1회 부른다. 맞은 5단계 노드의 `tree_id` 를 쓰고(별칭이면 `originTreeId`), 그 부모(4단계) 노드의 `tree_id` 는 넓힐 때 쓴다. 맞은 노드가 없으면 멈추고 그렇다고 보여 준다.
3. `cell_search` 로 단계별 후보를 찾는다.
   - 범위: 사용자가 나만의 DB 를 지정했으면 `problemBankScope: "exclude"` 와 `myDbIds`, 아니면 `problemBankScope: "include"`. 공통으로 `excludeSubQuestions: true`, `excludePassageQuestions: true`, `limit: 5`.
   - 낮은 쪽: `minDifficulty = maxDifficulty = d-1`. 비면 `d-2`. 높은 쪽도 같은 방식(`d+1`, 비면 `d+2`). 1~9 밖의 난이도는 검색하지 않고 "그 쪽 단계 없음"으로 적는다.
   - 같은 5단계 단원에 없으면 4단계 부모 단원(`tree_id`)으로 넓히고 「같은 유형(4단계)으로 넓힘」이라고 적는다. 넓힌 단계는 모두 밝힌다.
   - 각 단계의 `totalCount` 를 함께 적는다(그 난이도에 몇 문항이 있는지). `include` 로 센 수라 이 계정 문제은행 상태(`whoami`, 조회 시점)를 각주에 한 번 적는다(스킬의 「문제은행 구독」).
4. 기준 문항 자신과, 1번 `cell_detail.relations` 의 같은문제(PC)·숫자변형(NC) 문항 ID 는 후보에서 뺀다(난이도가 기준에서 복사된 경우가 많다). `relations` 가 비었거나 잘렸으면(`truncated`) 그 사실을 적는다. `isProblemBank: true` 후보는 `relations` 에 없어 기준 문항과 같은 묶음인지 확인할 수 없다. 「같은 묶음 여부 미확인(문제은행)」이라 적고 다른 후보를 우선한다. 서로 다른 출처(`sources`)를 우선한다.
5. `hasPermission: true`, `isUnavailable: false` 인 문항만 고른다. 「문제은행 구독 필요」(`isProblemBank: true`·`hasPermission: false`)·「문제은행에서 빠진 문항」은 고르지 않는다. 해설이 있는 문항(`isExplanation`)을 우선한다.
6. 단계는 `difficulty.mathSecr` 최댓값 하나로 비교한다. 검색 필터는 소문항마다 회원 값을 먼저 쓰고 그 최댓값으로 거르므로, 후보의 mathSecr 값이 그 단계와 다르면 빼고 그런 후보가 있었다고 적는다. 회원 값은 따로 표시한다.

## 결과

| 단계 | ID | 난이도 | 정답 유형 | 출처 | 해설 | 근거 |
|---|---|---|---|---|---|---|

- 근거 칸에는 `「수학비서 데이터」` 라는 문구를 그대로 쓰고 도구·필드·ID 를 붙인다(예: 「수학비서 데이터」 `cell_search` d=2, tree_id 7113). 도구 이름만 쓰지 않는다.
- 단계마다 그 난이도의 전체 문항 수(`totalCount`)와 바꿔 쓸 후보 1~2개를 적는다.
- 같은 단원에 어떤 난이도가 아예 없으면 그렇다고 말한다. 없는 단계를 다른 단원 문항으로 몰래 채우지 않는다.
- 난이도는 수학비서에 저장된 값이다. 실제로 같은 풀이 흐름에서 어려워지는지는 이미지를 읽지 않으면 확인되지 않는다. 확인이 필요하면 보기 링크(10문항까지 한 번에) 또는 직접 판독(기본 4장)을 제안한다.
