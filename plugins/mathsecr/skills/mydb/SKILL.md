---
name: mydb
description: 수학비서 나만의 DB 조회 — "내 DB 목록", "DB화 끝났어?". 상태·구성 조회, "문항 읽어줘" 판독, "문항 보여줘" 보기 링크.
allowed-tools: mcp__plugin_mathsecr_api__mydb_count, mcp__plugin_mathsecr_api__mydb_list, mcp__plugin_mathsecr_api__mydb_detail, mcp__plugin_mathsecr_api__mydb_cells, mcp__plugin_mathsecr_api__mydb_scores, mcp__plugin_mathsecr_api__mydb_purchasable, mcp__plugin_mathsecr_api__question_quota, mcp__plugin_mathsecr_api__voucher_status, mcp__plugin_mathsecr_api__curriculum_list, mcp__plugin_mathsecr_api__chapter_tree, mcp__plugin_mathsecr_api__cell_image_link, mcp__plugin_mathsecr_api__cell_image, mcp__plugin_mathsecr_api__cell_detail
---

# 수학비서 나만의 DB 조회

회원 본인의 나만의 DB 를 읽어 정리한다. 모든 도구는 읽기 전용이다.

## DB 찾기

- 전체 현황은 `mydb_count` 로 시작한다: 구매함·구매 가능·DB 처리 중·DB 완료·PDF 검토 중 개수.
- 목록은 `mydb_list`.
  - 이름으로 찾으면 `searchMode: "title"` 과 `keyword` 를 쓴다. 폴더(`folderId`)와 제목 검색은 함께 쓸 수 없다.
  - 상태로 거르기: `isDbCompleted`, `isDbProcessing`, `isPdfReviewing`, `isPurchasable`, `isAnswer`(정답 있음), `isExplanation`(해설 있음). 종류: `paperType`(`school_exams` 학교 기출, `general_etc` 기타).
  - 정렬: `order`(`created_at:desc`, `title:asc`, `title:desc`).
  - 더 보려면 `nextCursor` 를 `cursor` 로 넘기고 다른 인자는 그대로 둔다.
- 제목이 비슷한 DB 가 여럿이면 제목·등록일·문항 수로 표를 보여 주고 어느 것인지 묻는다. ID 를 사용자에게 먼저 묻지 않는다.

## DB 살펴보기

- `mydb_detail`(`dbId`): 진행 상태(`progress`, `dbStatus`), 문항 수, 난이도 분포(`difficultyCount`, `averageDifficulty`), 구매 가능 여부. 해설 수와 정답·해설 보유(`explanationCount`, `isAnswer`, `isExplanation`)는 상세에 없다. `mydb_list` 의 같은 DB 항목에서 읽는다.
- `mydb_cells`(`dbId`, `curriculumId`): 쪽별 문항 메타데이터(단원·난이도·정답 유형).
  - `curriculumId` 가 필요하다. 모르면 `curriculum_list` 로 정한다. 단원으로 거르려면 `chapter_tree` 의 `tree_id` 를 `chapterTreeIds` 로 넘긴다.
  - 쪽만 보려면 `pageNumbers`. 커서는 문항 단위다.
- `mydb_scores`(`dbId`): 문항별 **배점**이다. 학생 성적이 아니다. `null` 은 배점 미설정이다.
- `mydb_purchasable`(`dbId`): **업로드한 DB(PDF)** 의 쪽별 인식 문항과 그 문항·해설 이용 권한. 구매·공유받은 DB(`acquisitionType` 이 `PURCHASE`·`SHARED` 등)는 업로드한 쪽이 없어 늘 빈 목록이다. 빈 결과를 오류로 보지 말고, 구매 DB 의 구성은 `mydb_cells` 로 본다. 구매하지 않는다. 응답 안의 쪽 내부 잘림(`nestedTruncated`)은 커서로 이어지지 않으니 "일부 쪽은 요약만 왔다" 고 밝힌다.

## 한도·상품권

- `question_quota`: 출제할 수 있는 문항 수 상한(분석 잔액 아님).
- `voucher_status`: 보유 상품권의 문항 수·수량. 사용·구매하지 않는다.

## 결과 정리

- 목록은 제목·상태·문항 수·해설 여부·등록일로 표를 만들고 `totalCount` 를 함께 말한다. `totalCount` 는 첫 페이지에만 온다. 다음 페이지에 없으면 첫 페이지 값을 그대로 쓴다.
- `sourceless: true` 인 문항은 출처에서 빠진 문항이다(출처가 삭제·해제됨). 출처 칸에 "출처 없음"으로 적는다. 출처 ID 0 도 같은 뜻이다. 출처 정보가 누락됐다거나 오류라고 말하지 않는다.
- 문제은행 문항(`isProblemBank: true`)은 원래 출처 목록이 비어 있다(`sources: []`, `sourceless` 없음). 출처 칸에 "문제은행"으로 적는다.
- `sourceId`·`source` 가 없는 DB 는 출처가 연결되지 않은 DB 다. 출처 ID 0 으로 해석하거나 지어내지 않는다.
- `truncated`·`nestedTruncated` 가 true 면 일부라고 밝힌다.

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

- 업로드·DB화 요청·구매·이름 변경·이동·삭제는 할 수 없다. 수학비서 웹에서 진행하라고 안내한다.
- 메모·업로드 파일명·폴더 이름은 도구가 주지 않는다(폴더는 ID 만 온다).
- 학생 정보는 다루지 않는다. 재인증이 필요하면 `/mathsecr:status` 안내를 따른다.
- 웹에서만 되는 일(내려받기·구매·출제 등)을 브라우저 자동화(Chrome 등)로 대신하겠다고 제안하지 않는다. 수학비서 웹에서 직접 하도록 안내만 한다.
- 도구 정의에 있는 입력인데 "입력 형식" 오류가 나거나, "output schema" 불일치 오류가 나면 수학비서 서버가 업데이트돼 도구 정의가 바뀐 것이다. 같은 호출을 되풀이하지 말고 `/mathsecr:status` 안내대로 다시 연결하게 한다. 다시 연결한 뒤에도 같은 대화에서는 앞서 읽은 예전 정의를 따르기 쉬우니, 그 입력은 더 쓰지 않는다.
