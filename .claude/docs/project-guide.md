# 프로젝트 가이드

## 커맨드

```bash
npm run dev      # 개발 서버 (Node.js v22.12+ 필요, nvm use node)
npm run build    # tsc + vite 빌드
npm run lint     # ESLint
npm run preview  # 빌드 결과물 미리보기
```

## 아키텍처

### Provider 구조

`src/main.tsx` → `src/App.tsx` 순으로 진입하며, App에서 모든 Provider를 감싼다.

```
App
└── QueryClientProvider       # TanStack Query
    └── RouterProvider        # React Router v7
        └── RootLayout        # src/layouts/RootLayout.tsx
            └── <Outlet />    # 각 페이지
```

### 폴더 역할

- `src/api/` — Axios 인스턴스(`client.ts`)와 QueryClient 설정(`queryClient.ts`). API 함수는 도메인별 파일로 추가.
- `src/stores/` — Zustand 스토어. 파일명은 `use<Name>Store.ts` 형식.
- `src/pages/` — 라우트에 대응하는 페이지 컴포넌트.
- `src/layouts/` — Outlet을 포함하는 레이아웃 컴포넌트.
- `src/types/` — 공통 타입 정의. `ApiResponse<T>`, `PaginatedResponse<T>` 기본 제공.
- `src/hooks/` — 커스텀 훅.
- `src/components/` — 재사용 공통 컴포넌트.

### 경로 별칭

`@/` → `src/` 로 매핑. `tsconfig.app.json`과 `vite.config.ts` 양쪽에 설정.

## 라우트 추가

1. `src/pages/`에 페이지 컴포넌트 생성
2. `src/router.tsx`에 등록

레이아웃이 다른 페이지 그룹은 `src/layouts/`에 새 레이아웃을 추가하고 router에서 별도 children으로 묶는다.

## API

`src/api/client.ts` — `VITE_API_BASE_URL` 환경변수로 base URL 설정. 요청 시 `localStorage`의 `token`을 Authorization 헤더에 자동 주입, 401 응답 시 토큰 자동 삭제.

도메인별 파일을 `src/api/`에 추가하여 사용한다.

```ts
// src/api/posts.ts
import apiClient from '@/api/client';
export const getPosts = () =>
    apiClient.get('/posts').then((res) => res.data);

// 컴포넌트
const { data, isLoading } = useQuery({ queryKey: ['posts'], queryFn: getPosts });
```

## 상태관리

서버 데이터는 TanStack Query, 클라이언트 전역 상태(UI 상태, 세션 등)는 Zustand로 관리한다.

```ts
// src/stores/useUserStore.ts
import { create } from 'zustand';
const useUserStore = create<UserState>((set) => ({
    user: null,
    setUser: (user) => set({ user }),
}));
```

## 스타일링

스타터킷은 스타일링 도구를 포함하지 않는다. 프로젝트에 맞게 선택한다.

- **Tailwind CSS v4**: `npm install tailwindcss @tailwindcss/vite` → vite.config에 플러그인 추가 → `@import "tailwindcss"` in index.css
- **SCSS**: `npm install -D sass` → `.scss` 파일 생성 후 import
- **Plain CSS**: 별도 설치 없이 `.css` 파일 생성 후 사용
