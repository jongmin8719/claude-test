# figma-rest-api MCP 서버 — 생성 가이드

Figma REST API를 Claude Code에서 사용하기 위한 커스텀 MCP 서버.
이 파일만 있으면 소스코드 없이도 서버를 완전히 재생성할 수 있다.

---

## 파일 구조

```
mcp-servers/figma-rest-api/
  src/
    index.ts          ← MCP 서버 진입점 (tool 정의 + handler)
    figma-client.ts   ← Figma REST API 클라이언트
  dist/
    index.js          ← 빌드 결과물 (npm run build로 생성)
  package.json
  tsconfig.json
```

---

## 생성 방법

### 1. package.json

```json
{
    "name": "figma-rest-api-mcp",
    "version": "1.0.0",
    "description": "Figma REST API MCP Server",
    "type": "module",
    "main": "dist/index.js",
    "scripts": {
        "build": "tsc",
        "start": "node dist/index.js",
        "dev": "tsx src/index.ts"
    },
    "dependencies": {
        "@modelcontextprotocol/sdk": "^1.10.2",
        "axios": "^1.7.9"
    },
    "devDependencies": {
        "@types/node": "^22.10.7",
        "tsx": "^4.19.2",
        "typescript": "^5.7.3"
    }
}
```

### 2. tsconfig.json

```json
{
    "compilerOptions": {
        "target": "ES2022",
        "module": "ES2022",
        "moduleResolution": "bundler",
        "outDir": "./dist",
        "strict": true,
        "esModuleInterop": true,
        "skipLibCheck": true
    },
    "include": ["src/**/*"]
}
```

### 3. src/figma-client.ts

