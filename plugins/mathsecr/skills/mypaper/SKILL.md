---
name: mypaper
description: 수학비서 문제지 조회 — "내 문제지 목록", "배점 합계 맞아?". 구성·상태 조회, "문항 읽어줘" 판독, "문항 보여줘" 보기 링크.
allowed-tools: mcp__plugin_mathsecr_api__mypaper_count, mcp__plugin_mathsecr_api__mypaper_list, mcp__plugin_mathsecr_api__mypaper_detail, mcp__plugin_mathsecr_api__mypaper_cells, mcp__plugin_mathsecr_api__mypaper_pages, mcp__plugin_mathsecr_api__mypaper_scores, mcp__plugin_mathsecr_api__mypaper_cell_changes, mcp__plugin_mathsecr_api__curriculum_list, mcp__plugin_mathsecr_api__cell_detail, mcp__plugin_mathsecr_api__cell_image_link, mcp__plugin_mathsecr_api__cell_image
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

- `mypaper_detail`(`paperId`): 진행(`progress`), 문항·해설 수, 난이도 분포, 한글 파일 생성 상태(`hmlStatus`). 총점(`totalScore`)과 배점 설정 여부(`hasScore`)는 상세에 없고 `mypaper_list` 항목에만 있다. 파일 내려받기는 제공하지 않는다.
- `mypaper_cells`·`mypaper_pages` 는 **출제 완료 문제지만** 조회한다. 편집 중이면 도구가 거절하니 "편집 중인 문제지는 수학비서 웹에서 확인해 주세요" 라고 전한다(조회만으로도 웹이 문제지를 정리하는 동작이 있어 막아 두었다).
  - 둘 다 `paperId` 와 `curriculumId` 가 필요하다. 모르면 `curriculum_list` 로 정한다. 응답의 `curriculumId` 는 출제 당시 보관된 값이라 요청과 다를 수 있고, 그 값을 기준으로 설명한다. 다만 문항이 하나도 오지 않으면(아래 잠김 등) 응답의 `curriculumId` 는 요청값을 되돌려 준 것이라 보관된 값으로 말하지 않는다.
  - `mypaper_cells`: 단원별(또는 `isNumberSorted: true` 면 번호순) 문항 메타데이터. `lockedProblemBankCount` 가 1 이상이면 구독이 끝나 문제은행 문항이 잠겨 출제 완료 문제지의 문항이 오지 않는다 — 빈 문제지가 아니다. 그 필드가 없거나 null 인데 문항이 오지 않으면 구독을 확인하지 못한 것일 수 있어 비었다고 단정하지 않는다. 문항 수는 `mypaper_detail` 로 적고, 확인은 수학비서 웹에서 하도록 안내한다(구독을 권하지 않는다).
  - `mypaper_pages`: 문항 사이의 개념·간지 페이지 배치(`concept`, `divider`). 전체 레이아웃이 아니며(`partialLayout`) 페이지 내용은 주지 않는다.
- `mypaper_scores`(`paperId`): 문항별 **배점**이다. 학생 성적이 아니다. `null` 은 미설정, `0` 은 0점으로 설정한 것이다.
- `mypaper_cell_changes`(`paperId`): 문제은행에서 바뀐 문항 수·사유와 갱신 가능 여부(`refreshable`, 막힌 이유 `blockedReason`). `cellId`는 사유가 `replaced`면 지금 원본 문제지의 문항(갱신하면 들어올 문항), 그 밖의 사유면 출제본 문항이라 `mypaper_cells`의 같은 번호와 다를 수 있다. `comparisonAvailable`이 false면 `changedCount`는 하한이다. `replaced`의 새 `cellId`가 출제본의 다른 번호에 이미 있으면 갱신 뒤 두 번호가 같은 문항이 된다고 미리 알린다. 출제 완료 문제지에서 같은 `cellId`가 두 번호에 있는 것은 같은 문제 처리 뒤 번호·배점·학생 답안 순서를 지키려고 둘 다 남긴 것일 수 있으니 오류로 보지 않는다. 바꾸려면 수학비서 웹에서 교체한다. 갱신은 하지 않는다.
- 특정 문항의 단원·난이도는 `cell_detail` 로 확인한다.

