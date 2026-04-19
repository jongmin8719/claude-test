# MCP 서버 설정 가이드

타 PC에서 이 프로젝트의 Figma 워크플로우를 실행하기 위한 MCP 설정 방법.

---

## 사용 중인 MCP 서버 구성

| 서버 | 종류 | 용도 | 설정 위치 |
|------|------|------|----------|
| `figma` | Remote (Figma 공식) | 디자인 읽기/캡처/생성, FigJam | Claude Code MCP 전역 설치 |
| `figma-rest-api` | Local (커스텀 빌드) | 댓글, 컴포넌트, 스타일, 웹훅 등 REST API | `.mcp.json` |

---

## 1. Figma API 토큰 발급

두 서버 모두 Figma 토큰이 필요하다.

1. Figma 웹 → 우상단 프로필 → **Settings**
2. **Security** 탭 → **Personal access tokens** → **Generate new token**
3. 토큰 이름 입력 후 생성 → 값 복사 (재확인 불가)
4. 토큰 형식: `figd_xxxxxxxxxxxxxxxx`

---

## 2. figma (공식 Remote MCP) 설정

npm으로 전역 설치한다. `.mcp.json`으로 관리하지 않는다.

```bash
claude mcp add --transport http figma https://mcp.figma.com/sse
```

### 인증

설치 후 Claude Code 재시작 → `mcp__figma__whoami` 도구로 인증 상태 확인:
```
인증되지 않은 경우 → mcp__figma__authenticate 실행 → 브라우저에서 Figma 로그인
```

---

## 3. figma-rest-api (커스텀 로컬 MCP) 설정

### 3-1. Node.js 설치 확인

```bash
node -v   # v18 이상 권장
npm -v
```

### 3-2. 의존성 설치 및 빌드

```bash
cd mcp-servers/figma-rest-api
npm install
npm run build
# → dist/index.js 생성 확인
```

---

## 4. .mcp.json 설정 (figma-rest-api만 등록)

프로젝트 루트의 `.mcp.json`에 `figma-rest-api`만 등록한다.  
경로는 **해당 PC의 실제 절대 경로**로 변경할 것.

**예시 (Windows):**
```json
{
  "mcpServers": {
    "figma-rest-api": {
      "command": "node",
      "args": ["C:/Users/{사용자명}/Desktop/claude-project/mcp-servers/figma-rest-api/dist/index.js"],
      "env": {
        "FIGMA_TOKEN": "figd_xxxxxxxxxxxxxxxxxxxx"
      }
    }
  }
}
```

**예시 (macOS/Linux):**
```json
{
  "mcpServers": {
    "figma-rest-api": {
      "command": "node",
      "args": ["/Users/{사용자명}/Desktop/claude-project/mcp-servers/figma-rest-api/dist/index.js"],
      "env": {
        "FIGMA_TOKEN": "figd_xxxxxxxxxxxxxxxxxxxx"
      }
    }
  }
}
```

---

## 5. 연결 확인

Claude Code 재시작 후 두 서버가 모두 연결됐는지 확인한다.

```
# figma 공식 서버
mcp__figma__whoami
→ { email, handle, plans } 반환되면 정상

# figma-rest-api 커스텀 서버
mcp__figma-rest-api__figma_get_me
→ { id, email, handle } 반환되면 정상
```

---

## 6. 추가 설정 (Figma 워크플로우 전용)

### Pretendard 폰트 시스템 설치

Figma 데스크탑에서 커스텀 폰트를 사용하려면 시스템 설치가 필요하다.

- `figma-test/fonts/` 내 `.otf` 파일 전체 선택
- 우클릭 → **모든 사용자용으로 설치** (Windows)
- 또는 Font Book에 드래그 (macOS)

### Pretendard Font Converter 플러그인 설치

- Figma 데스크탑 → **Plugins → Development → Import plugin from manifest**
- `figma-plugin/manifest.json` 선택
- 플러그인이 없는 경우 `figma-plugin/CLAUDE.md` 참조하여 재생성

---

## 7. 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| `figma-rest-api` 연결 안 됨 | `dist/index.js` 없음 | `npm run build` 실행 |
| `figma-rest-api` 연결 안 됨 | 절대 경로 오류 | `.mcp.json`의 args 경로 확인 |
| `figma-rest-api` 401 에러 | 토큰 만료/오류 | Figma에서 토큰 재발급 후 `.mcp.json` 업데이트 |
| `figma` 인증 안 됨 | 로그인 필요 | `mcp__figma__authenticate` 실행 |
| `figma_get_local_variables` 403 | Enterprise 전용 API | `get_design_context`로 대체 |
