# Pretendard Font Converter — Figma 플러그인 가이드

## 목적

HTML 캡처(`generate_figma_design`)로 Figma에 삽입된 텍스트 노드의 폰트를 **Pretendard로 일괄 변환**하는 로컬 Figma 플러그인.

- 선택 프레임 또는 현재 페이지 전체의 모든 텍스트 노드를 대상으로 함
- 기존 font-weight를 유지하여 Pretendard의 대응 variant로 1:1 매핑
- Mixed 폰트(노드 내 구간별 다른 폰트) 처리 포함

**전제 조건:** Pretendard가 Windows 시스템 폰트로 설치되어 있어야 함.

---

## 파일 구조

```
figma-plugin/
  manifest.json       ← Figma 플러그인 메타데이터
  src/
    code.ts           ← 플러그인 로직 (Figma 메인 스레드)
    ui.html           ← 플러그인 UI
  dist/
    code.js           ← 빌드 결과물 (manifest에서 참조)
  package.json
  tsconfig.json
```

---

## 생성 방법 (신규 프로젝트에서 재생성 시)

### 1. 디렉토리 및 파일 생성

```bash
mkdir -p figma-plugin/src figma-plugin/dist
```

### 2. manifest.json

```json
{
    "name": "Pretendard Font Converter",
    "id": "pretendard-font-converter-001",
    "api": "1.0.0",
    "main": "dist/code.js",
    "ui": "src/ui.html",
    "editorType": ["figma"]
}
```

### 3. package.json

```json
{
    "name": "pretendard-font-converter",
    "version": "1.0.0",
    "scripts": {
        "build": "esbuild src/code.ts --bundle --outfile=dist/code.js --target=es6",
        "watch": "esbuild src/code.ts --bundle --outfile=dist/code.js --target=es6 --watch"
    },
    "devDependencies": {
        "@figma/plugin-typings": "^1.0.0",
        "esbuild": "^0.24.0",
        "typescript": "^5.7.0"
    }
}
```

### 4. tsconfig.json

```json
{
    "compilerOptions": {
        "target": "ES6",
        "lib": ["ES6"],
        "strict": true,
        "moduleResolution": "bundler",
        "typeRoots": ["./node_modules/@figma/plugin-typings"]
    },
    "include": ["src/**/*"]
}
```

### 5. src/code.ts

```typescript
// Pretendard Font Converter
// 선택 영역(또는 현재 페이지)의 모든 텍스트를 Pretendard로 일괄 변환
// 기존 font-weight를 유지하여 Pretendard의 대응 variant로 매핑

const STYLE_TO_WEIGHT: Record<string, number> = {
    'thin': 100,
    'extralight': 200, 'extra light': 200, 'ultralight': 200, 'ultra light': 200,
    'light': 300,
    'regular': 400, 'normal': 400, 'roman': 400, 'text': 400, 'book': 400,
    'medium': 500,
    'semibold': 600, 'semi bold': 600, 'demibold': 600, 'demi bold': 600,
    'bold': 700,
    'extrabold': 800, 'extra bold': 800, 'ultrabold': 800, 'ultra bold': 800,
    'black': 900, 'heavy': 900,
}

const WEIGHT_TO_PRETENDARD: Record<number, string> = {
    100: 'Thin',
    200: 'ExtraLight',
    300: 'Light',
    400: 'Regular',
    500: 'Medium',
    600: 'SemiBold',
    700: 'Bold',
    800: 'ExtraBold',
    900: 'Black',
}

function toPretendardStyle(style: string): string {
    const normalized = style.toLowerCase().replace(/\s*(italic|oblique)\s*/g, '').trim()
    const weight = STYLE_TO_WEIGHT[normalized] ?? 400
    const weights = Object.keys(WEIGHT_TO_PRETENDARD).map(Number)
    const closest = weights.reduce((a, b) => Math.abs(b - weight) < Math.abs(a - weight) ? b : a)
    return WEIGHT_TO_PRETENDARD[closest]
}

function collectTextNodes(node: BaseNode): TextNode[] {
    if (node.type === 'TEXT') return [node as TextNode]
    if (!('children' in node)) return []
    const results: TextNode[] = []
    for (const child of (node as ChildrenMixin).children) {
        results.push(...collectTextNodes(child))
    }
    return results
}

figma.showUI(__html__, { width: 320, height: 220 })

figma.ui.onmessage = async (msg: { type: string }) => {
    if (msg.type !== 'convert-fonts') return

    const roots: readonly BaseNode[] =
        figma.currentPage.selection.length > 0
            ? figma.currentPage.selection
            : figma.currentPage.children

    const textNodes: TextNode[] = []
    for (const root of roots) {
        textNodes.push(...collectTextNodes(root))
    }

    let fixedCount = 0
    const errors: string[] = []

    for (const node of textNodes) {
        try {
            if (node.fontName === figma.mixed) {
                const len = node.characters.length
                let i = 0
                while (i < len) {
                    const rangeFont = node.getRangeFontName(i, i + 1) as FontName
                    let j = i + 1
                    while (j < len) {
                        const nextFont = node.getRangeFontName(j, j + 1) as FontName
                        if (nextFont.family !== rangeFont.family || nextFont.style !== rangeFont.style) break
                        j++
                    }
                    const pretendardStyle = toPretendardStyle(rangeFont.style)
                    await figma.loadFontAsync({ family: 'Pretendard', style: pretendardStyle })
                    node.setRangeFontName(i, j, { family: 'Pretendard', style: pretendardStyle })
                    fixedCount++
                    i = j
                }
            } else {
                const fontName = node.fontName as FontName
                const pretendardStyle = toPretendardStyle(fontName.style)
                await figma.loadFontAsync({ family: 'Pretendard', style: pretendardStyle })
                node.fontName = { family: 'Pretendard', style: pretendardStyle }
                fixedCount++
            }
        } catch (e) {
            errors.push(`"${node.name}": ${String(e)}`)
        }
    }

    figma.ui.postMessage({ type: 'result', fixedCount, totalScanned: textNodes.length, errors })
}
```

