---
description: 수학비서 연결 상태를 확인한다 (어떤 계정으로 연결됐는지)
allowed-tools: mcp__plugin_mathsecr_api__mathsecr_whoami
---

`mathsecr_whoami` 도구를 불러 지금 연결된 수학비서 계정을 확인하고, 결과를 한국어로 한두 문장으로 알려 줘.

- 성공하면: "수학비서에 **{이름}** 님 계정으로 연결돼 있어요." 처럼 이름만 말한다. 회원 ID 는 묻지 않으면 말하지 않는다.
- 도구가 없거나 인증이 필요하다는 오류면: Claude Code 에서는 `/mcp` 를 열어 `plugin:mathsecr:api` 서버의 인증(Authenticate)을 진행하라고 안내하고, Claude Desktop 에서는 설정 → 커넥터에서 수학비서를 연결하라고 안내한다.
- "연결이 끊겼습니다" 류 오류면: 수학비서 로그인이 만료됐다는 뜻이니 같은 방법으로 다시 연결하라고 안내한다.