```typescript
import axios, { AxiosInstance } from "axios";

export class FigmaClient {
  private client: AxiosInstance;

  constructor(token: string) {
    this.client = axios.create({
      baseURL: "https://api.figma.com/v1",
      headers: {
        "X-Figma-Token": token,
      },
    });
  }

  // ─── Users ───────────────────────────────────────────────────────────────

  async getMe() {
    const res = await this.client.get("/me");
    return res.data;
  }

  // ─── Files ───────────────────────────────────────────────────────────────

  async getFile(fileKey: string, params?: {
    version?: string; ids?: string; depth?: number;
    geometry?: string; plugin_data?: string; branch_data?: boolean;
  }) {
    const res = await this.client.get(`/files/${fileKey}`, { params });
    return res.data;
  }

  async getFileNodes(fileKey: string, ids: string, params?: {
    version?: string; depth?: number; geometry?: string; plugin_data?: string;
  }) {
    const res = await this.client.get(`/files/${fileKey}/nodes`, { params: { ids, ...params } });
    return res.data;
  }

  async getImages(fileKey: string, ids: string, params?: {
    scale?: number; format?: "jpg" | "png" | "svg" | "pdf";
    svg_include_id?: boolean; svg_simplify_stroke?: boolean;
    use_absolute_bounds?: boolean; version?: string;
  }) {
    const res = await this.client.get(`/images/${fileKey}`, { params: { ids, ...params } });
    return res.data;
  }

  async getImageFills(fileKey: string) {
    const res = await this.client.get(`/files/${fileKey}/images`);
    return res.data;
  }

  // ─── Comments ─────────────────────────────────────────────────────────────

  async getComments(fileKey: string, params?: { as_md?: boolean }) {
    const res = await this.client.get(`/files/${fileKey}/comments`, { params });
    return res.data;
  }

  async postComment(fileKey: string, body: {
    message: string; client_meta?: object; comment_id?: string;
  }) {
    const res = await this.client.post(`/files/${fileKey}/comments`, body);
    return res.data;
  }

  async deleteComment(fileKey: string, commentId: string) {
    const res = await this.client.delete(`/files/${fileKey}/comments/${commentId}`);
    return res.data;
  }

  async getCommentReactions(fileKey: string, commentId: string) {
    const res = await this.client.get(`/files/${fileKey}/comments/${commentId}/reactions`);
    return res.data;
  }

  async postCommentReaction(fileKey: string, commentId: string, emoji: string) {
    const res = await this.client.post(`/files/${fileKey}/comments/${commentId}/reactions`, { emoji });
    return res.data;
  }

  async deleteCommentReaction(fileKey: string, commentId: string, emoji: string) {
    const res = await this.client.delete(`/files/${fileKey}/comments/${commentId}/reactions`, { params: { emoji } });
    return res.data;
  }

  // ─── Versions ─────────────────────────────────────────────────────────────

  async getFileVersions(fileKey: string) {
    const res = await this.client.get(`/files/${fileKey}/versions`);
    return res.data;
  }

  // ─── Projects ─────────────────────────────────────────────────────────────

  async getTeamProjects(teamId: string) {
    const res = await this.client.get(`/teams/${teamId}/projects`);
    return res.data;
  }

  async getProjectFiles(projectId: string, params?: { branch_data?: boolean }) {
    const res = await this.client.get(`/projects/${projectId}/files`, { params });
    return res.data;
  }

  // ─── Components ───────────────────────────────────────────────────────────

  async getTeamComponents(teamId: string, params?: { page_size?: number; after?: number; before?: number }) {
    const res = await this.client.get(`/teams/${teamId}/components`, { params });
    return res.data;
  }

  async getFileComponents(fileKey: string) {
    const res = await this.client.get(`/files/${fileKey}/components`);
    return res.data;
  }

  async getComponent(componentKey: string) {
    const res = await this.client.get(`/components/${componentKey}`);
    return res.data;
  }

  // ─── Component Sets ────────────────────────────────────────────────────────

  async getTeamComponentSets(teamId: string, params?: { page_size?: number; after?: number; before?: number }) {
    const res = await this.client.get(`/teams/${teamId}/component_sets`, { params });
    return res.data;
  }

  async getFileComponentSets(fileKey: string) {
    const res = await this.client.get(`/files/${fileKey}/component_sets`);
    return res.data;
  }

  async getComponentSet(componentSetKey: string) {
    const res = await this.client.get(`/component_sets/${componentSetKey}`);
    return res.data;
  }

  // ─── Styles ───────────────────────────────────────────────────────────────

  async getTeamStyles(teamId: string, params?: { page_size?: number; after?: number; before?: number }) {
    const res = await this.client.get(`/teams/${teamId}/styles`, { params });
    return res.data;
  }

  async getFileStyles(fileKey: string) {
    const res = await this.client.get(`/files/${fileKey}/styles`);
    return res.data;
  }

  async getStyle(styleKey: string) {
    const res = await this.client.get(`/styles/${styleKey}`);
    return res.data;
  }

  // ─── Variables ────────────────────────────────────────────────────────────

  async getLocalVariables(fileKey: string) {
    const res = await this.client.get(`/files/${fileKey}/variables/local`);
    return res.data;
  }

  async getPublishedVariables(fileKey: string) {
    const res = await this.client.get(`/files/${fileKey}/variables/published`);
    return res.data;
  }

  async postVariables(fileKey: string, body: object) {
    const res = await this.client.post(`/files/${fileKey}/variables`, body);
    return res.data;
  }

  // ─── Webhooks ─────────────────────────────────────────────────────────────

  async getWebhooks(teamId: string) {
    const res = await this.client.get(`/teams/${teamId}/webhooks`);
    return res.data;
  }

  async getWebhook(webhookId: string) {
    const res = await this.client.get(`/webhooks/${webhookId}`);
    return res.data;
  }

  async createWebhook(body: {
    event_type: string; team_id: string; endpoint: string;
    passcode: string; status?: string; description?: string;
  }) {
    const res = await this.client.post("/webhooks", body);
    return res.data;
  }

  async updateWebhook(webhookId: string, body: {
    event_type?: string; endpoint?: string; passcode?: string;
    status?: string; description?: string;
  }) {
    const res = await this.client.put(`/webhooks/${webhookId}`, body);
    return res.data;
  }

  async deleteWebhook(webhookId: string) {
    const res = await this.client.delete(`/webhooks/${webhookId}`);
    return res.data;
  }

  async getWebhookRequests(webhookId: string) {
    const res = await this.client.get(`/webhooks/${webhookId}/requests`);
    return res.data;
  }

  // ─── Dev Resources ────────────────────────────────────────────────────────

  async getDevResources(fileKey: string, params?: { node_ids?: string }) {
    const res = await this.client.get(`/files/${fileKey}/dev_resources`, { params });
    return res.data;
  }

  async postDevResources(fileKey: string, devResources: object[]) {
    const res = await this.client.post(`/files/${fileKey}/dev_resources`, { dev_resources: devResources });
    return res.data;
  }

  async putDevResources(fileKey: string, devResources: object[]) {
    const res = await this.client.put(`/files/${fileKey}/dev_resources`, { dev_resources: devResources });
    return res.data;
  }

  async deleteDevResource(fileKey: string, devResourceId: string) {
    const res = await this.client.delete(`/files/${fileKey}/dev_resources/${devResourceId}`);
    return res.data;
  }

  // ─── Activity Logs ────────────────────────────────────────────────────────

  async getActivityLogs(params?: {
    events?: string; start_time?: number; end_time?: number;
    limit?: number; order?: "asc" | "desc";
  }) {
    const res = await this.client.get("/activity_logs", { params });
    return res.data;
  }

  // ─── Library Analytics ────────────────────────────────────────────────────

  async getLibraryAnalyticsComponentActions(fileKey: string, params: {
    cursor?: string; group_by: "component" | "team" | "file_key";
    start_date?: string; end_date?: string; order?: "asc" | "desc";
  }) {
    const res = await this.client.get(`/files/${fileKey}/library_analytics/component/actions`, { params });
    return res.data;
  }

  async getLibraryAnalyticsComponentUsages(fileKey: string, params: {
    cursor?: string; group_by: "component" | "team" | "file_key"; order?: "asc" | "desc";
  }) {
    const res = await this.client.get(`/files/${fileKey}/library_analytics/component/usages`, { params });
    return res.data;
  }

  async getLibraryAnalyticsStyleActions(fileKey: string, params: {
    cursor?: string; group_by: "style" | "team" | "file_key";
    start_date?: string; end_date?: string; order?: "asc" | "desc";
  }) {
    const res = await this.client.get(`/files/${fileKey}/library_analytics/style/actions`, { params });
    return res.data;
  }

  async getLibraryAnalyticsStyleUsages(fileKey: string, params: {
    cursor?: string; group_by: "style" | "team" | "file_key"; order?: "asc" | "desc";
  }) {
    const res = await this.client.get(`/files/${fileKey}/library_analytics/style/usages`, { params });
    return res.data;
  }

  async getLibraryAnalyticsVariableActions(fileKey: string, params: {
    cursor?: string; start_date?: string; end_date?: string; order?: "asc" | "desc";
  }) {
    const res = await this.client.get(`/files/${fileKey}/library_analytics/variable/actions`, { params });
    return res.data;
  }

  async getLibraryAnalyticsVariableUsages(fileKey: string, params?: {
    cursor?: string; order?: "asc" | "desc";
  }) {
    const res = await this.client.get(`/files/${fileKey}/library_analytics/variable/usages`, { params });
    return res.data;
  }
}
```

