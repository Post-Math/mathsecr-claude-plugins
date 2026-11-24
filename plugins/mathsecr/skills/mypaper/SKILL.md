---
name: mypaper
description: 수학비서 문제지 조회 — 내 문제지·학습지·시험지의 목록·진행 상태·문항 구성·배점·페이지 배치·문제은행 변경을 본다. "내 문제지 목록", "○○ 시험지 문항 구성 보여줘", "배점 합계 맞아?", "채점할 거 남았어?", "문제은행에서 바뀐 문항 있어?" 같은 요청에 쓴다. 문제지를 만들거나 고치거나 배포하지 않는다.
allowed-tools: mcp__plugin_mathsecr_api__mypaper_count, mcp__plugin_mathsecr_api__mypaper_list, mcp__plugin_mathsecr_api__mypaper_detail, mcp__plugin_mathsecr_api__mypaper_cells, mcp__plugin_mathsecr_api__mypaper_pages, mcp__plugin_mathsecr_api__mypaper_scores, mcp__plugin_mathsecr_api__mypaper_cell_changes, mcp__plugin_mathsecr_api__curriculum_list, mcp__plugin_mathsecr_api__cell_detail
---

# 수학비서 문제지 조회

회원 본인의 문제지(`paper` 문제지, `study` 학습지, `test` 시험지)를 읽는다. 모든 도구는 읽기 전용이다. 재학습지는 다루지 않는다.

## 문제지 찾기

- `mypaper_count`: **채점 확인 필요**(`ungradedCount`)와 **분석 리포트 생성 필요**(`statisticsCount`) 건수다. 전체 문제지 수가 아니다.
- `mypaper_list`: 제목·종류·진행·문항 수·배점 합계·난이도 분포·응시 학생 수(집계 숫자).
  - 이름으로 찾으면 `searchMode: "title"` 과 `keyword`. 종류는 `category`, 상태는 `status`(`editing` 편집 중, `completed` 출제 완료, `recentlyCompleted`, `ungraded` 채점 필요, `graded` 채점 완료).
  - 정렬: `order`(`created_at:desc`, `updated_at:desc`, `title:asc`, `title:desc`). 폴더는 `folderId`.
  - 더 보려면 `nextCursor` 를 `cursor` 로 넘기고 다른 인자는 그대로 둔다.
- 비슷한 제목이 여럿이면 표로 보여 주고 어느 것인지 묻는다.

## 문제지 살펴보기

- `mypaper_detail`(`paperId`): 진행(`progress`), 문항·해설 수, 난이도 분포, 배점 설정 여부와 합계, 한글 파일 생성 상태(`hmlStatus`). 파일 내려받기는 제공하지 않는다.
- `mypaper_cells`·`mypaper_pages` 는 **출제 완료 문제지만** 조회한다. 편집 중이면 도구가 거절하니 "편집 중인 문제지는 수학비서 웹에서 확인해 주세요" 라고 전한다(조회만으로도 웹이 문제지를 정리하는 동작이 있어 막아 두었다).
  - 둘 다 `paperId` 와 `curriculumId` 가 필요하다. 모르면 `curriculum_list` 로 정한다. 응답의 `curriculumId` 는 출제 당시 보관된 값이라 요청과 다를 수 있고, 그 값을 기준으로 설명한다.
  - `mypaper_cells`: 단원별(또는 `isNumberSorted: true` 면 번호순) 문항 메타데이터. `lockedProblemBankCount` 는 구독이 끝나 잠긴 문제은행 문항 수다.
  - `mypaper_pages`: 문항 사이의 개념·간지 페이지 배치(`concept`, `divider`). 전체 레이아웃이 아니며(`partialLayout`) 페이지 내용은 주지 않는다.
- `mypaper_scores`(`paperId`): 문항별 **배점**이다. 학생 성적이 아니다. `null` 은 미설정, `0` 은 0점으로 설정한 것이다.
- `mypaper_cell_changes`(`paperId`): 문제은행에서 바뀐 문항 수·사유와 갱신 가능 여부(`refreshable`, 막힌 이유 `blockedReason`). 갱신은 하지 않는다.
- 특정 문항의 단원·난이도는 `cell_detail` 로 확인한다.

## 점검 요청에 답하는 법

"배점 맞아?" 같은 요청은 `mypaper_detail` 의 `totalScore` 와 `mypaper_scores` 합계를 비교하고, 미설정(`null`) 문항 번호를 짚는다. "구성 어때?" 는 단원별 문항 수와 난이도 분포로 요약한다. 결과가 잘렸으면(`truncated`·`nestedTruncated`) 일부라고 밝힌다.

- `sourceless: true` 인 문항은 출처에서 빠진 문항이다(출처가 삭제·해제됨). 출처 칸에 "출처 없음"으로 적는다. 출처 ID 0 도 같은 뜻이다. 출처 정보가 누락됐다거나 오류라고 말하지 않는다.
- 문제은행 문항(`isProblemBank: true`)은 원래 출처 목록이 비어 있다(`sources: []`, `sourceless` 없음). 출처 칸에 "문제은행"으로 적는다.

## 하지 않는 것

- 문제지 만들기·문항 담기/교체·배점 설정·출제·학생 배포·채점·재학습지·파일 내려받기·이름 변경·삭제는 할 수 없다. 수학비서 웹에서 진행하라고 안내한다.
- 학생별 풀이·점수·명단은 다루지 않는다(응시 학생 수 같은 집계만 있다). 문항 본문·이미지·정답·해설과 메모도 도구가 주지 않는다.
- 재인증이 필요하면 `/mathsecr:status` 안내를 따른다.
- 웹에서만 되는 일(내려받기·구매·출제 등)을 브라우저 자동화(Chrome 등)로 대신하겠다고 제안하지 않는다. 수학비서 웹에서 직접 하도록 안내만 한다.
- 도구 정의에 있는 입력인데 "입력 형식" 오류가 나거나, "output schema" 불일치 오류가 나면 수학비서 서버가 업데이트돼 도구 정의가 바뀐 것이다. 같은 호출을 되풀이하지 말고 `/mathsecr:status` 안내대로 다시 연결하게 한다. 다시 연결한 뒤에도 같은 대화에서는 앞서 읽은 예전 정의를 따르기 쉬우니, 그 입력은 더 쓰지 않는다.

Claude Code 에서는 문제지 하나를 여러 번 읽어 점검하는 일을 `paper-reviewer` 에이전트에 맡기면 대화가 짧게 유지된다.
