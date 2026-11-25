---
name: myanalysis
description: 수학비서 저장 분석지 조회 — "분석 끝났어?", "분석 결과 요약해줘". 결과 조회, "문항 읽어줘" 판독, "문항 보여줘" 보기 링크.
allowed-tools: mcp__plugin_mathsecr_api__myanalysis_count, mcp__plugin_mathsecr_api__myanalysis_list, mcp__plugin_mathsecr_api__myanalysis_cells, mcp__plugin_mathsecr_api__cell_detail, mcp__plugin_mathsecr_api__cell_image_link, mcp__plugin_mathsecr_api__cell_image
---

# 수학비서 분석지 조회

회원 본인의 분석지와 **이미 저장된** 분석 결과를 읽는다. 모든 도구는 읽기 전용이며 서버의 분석을 새로 실행하지 않는다. 이미지 비교는 Claude의 해석이다.

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
- `sourceless: true` 인 문항은 출처에서 빠진 문항이다(출처가 삭제·해제됨). 출처 칸에 "출처 없음"으로 적는다. 출처 ID 0 도 같은 뜻이다. 출처 정보가 누락됐다거나 오류라고 말하지 않는다.
- 문제은행 문항(`isProblemBank: true`)은 원래 출처 목록이 비어 있다(`sources: []`, `sourceless` 없음). 출처 칸에 "문제은행"으로 적는다.

## 메타데이터·이미지 판독·보기 링크

표의 각 행에 근거 라벨·실제 필드·ID를 적고 생략하지 않는다. 보기 링크의 포함 항목·거절·만료·조회 횟수도 `「수학비서 데이터」 cell_image_link.entries/denied/expiresAt/maxViews + cellId`로 표시한다. 이미지 사용량은 `cell_image.returned/budgetRemainingToday` 또는 사용자 제공값이라고 출처를 붙인다. 오류·권한 차이·잔액 감소의 내부 원인은 추정하지 않는다.

- 단원·난이도·배점·상태 질문은 메타데이터로 답한다. "보여줘"는 `cell_image_link`, "읽어/분석/비교"는 내용이 필요할 때 `cell_image`를 쓴다. 메타데이터 응답에 문항 본문은 없지만 허용된 이미지는 직접 판독할 수 있다.
- 보기 링크: `curriculumId`와 단일 `cellId` **또는** 중복 없는 `cellIds`(최대 10개), `parts: ["question"]` 또는 `["question", "explanation"]`을 보낸다. 한 이미지는 5분·5회, 여러 이미지는 10분·항목당 3회다. 실제 `url`, `expiresAt`, `maxViews`, `entries`를 전달한다. 링크를 직접 열거나 내려받지 않는다(WebFetch·브라우저 자동화 금지).
- 직접 판독: `cell_image(cellId, curriculumId, parts)`는 허용된 문항·지문·해설 이미지를 호출당 최대 4장 반환한다. parts 입력은 `question`(기본)·`explanation`만, `explanationIds`는 확인된 해설 ID만 쓴다(`cell_detail.explanations[].id`, 기본 해설은 `isMain`·선택 해설은 `isChosen`). 해설은 정답과 전체 풀이를 포함하므로 필요한 분석에만 쓰고, 요청 없이 정답을 노출하지 않는다. 문항·풀이·LaTeX 전사는 하지 않는다.
- 회원 공용 일일 이미지 예산은 **300장**. 요청 누적 기본 **4장**, 필요한 비교는 이유를 알리고 **8장 이내**, 8장 초과 전에는 사용자에게 묻는다. 위임 전 사용량을 차감하며 순차 호출한다. 호출 전 여유는 이번 호출이 돌려줄 수 있는 최대 장수로 계산한다(question 1장, 지문이 있으면 +1, 해설은 지정한 해설 ID 수, 기본 1장), `returned` 장수로 정산한다. `budgetRemainingToday`가 이번 호출 예상 장수보다 작으면 멈춘다. 이미지 호출은 병렬로 하지 않는다(동시 호출 중에는 다른 호출의 예약분이 잔액에서 잠시 빠져 보인다). 잔액은 가장 최근 응답의 `budgetRemainingToday`를 쓰고, 도구 오류로 잔액이 오지 않으면 직전 잔액에서 요청 장수를 뺀 값으로 이어 간다. 직전 잔액이 없으면 멈춘다. 실패한 부분은 차감되지 않으며 `denied`에 사유가 온다. 응답에 `fallback: "cell_image_link"`가 있으면 읽지 못한 그 부분은 사용자에게 보기 링크를 제안한다.
- 두 이미지 도구 모두 `denied`에 거절된 문항·부분·사유가 온다. 이를 보고하고 자동 재시도하지 않는다. `cell_image`의 "이미지를 가져올 수 없습니다" 오류도 재시도하지 않고 `cell_image_link` 보기를 제안하며 수학비서 데이터만으로 계속한다.
- 모든 분석 근거를 `「수학비서 데이터」 도구.필드 + ID` 또는 `「이미지 판독(Claude 해석)」 cellId + part + 판독자`로 구분한다. 해석을 저장 정보라고 쓰지 않는다. `similar_cells` 결과에는 `hasPermission`/`isUnavailable`이 없으므로 활용 전 `cell_detail`로 확인한다.
- 깊은 진단·비교·교체 추천은 `analysis` 스킬로 이어 간다. Code에서는 선택한 레시피 전문·메타데이터 출처·이미지 사용량/잔액을 `math-analyst`에 전달한다. 최종 분석자가 근거 이미지를 직접 가져온다. 앞선 판독은 판독자·출처를 보존하고 재판독하지 않았다고 표시한다.

## 하지 않는 것

- 분석 실행·재실행, 기준 문항 담기·변경, 한글 파일 만들기·내려받기, 이름 변경·이동·삭제는 할 수 없다. 수학비서 웹에서 진행하라고 안내한다. 재실행은 이전 결과를 지우므로 특히 웹에서 확인하게 한다.
- 메모는 도구가 주지 않는다. 학생 정보는 다루지 않는다. 재인증이 필요하면 `/mathsecr:status` 안내를 따른다.
- 웹에서만 되는 일(내려받기·구매·출제 등)을 브라우저 자동화(Chrome 등)로 대신하겠다고 제안하지 않는다. 수학비서 웹에서 직접 하도록 안내만 한다.
- 도구 정의에 있는 입력인데 "입력 형식" 오류가 나거나, "output schema" 불일치 오류가 나면 수학비서 서버가 업데이트돼 도구 정의가 바뀐 것이다. 같은 호출을 되풀이하지 말고 `/mathsecr:status` 안내대로 다시 연결하게 한다. 다시 연결한 뒤에도 같은 대화에서는 앞서 읽은 예전 정의를 따르기 쉬우니, 그 입력은 더 쓰지 않는다.