## 점검 요청에 답하는 법

"배점 맞아?" 같은 요청은 `mypaper_scores` 합계(Claude 계산)를 `mypaper_list` 항목의 `totalScore` 와 비교하고(`mypaper_detail` 에는 `totalScore`·`hasScore` 가 없다), 미설정(`null`) 문항 번호를 짚는다. "구성 어때?" 는 단원별 문항 수와 난이도 분포로 요약한다. 결과가 잘렸으면(`truncated`·`nestedTruncated`) 일부라고 밝힌다.

- `sourceless: true` 인 문항은 출처에서 빠진 문항이다(출처가 삭제·해제됨). 출처 칸에 "출처 없음"으로 적는다. 출처 ID 0 도 같은 뜻이다. 출처 정보가 누락됐다거나 오류라고 말하지 않는다.
- 문제은행 문항(`isProblemBank: true`)은 원래 출처 목록이 비어 있다(`sources: []`, `sourceless` 없음). 출처 칸에 "문제은행"으로 적는다.
- **문제은행 구독.** 문제은행 문항은 회원의 구독 상태에 따라 열 수 있는지가 달라진다. 이 계정의 결과가 다른 계정에도 같다고 말하지 않는다. 구독 상태를 알려 주는 도구는 없으므로 구독 여부를 단정하지 않고 구독을 권하지 않는다. 확인·변경은 수학비서 웹에서 하도록 안내한다.
  - `isProblemBank: true`·`hasPermission: false`·`isProblemBankRemoved` 가 true 아님 = 「문제은행 구독 필요」. 후보·앵커·난이도 단계·대체 추천에 쓰지 않고 `cell_image`·`cell_image_link` 를 부르지 않는다. 단원·난이도는 「잠긴 문항의 저장값」으로만 적는다.
  - `isProblemBankRemoved: true` = 「문제은행에서 빠진 문항」. 구독과 상관없이 쓰지 않는다.
  - 그 밖의 `hasPermission: false` = 「권한 없음」, `isUnavailable: true` = 「사용 불가」.
  - 개수는 「쓸 수 있음 N / 열 수 없음 M(사유별)」으로 나눠 적고, 나눌 수 없으면 「잠긴 문항 포함 여부 미확인」이라고 밝힌다.

## 메타데이터·이미지 판독·보기 링크

표의 각 행에 근거 라벨·실제 필드·ID를 적고 생략하지 않는다. 보기 링크의 포함 항목·거절·만료·조회 횟수도 `「수학비서 데이터」 cell_image_link.entries/denied/expiresAt/maxViews + cellId`로 표시한다. 이미지 사용량은 `cell_image.returned/budgetRemainingToday` 또는 사용자 제공값이라고 출처를 붙인다. 오류·권한 차이·잔액 감소의 내부 원인은 추정하지 않는다.

