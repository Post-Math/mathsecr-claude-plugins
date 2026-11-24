---
name: myanalysis
description: 수학비서 분석지 조회 — 내 분석지 목록·진행 상태와 저장된 분석 결과(기준 문항별 같은문제·숫자변형·유사문제)를 본다. "내 분석지 보여줘", "분석 끝났어?", "○○ 분석지 결과 요약해줘", "기준 문항마다 유사문제 몇 개야?" 같은 요청에 쓴다. 분석을 새로 실행하거나 파일을 만들지 않는다.
allowed-tools: mcp__plugin_mathsecr_api__myanalysis_count, mcp__plugin_mathsecr_api__myanalysis_list, mcp__plugin_mathsecr_api__myanalysis_cells, mcp__plugin_mathsecr_api__cell_detail
---

# 수학비서 분석지 조회

회원 본인의 분석지와 **이미 저장된** 분석 결과를 읽는다. 모든 도구는 읽기 전용이며 분석을 실행하지 않는다.

## 분석지 찾기

- `myanalysis_count`: 분석 중·완료 개수.
- `myanalysis_list`: 제목·상태·기준 문항 수·분석 수·해설 수·파일(`hmlStatus`) 상태.
  - 이름으로 찾으면 `searchMode: "title"` 과 `keyword`. 상태로 거르면 `analysisStatus`(`analyzing`, `completed`).
  - 정렬: `order`(`created_at:desc`, `updated_at:desc`, `title:asc`, `title:desc`). 폴더는 `folderId`.
  - 더 보려면 `nextCursor` 를 `cursor` 로 넘기고 다른 인자는 그대로 둔다.
- 비슷한 제목이 여럿이면 표로 보여 주고 어느 것인지 묻는다.

## 결과 읽기

- `myanalysis_cells`(`analysisId`): 기준 문항마다 비교 문항(결과)과 관계 종류(`analysisType`), 선택 여부(`isSelected`)를 준다.
  - 관계 종류: `PC` 같은문제, `NC` 숫자변형, `5D` 유사문제, `TX` 텍스트 유사성.
  - `standardLimit`(기준 문항 수, 최대 20)과 `resultLimit`(기준 문항당 결과 수, 최대 5)은 **출력 상한이지 페이지가 아니다.** 커서가 없으므로 더 많이 보려면 상한을 올려 다시 부른다.
  - `resultsOmittedCount` 는 그 기준 문항에서 생략된 결과 수다. `truncated`·`nestedTruncated` 가 true 면 일부라고 밝힌다.
- 분석이 끝나지 않았으면(`analysisStatus` 가 `analyzing`) 결과가 비어 있을 수 있다고 알린다.
- 특정 문항의 단원·난이도는 `cell_detail`(`cellId`, `curriculumId` — 분석지의 `curriculumId` 사용)로 확인한다.

## 결과 정리

- 기준 문항별로 "같은문제 n · 숫자변형 n · 유사문제 n" 처럼 세고, 결과가 없는 기준 문항을 따로 표시한다.
- 문항 본문·이미지·정답·해설은 도구가 주지 않는다. 지어내지 말고 웹에서 보라고 안내한다.

## 하지 않는 것

- 분석 실행·재실행, 기준 문항 담기·변경, 한글 파일 만들기·내려받기, 이름 변경·이동·삭제는 할 수 없다. 수학비서 웹에서 진행하라고 안내한다. 재실행은 이전 결과를 지우므로 특히 웹에서 확인하게 한다.
- 메모는 도구가 주지 않는다. 학생 정보는 다루지 않는다. 재인증이 필요하면 `/mathsecr:status` 안내를 따른다.
