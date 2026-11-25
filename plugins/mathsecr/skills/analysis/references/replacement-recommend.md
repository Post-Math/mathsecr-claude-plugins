# 대체 문항 추천

## 입력

기준 `cellId`, 선택 `paperId`, 교육과정, 유지/변경할 단원·난이도·정답 유형, 후보 수(기본 3), 이미지 사용량·잔액·상한. 실제 교체는 하지 않는다.

## 호출 순서와 상한

1. **`paperId`가 있으면 다른 문항 조회보다 먼저 `mypaper_detail(paperId)`** 1회. `progress: editing`이면 `mypaper_cells`/`mypaper_pages`를 부르지 않는다. 저장 교육과정을 확인할 수 없으므로 종속 조회를 멈추고 「기존 문항 중복 미확인」이라고 보고한다(이미 전달된 후보가 있으면 이 제목 아래만 표시).
2. 완료 문제지라면 필요시 `curriculum_list` 1회로 최초 요청 ID를 정하고 `mypaper_cells(paperId, curriculumId, isNumberSorted: true, limit: 20)` 최대 10페이지. **응답에 저장된 `curriculumId`를 이후 `cell_detail`, `chapter_tree`, `cell_search`, `similar_cells`, 이미지 조회 모두에 사용**한다. 요청 교육과정과 다르면 차이를 알린다. 페이지 커서는 최초 필터를 유지한다. 필터를 바꿀 때는 커서를 버리고 처음부터 시작하며 총 10페이지를 넘기지 않는다. 저장 ID 미상이면 종속 조회 중단. `hasMore`가 남거나 복구 불가 `nestedTruncated`면 이후 후보는 모두 **「기존 문항 중복 미확인」**으로 표시하고 확정 추천하지 않는다. 내부 잘림은 커서만으로 해결되지 않는다.
3. 문제지 없이 시작할 때만 요청/`curriculum_list` 교육과정을 쓴다. `cell_detail(cellId, curriculumId)` 1회로 기준 문항의 단원·난이도·정답 유형·권한 확인. `hasPermission`이 true가 아니거나 사용 불가/삭제면 기준 내용 활용 중단. `chapter_tree(curriculumId, keyword, depth: 4)` 최대 2회로 검색용 **tree_id**를 확인한다(`chapter_id`와 혼동 금지).
4. `cell_search(curriculumId, chapterTreeIds, minDifficulty, maxDifficulty, answerTypes, problemBankScope: "include", limit: 10)` 최대 2페이지. 기본은 동일 단원·동일 난이도·동일 답 유형. 응답 유형은 원문을 보존하되 `single_choice`/`multiple_choice`는 `choice`, `short_answer`/`long_answer`/`integer_answer`/`latex_answer`는 `answer`, `proof_answer`는 `proof`, `boolean_choice`/`multiple_boolean_choice`는 `boolean` 필터로 대응한다(필터는 묶음이므로 세부 유형 일치는 6단계에서 다시 대조한다). 모르는 유형·null 난이도는 추정하지 말고 조건 확인 후 진행한다. DB/연도 제한이 있으면 `exclude`로 검색하며 `include`/`only`와 섞지 않는다. 조건 완화는 이유·차이를 표시한다.
5. `similar_cells(cellId, curriculumId, relationType: "5depth", limit: 10, currentPaperId: paperId)` 1페이지(문제지 없으면 `currentPaperId` 생략. 이 값은 그 문제지에 든 문항을 결과에서 빼므로 대체 추천에만 쓴다). 비었을 때만 `4depth` 1페이지, **최대 2회**. `TX`가 적용되면 중단하고 텍스트 유사성이라고 알린다. 중복 후보와 기준 ID, 읽은 기존 문제지 ID를 제외한다. `currentPaperId`만으로 중복 검증 완료라고 말하지 않는다. `similar_cells`는 DB·연도 필터가 없고 `cell_detail`에도 DB·연도 필드가 없으므로, **DB/연도 제한이 있는 요청에서는 4단계 필터 검색 결과에도 나온 유사 후보만** 순위에 넣는다. 나머지는 「DB·연도 조건 미확인」으로 따로 적고 추천하지 않는다.
6. 후보 최대 6개를 `cell_detail` 각 1회로 확인한다. **유사 결과에 권한/사용 불가 필드가 없으므로 반드시 상세 확인 후 활용**한다. `hasPermission` true가 아니거나 `isUnavailable`/`isProblemBankRemoved` true면 제외. 단원·난이도·답 유형이 기준과 맞는지 다시 대조한다. `sourceless`/출처 ID 0은 "출처 없음", 문제은행의 빈 출처는 "문제은행"이다.
7. 상위 후보 최대 2개에 `cell_image(cellId, curriculumId, parts: ["question"])`로 내용 확인. 기준과 조건 차이를 직접 비교하려면 기준 이미지도 예산 안에서 최종 분석자가 읽는다. 읽지 못하면 조건·계산량은 미확인, 메타데이터 기반 순위만 제시한다. 태그 조건은 `cell_tag_search(keywords, limit: 10)` 1페이지와 검색 필터의 근거만 허용하며 **「문항별 부착 태그 확인 불가」**를 적는다.