### 6. src/ui.html

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; font-size: 13px; background: #fff; }
        .container { padding: 16px; display: flex; flex-direction: column; gap: 12px; }
        .header-title { font-weight: 600; font-size: 14px; color: #111; }
        .header-desc { font-size: 12px; color: #888; margin-top: 4px; line-height: 1.5; }
        button { background: #18A0FB; color: #fff; border: none; border-radius: 6px; padding: 10px 16px; font-size: 13px; font-weight: 600; cursor: pointer; width: 100%; }
        button:hover { background: #0D8CE8; }
        button:disabled { background: #B3B3B3; cursor: not-allowed; }
        .status { font-size: 12px; line-height: 1.6; min-height: 18px; }
        .success { color: #1B8F54; }
        .error-item { color: #E03E3E; }
        .divider { border: none; border-top: 1px solid #f0f0f0; }
    </style>
</head>
<body>
    <div class="container">
        <div>
            <p class="header-title">Pretendard Font Converter</p>
            <p class="header-desc">모든 텍스트를 Pretendard로 일괄 변환합니다.<br>기존 weight를 유지하여 대응 variant로 매핑합니다.<br>선택 없으면 현재 페이지 전체를 변환합니다.</p>
        </div>
        <hr class="divider">
        <button id="btn">Pretendard로 변환</button>
        <div id="status" class="status"></div>
    </div>
    <script>
        const btn = document.getElementById('btn')
        const statusEl = document.getElementById('status')
        btn.addEventListener('click', () => {
            btn.disabled = true
            btn.textContent = '처리 중...'
            statusEl.textContent = ''
            parent.postMessage({ pluginMessage: { type: 'convert-fonts' } }, '*')
        })
        window.onmessage = (event) => {
            const msg = event.data.pluginMessage
            if (!msg || msg.type !== 'result') return
            btn.disabled = false
            btn.textContent = 'Pretendard로 변환'
            if (msg.errors.length === 0) {
                statusEl.innerHTML = `<span class="success">✓ ${msg.fixedCount}개 노드 수정 완료</span><br><span style="color:#aaa">(스캔: ${msg.totalScanned}개)</span>`
            } else {
                statusEl.innerHTML = `<span class="success">✓ ${msg.fixedCount}개 수정</span><br>` +
                    msg.errors.map(e => `<span class="error-item">✗ ${e}</span>`).join('<br>')
            }
        }
    </script>
</body>
</html>
```

### 7. 빌드

```bash
cd figma-plugin
npm install
npm run build
# → dist/code.js 생성됨
```

---

## Figma 설치 방법

1. Figma 데스크탑 앱 → 메뉴 → **Plugins → Development → Import plugin from manifest**
2. `figma-plugin/manifest.json` 선택
3. 이후 **Plugins → Development → Pretendard Font Converter** 로 실행

코드 수정 후 `npm run build` 재실행 시 Figma에서 자동으로 최신 빌드 반영 (재설치 불필요).

---

## weight 매핑 표

| 기존 style 이름 | → | Pretendard style |
|----------------|---|-----------------|
| Thin | → | Thin |
| ExtraLight / Extra Light / UltraLight | → | ExtraLight |
| Light | → | Light |
| Regular / Normal / Roman / Book | → | Regular |
| Medium | → | Medium |
| SemiBold / Semi Bold / DemiBold | → | SemiBold |
| Bold | → | Bold |
| ExtraBold / Extra Bold / UltraBold | → | ExtraBold |
| Black / Heavy | → | Black |
| Italic / Oblique 포함 시 | → | italic 제거 후 weight만 추출 |
| 매핑 불가 시 | → | Regular (기본값) |

---

## 주의사항

- `use_figma`(클라우드)로는 Pretendard `loadFontAsync` 불가 → 반드시 Figma 데스크탑에서 로컬 플러그인으로 실행
- Pretendard가 Windows 시스템에 설치되어 있지 않으면 플러그인 실행 시 에러 발생
- Gothic A1 → Pretendard 변환 시 weight 이름이 완전 동일하므로 손실 없이 1:1 변환됨