### 4. src/index.ts

전체 소스는 길어서 tool 정의와 handler로 나눠 기술한다.

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { CallToolRequestSchema, ListToolsRequestSchema } from "@modelcontextprotocol/sdk/types.js";
import { FigmaClient } from "./figma-client.js";

const FIGMA_TOKEN = process.env.FIGMA_TOKEN;
if (!FIGMA_TOKEN) { console.error("FIGMA_TOKEN environment variable is required"); process.exit(1); }

const figma = new FigmaClient(FIGMA_TOKEN);
const server = new Server(
  { name: "figma-rest-api", version: "1.0.0" },
  { capabilities: { tools: {} } }
);
```

tool 정의 및 handler는 기존 `src/index.ts` 참조. 또는 아래 규칙으로 재작성:
- 각 Figma REST API 엔드포인트에 대응하는 tool을 `ListToolsRequestSchema` handler에 등록
- `CallToolRequestSchema` handler에서 tool name으로 분기하여 FigmaClient 메서드 호출
- 결과는 `{ content: [{ type: "text", text: JSON.stringify(result, null, 2) }] }` 형식으로 반환

### 5. 빌드

```bash
cd mcp-servers/figma-rest-api
npm install
npm run build
# → dist/index.js 생성 확인
```

---

## .mcp.json 연결

빌드 완료 후 프로젝트 루트 `.mcp.json`에 등록:

```json
{
  "mcpServers": {
    "figma-rest-api": {
      "command": "node",
      "args": ["{프로젝트 절대경로}/mcp-servers/figma-rest-api/dist/index.js"],
      "env": {
        "FIGMA_TOKEN": "{Figma Personal Access Token}"
      }
    }
  }
}
```

상세 설정 방법은 `mcp-servers/SETUP.md` 참조.
