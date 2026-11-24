---
name: mydb
description: 수학비서 나만의 DB 조회 — 내가 올리거나 구매한 DB(시험지·교재)의 목록·상태·문항 구성·배점·이용 권한을 본다. "내 DB 목록", "DB화 끝났어?", "○○ 기출 DB 문항 보여줘", "이 DB 몇 문항이야?", "해설 있는 DB", "상품권 남았어?" 같은 요청에 쓴다. 업로드·구매·삭제는 하지 않는다.
allowed-tools: mcp__plugin_mathsecr_api__mydb_count, mcp__plugin_mathsecr_api__mydb_list, mcp__plugin_mathsecr_api__mydb_detail, mcp__plugin_mathsecr_api__mydb_cells, mcp__plugin_mathsecr_api__mydb_scores, mcp__plugin_mathsecr_api__mydb_purchasable, mcp__plugin_mathsecr_api__question_quota, mcp__plugin_mathsecr_api__voucher_status, mcp__plugin_mathsecr_api__curriculum_list, mcp__plugin_mathsecr_api__chapter_tree
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

- `mydb_detail`(`dbId`): 진행 상태(`progress`, `dbStatus`), 문항·해설 수, 난이도 분포(`difficultyCount`, `averageDifficulty`), 정답·해설 보유와 구매 가능 여부.
- `mydb_cells`(`dbId`, `curriculumId`): 쪽별 문항 메타데이터(단원·난이도·정답 유형).
  - `curriculumId` 가 필요하다. 모르면 `curriculum_list` 로 정한다. 단원으로 거르려면 `chapter_tree` 의 `tree_id` 를 `chapterTreeIds` 로 넘긴다.
  - 쪽만 보려면 `pageNumbers`. 커서는 문항 단위다.
- `mydb_scores`(`dbId`): 문항별 **배점**이다. 학생 성적이 아니다. `null` 은 배점 미설정이다.
- `mydb_purchasable`(`dbId`): 쪽별로 문항·해설을 이미 이용할 수 있는지. 구매하지 않는다. 응답 안의 쪽 내부 잘림(`nestedTruncated`)은 커서로 이어지지 않으니 "일부 쪽은 요약만 왔다" 고 밝힌다.

## 한도·상품권

- `question_quota`: 출제할 수 있는 문항 수 상한(분석 잔액 아님).
- `voucher_status`: 보유 상품권의 문항 수·수량. 사용·구매하지 않는다.

## 결과 정리

- 목록은 제목·상태·문항 수·해설 여부·등록일로 표를 만들고 `totalCount` 를 함께 말한다.
- 문항 본문·이미지·정답은 도구가 주지 않는다. 지어내지 말고 웹에서 보라고 안내한다.
- `truncated`·`nestedTruncated` 가 true 면 일부라고 밝힌다.

## 하지 않는 것

- 업로드·DB화 요청·구매·이름 변경·이동·삭제는 할 수 없다. 수학비서 웹에서 진행하라고 안내한다.
- 메모·업로드 파일명·폴더 이름은 도구가 주지 않는다(폴더는 ID 만 온다).
- 학생 정보는 다루지 않는다. 재인증이 필요하면 `/mathsecr:status` 안내를 따른다.