## 이미지 예산과 중단

회원 공용 일 300장. 요청 누적 기본 4장, 위임 전 사용량 차감. 순차 호출만. 호출 전 여유는 이번 호출이 돌려줄 수 있는 최대 장수로 계산한다(question 1장, 지문이 있으면 +1, 해설은 지정한 해설 ID 수, 기본 1장), `returned` 장수로 정산한다. 필요한 비교는 이유를 밝히고 8장 이내까지, **8장 초과 전 사용자 확인**. `budgetRemainingToday`가 이번 호출 예상 장수보다 작으면 멈춘다. 이미지 호출은 병렬로 하지 않는다(동시 호출 중에는 다른 호출의 예약분이 잔액에서 잠시 빠져 보인다). 잔액은 가장 최근 응답의 `budgetRemainingToday`를 쓰고, 도구 오류로 잔액이 오지 않으면 직전 잔액에서 요청 장수를 뺀 값으로 이어 간다. 직전 잔액이 없으면 멈춘다. 실패한 부분은 차감되지 않으며 `denied`에 사유가 온다. 응답에 `fallback: "cell_image_link"`가 있으면 읽지 못한 그 부분은 사용자에게 보기 링크를 제안한다. `denied`의 ID·부분·사유를 보고하고 자동 재시도하지 않는다. "이미지를 가져올 수 없습니다" 오류는 `cell_image_link` 보기를 제안하고 수학비서 데이터만으로 계속한다. 최종 분석자가 근거 이미지를 직접 가져오며 앞선 판독은 판독자·출처, 재판독하지 않음을 표시한다. 해설 `explanation`은 필요할 때만, 요청 없이 정답을 공개하지 않고 문항·풀이·LaTeX 전사를 하지 않는다.

페이지·후보·이미지 상한에 도달하면 추가 호출 중단. 적합 후보 부족이면 그 수만 보고하며 부적합 후보로 채우지 않는다. 인증·입력/output schema 오류는 반복하지 않고 `/mathsecr:status` 안내. 학생 정보·쓰기·구매·브라우저 자동화 금지.

## 출력

| 순위(Claude 제안) | 후보 ID | 단원·난이도·정답 유형 일치 | 추천 이유/차이 | 권한·기존 문항 대조 | 근거 |
|---|---|---|---|---|---|
| 1 | ID | 저장값과 조건 | 메타데이터 근거 / 확인한 조건 차이 | 확인 범위 또는 기존 문항 중복 미확인 | 「수학비서 데이터」 도구.필드 + cellId/paperId; 「이미지 판독(Claude 해석)」 cellId + part + 판독자 |

모든 이유에 근거 라벨을 붙이고 순위는 수학비서의 추천 점수로 표현하지 않는다. 단순 ID 대조는 내용상 중복 없음 보장이 아니다. 읽은 문제지 범위·교육과정 차이·이미지 장수/잔액·미확인 항목을 밝힌다. 마지막은 "실제 문항 교체·구매는 수학비서 웹에서 진행해 주세요."
