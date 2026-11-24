# 수학비서 Claude 플러그인

수학비서 사용자를 위한 Claude Code · Claude Desktop 플러그인입니다. 플러그인을 설치하고 수학비서 계정으로 로그인하면 Claude 에서 수학비서 데이터를 볼 수 있습니다. 사용자 PC 에 따로 설치하거나 설정할 것은 없습니다.

## 설치 (Claude Code)

```
/plugin marketplace add Post-Math/mathsecr-claude-plugins
/plugin install mathsecr@postdemy
```

설치한 뒤 `/mcp` 에서 `plugin:mathsecr:api` 를 골라 **Authenticate** 를 누르면 브라우저에서 수학비서 로그인 화면이 열립니다. 로그인하고 접근을 허용하면 연결이 끝납니다. `/mathsecr:status` 로 연결된 계정을 확인할 수 있습니다.

## 설치 (Claude Desktop)

설정 → **Plugins** 에서 마켓플레이스 `Post-Math/mathsecr-claude-plugins` 를 추가하고 `mathsecr` 를 설치한 뒤, 커넥터에서 수학비서를 연결해 로그인합니다.

## 개인정보

- 로그인은 수학비서(mathsecr.com) 계정으로 하며, Claude 에는 수학비서 비밀번호나 로그인 토큰이 전달되지 않습니다.
- 지금 제공하는 기능은 읽기 전용입니다. 학생 개인정보는 제공하지 않습니다.
