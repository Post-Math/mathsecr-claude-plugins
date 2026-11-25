---
name: analysis
description: 수학비서 데이터 분석 — "문제지 진단해줘", "두 문항 비교해줘", "대체 문항 추천해줘". 저장 정보와 필요한 이미지 판독으로 근거를 정리한다.
allowed-tools: mcp__plugin_mathsecr_api__curriculum_list, mcp__plugin_mathsecr_api__chapter_tree, mcp__plugin_mathsecr_api__cell_tag_search, mcp__plugin_mathsecr_api__cell_search, mcp__plugin_mathsecr_api__cell_detail, mcp__plugin_mathsecr_api__similar_search_setting, mcp__plugin_mathsecr_api__similar_cells, mcp__plugin_mathsecr_api__mypaper_list, mcp__plugin_mathsecr_api__mypaper_detail, mcp__plugin_mathsecr_api__mypaper_cells, mcp__plugin_mathsecr_api__mypaper_scores, mcp__plugin_mathsecr_api__cell_image, mcp__plugin_mathsecr_api__cell_image_link
---

# 수학비서 근거 기반 분석

회원 본인의 자료를 읽어 분석한다. 저장된 분석지 조회는 `myanalysis`를 쓴다. 아래에서 필요한 레시피 하나만 읽는다. 본문 도구 약칭의 실제 이름은 `mcp__plugin_mathsecr_api__<도구명>`이다.

**도구 호출 전에 선택한 레시피 전문을 반드시 읽는다.** `paperId`가 있으면 `mypaper_detail` 응답을 기다리고, 완료 문제지의 `mypaper_cells`에서 저장 교육과정을 확인한 뒤에만 문항 상세·단원·검색을 시작한다. 이 단계들을 병렬 호출하지 않는다. 편집 중이거나 저장 교육과정 미상이면 종속 조회를 멈춘다.

| 요청 | 레시피 |
|---|---|
| 문제지 진단 | [paper-diagnosis.md](references/paper-diagnosis.md) |
| 두 문항 비교 | [question-compare.md](references/question-compare.md) |
| 대체 문항 추천 | [replacement-recommend.md](references/replacement-recommend.md) |

## 실행과 근거

- Desktop Chat에서는 스킬이 레시피를 직접 실행한다. Claude Code에서는 `math-analyst`에 선택한 **레시피 전문**, 요청·ID·교육과정·이미 읽은 필드와 출처·잘림 상태·이미지 사용 장수와 잔액·상한을 전달한다. 에이전트가 없으면 직접 실행한다. 에이전트에 파일 읽기 도구가 있다고 가정하지 않는다.
- 모든 사실·집계에 `「수학비서 데이터」 도구.필드, paperId/cellId/단원 ID`를 붙인다. 집계는 분모·범위·계산 방식을 적고, 권장·평가에는 **Claude 제안**이라고 명시하여 저장된 평가로 오인시키지 않는다. 이미지 판단은 `「이미지 판독(Claude 해석)」 cellId, part, 판독 주체`로 구분한다. 판독하지 않은 내용은 미확인이다.
- 태그 사전과 `cell_search.adaptiveTagIds` 검색 조건만 근거로 쓴다. **「문항별 부착 태그 확인 불가」**를 표시한다. `similar_cells` 결과에는 `hasPermission`/`isUnavailable`이 없으므로 채택 전에 `cell_detail`로 확인한다.
- 문제은행 문항의 빈 `sources`는 "문제은행"으로 적는다. `sourceless: true` 또는 출처 ID 0만 "출처 없음"이다. 누락·null 필드는 추정하지 않는다.

## 이미지 공통 규칙

표의 각 행에 근거 라벨·실제 필드·ID를 적고 생략하지 않는다. 보기 링크의 포함 항목·거절·만료·조회 횟수도 `「수학비서 데이터」 cell_image_link.entries/denied/expiresAt/maxViews + cellId`로 표시한다. 이미지 사용량은 `cell_image.returned/budgetRemainingToday` 또는 사용자 제공값이라고 출처를 붙인다. 오류·권한 차이·잔액 감소의 내부 원인은 추정하지 않는다.

- 메타데이터 질문은 저장 필드로 답한다. "보여줘"는 `cell_image_link`, "읽어/분석/비교"는 내용 확인이 꼭 필요할 때 `cell_image`다. 분석을 뒷받침할 소수 문항만 판독한다.
- 회원 공용 하루 300장. 요청 전체 기본 상한 **4장**, 꼭 필요한 비교는 이유를 밝히고 **8장 이내**로 확대할 수 있다. **8장 초과는 사용자에게 묻기 전 실행하지 않는다.** 위임 전 사용량도 차감한다. 호출 전 여유는 이번 호출이 돌려줄 수 있는 최대 장수로 계산한다(question 1장, 지문이 있으면 +1, 해설은 지정한 해설 ID 수, 기본 1장). 부족하면 중단하거나 위 한도 안에서 확대한다. 반환 `returned` 장수(지문·해설 포함)를 누적하며 **순차 호출**만 한다. 매 응답의 `budgetRemainingToday`를 읽는다. `budgetRemainingToday`가 이번 호출 예상 장수보다 작으면 멈춘다. 도구 오류로 잔액이 오지 않으면 직전 잔액에서 요청 장수를 뺀 값으로 보고 이어 간다(서버 0.7.2 이하는 실패도 차감할 수 있다). 직전 잔액이 없으면 멈춘다.
- `denied`의 문항·부분·사유를 알리고 자동 재시도하지 않는다. "이미지를 가져올 수 없습니다" 같은 오류도 재시도하지 않고 `cell_image_link`로 사용자가 볼 수 있다고 안내하며 **「수학비서 데이터」만으로 분석 계속**이라고 표시한다. 링크는 직접 열거나 다운로드하지 않는다.
- 최종 분석 주체(Code의 `math-analyst`, Chat의 스킬)가 근거로 쓸 이미지를 직접 가져온다. 앞선 판독을 전달받으면 판독자·출처를 보존하고 "전달받은 판독, 재판독하지 않음"이라 쓴다. 예산 부족이면 이를 새 판독으로 주장하지 않는다.
- `parts`는 `question`(기본)·`explanation`. 해설은 정답과 전체 풀이를 포함하므로 분석에 필요한 경우만 읽고, 사용자가 요청하지 않으면 정답 내용을 노출하지 않는다. 문항·정답·풀이의 전사와 LaTeX 변환은 하지 않는다.

읽기 전용이다. 문항 교체·출제·구매·파일 생성은 수학비서 웹에서 사용자가 한다. 학생 이름·답안·점수·명단은 요청하거나 다루지 않는다. 브라우저 자동화·WebFetch로 링크나 웹 작업을 대신하지 않는다. 인증 오류는 `/mathsecr:status` 재연결 안내를 따른다. 입력 형식/output schema 오류는 반복하지 않고 재연결을 안내하며 같은 대화에서 예전 정의의 입력을 더 쓰지 않는다.