- 단원·난이도·배점·상태 질문은 메타데이터로 답한다. "보여줘"는 `cell_image_link`, "읽어/분석/비교"는 내용이 필요할 때 `cell_image`를 쓴다. 메타데이터 응답에 문항 본문은 없지만 허용된 이미지는 직접 판독할 수 있다.
- 보기 링크: `curriculumId`와 단일 `cellId` **또는** 중복 없는 `cellIds`(최대 10개), `parts: ["question"]` 또는 `["question", "explanation"]`을 보낸다. 한 이미지는 5분·5회, 여러 이미지는 10분·항목당 3회다. 실제 `url`, `expiresAt`, `maxViews`, `entries`를 전달한다. 링크를 직접 열거나 내려받지 않는다(WebFetch·브라우저 자동화 금지).
- 직접 판독: `cell_image(cellId, curriculumId, parts)`는 허용된 문항·지문·해설 이미지를 호출당 최대 4장 반환한다. parts 입력은 `question`(기본)·`explanation`만, `explanationIds`는 확인된 해설 ID만 쓴다(`cell_detail.explanations[].id`, 기본 해설은 `isMain`·선택 해설은 `isChosen`). 해설은 정답과 전체 풀이를 포함하므로 필요한 분석에만 쓰고, 요청 없이 정답을 노출하지 않는다. 문항·풀이·LaTeX 전사는 하지 않는다.
- 회원 공용 일일 이미지 예산은 **300장**. 요청 누적 기본 **4장**, 필요한 비교는 이유를 알리고 **8장 이내**, 8장 초과 전에는 사용자에게 묻는다. 위임 전 사용량을 차감하며 순차 호출한다. 호출 전 여유는 이번 호출이 돌려줄 수 있는 최대 장수로 계산한다(question 1장, 지문이 있으면 +1, 해설은 지정한 해설 ID 수, 기본 1장), `returned` 장수로 정산한다. `budgetRemainingToday`가 이번 호출 예상 장수보다 작으면 멈춘다. 이미지 호출은 병렬로 하지 않는다(동시 호출 중에는 다른 호출의 예약분이 잔액에서 잠시 빠져 보인다). 잔액은 가장 최근 응답의 `budgetRemainingToday`를 쓰고, 도구 오류로 잔액이 오지 않으면 직전 잔액에서 요청 장수를 뺀 값으로 이어 간다. 직전 잔액이 없으면 멈춘다. 실패한 부분은 차감되지 않으며 `denied`에 사유가 온다(해설은 `explanationId` 포함; `not_found` = 그 문항의 해설 ID가 아님, `no_explanation` = 해설이 없는 문항, `permission_denied` = 권한 없음). 응답에 `fallback: "cell_image_link"`가 있으면 읽지 못한 그 부분은 사용자에게 보기 링크를 제안한다.
- 두 이미지 도구 모두 `denied`에 거절된 문항·부분·사유가 온다. 이를 보고하고 자동 재시도하지 않는다. `cell_image`의 "이미지를 가져올 수 없습니다" 오류도 재시도하지 않고 `cell_image_link` 보기를 제안하며 수학비서 데이터만으로 계속한다.
- 모든 분석 근거를 `「수학비서 데이터」 도구.필드 + ID` 또는 `「이미지 판독(Claude 해석)」 cellId + part + 판독자`로 구분한다. 해석을 저장 정보라고 쓰지 않는다. `similar_cells` 결과에는 `hasPermission`/`isUnavailable`이 없으므로 활용 전 `cell_detail`로 확인한다.
- 깊은 진단·비교·교체 추천은 `analysis` 스킬로 이어 간다. Code에서는 선택한 레시피 전문·메타데이터 출처·이미지 사용량/잔액을 `math-analyst`에 전달한다. 최종 분석자가 근거 이미지를 직접 가져온다. 앞선 판독은 판독자·출처를 보존하고 재판독하지 않았다고 표시한다.

## 하지 않는 것

- 문제지 만들기·문항 담기/교체·배점 설정·출제·학생 배포·채점·재학습지·파일 내려받기·이름 변경·삭제는 할 수 없다. 수학비서 웹에서 진행하라고 안내한다.
- 학생별 풀이·점수·명단은 다루지 않는다(응시 학생 수 같은 집계만 있다). 메모는 제공하지 않는다.
- 재인증이 필요하면 `/mathsecr:status` 안내를 따른다.
- 웹에서만 되는 일(내려받기·구매·출제 등)을 브라우저 자동화(Chrome 등)로 대신하겠다고 제안하지 않는다. 수학비서 웹에서 직접 하도록 안내만 한다.
- 도구 정의에 있는 입력인데 "입력 형식" 오류가 나거나, "output schema" 불일치 오류가 나면 수학비서 서버가 업데이트돼 도구 정의가 바뀐 것이다. 같은 호출을 되풀이하지 말고 `/mathsecr:status` 안내대로 다시 연결하게 한다. 다시 연결한 뒤에도 같은 대화에서는 앞서 읽은 예전 정의를 따르기 쉬우니, 그 입력은 더 쓰지 않는다.

Claude Code 에서는 문제지 하나를 여러 번 읽어 점검하는 일을 `paper-reviewer` 에이전트에 맡기면 대화가 짧게 유지된다.
