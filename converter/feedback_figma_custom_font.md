---
name: Figma 커스텀 폰트 적용 방법
description: use_figma는 클라우드 전용이라 로컬 폰트 불가 → Gothic A1 중간 폰트 + 플러그인 변환 방식으로 우회
type: feedback
---

use_figma로 커스텀 폰트(Pretendard, GmarketSans 등) 변경 시도하지 말 것. 클라우드 샌드박스에서 실행되어 로컬 폰트를 인식하지 못함.

**Why:** use_figma는 Remote(클라우드) 전용. listAvailableFontsAsync에 Google Fonts만 반환. Pretendard loadFontAsync 전부 실패 확인됨.

---

## 확정 워크플로우 전체

### HTML 구조 (figma-capture/index.html)

한 파일에 두 영역을 함께 작성한다.

```
[왼쪽] .capture-gothic   — Gothic A1   → Figma 캡처 대상
[오른쪽] .preview-pretendard — Pretendard → 브라우저 검수용 (캡처 제외)
```

**폰트 선언:**
```html
<!-- Gothic A1: Google Fonts (Figma 클라우드 인식 가능) -->
<link href="https://fonts.googleapis.com/css2?family=Gothic+A1:wght@100;200;300;400;500;600;700;800;900&display=swap" rel="stylesheet">

<!-- Pretendard: @font-face로 로컬 파일 참조 (검수용) -->
@font-face { font-family: 'Pretendard'; src: url('../figma-test/fonts/Pretendard-Light.otf') format('opentype'); font-weight: 300; }
/* ... 나머지 weight도 동일하게 선언 */
```

**CSS 폰트 지정:**
```css
.capture-gothic     * { font-family: 'Gothic A1', sans-serif; }
.preview-pretendard * { font-family: 'Pretendard', sans-serif; }
```

**Why Gothic A1인가:**
- Figma 클라우드 폰트 → 캡처 시 fontStyle(Bold/SemiBold 등)이 정확히 설정됨
- Pretendard와 weight variant 이름 완전 동일 (Thin~Black 9개) → 플러그인 1:1 변환 가능
- Noto Sans KR 사용 금지: 200/600/800 weight 없어 불완전 매핑 발생

---

### 1단계: 로컬 서버 실행

```bash
cd figma-capture && npx -y http-server . -p 8765 --cors -c-1
```

서버 확인: `curl -s -o /dev/null -w "%{http_code}" http://localhost:8765/index.html` → 200

---

### 2단계: 검수 (브라우저)

```
http://localhost:8765/index.html
```

hash 파라미터 없이 열면 캡처 없이 일반 페이지로 표시됨.
좌측 Gothic A1 / 우측 Pretendard를 나란히 비교하며 디자인 검수.

---

### 3단계: Figma 캡처

```
generate_figma_design(outputMode: "existingFile", fileKey: "...")
```

브라우저 열기 (figmaselector로 Gothic A1 영역만 지정):
```
http://localhost:8765/index.html
  #figmacapture={captureId}
  &figmaendpoint=https://mcp.figma.com/mcp/capture/{captureId}/submit
  &figmadelay=4000
  &figmaselector=.capture-gothic
```

- `figmadelay=4000`: Google Fonts 로딩 대기
- `figmaselector=.capture-gothic`: 검수용 Pretendard 영역 제외, Gothic A1만 전송

폴링: `generate_figma_design(captureId: "...")` → completed 될 때까지 반복

---

### 4단계: 플러그인으로 Pretendard 변환 (사용자 직접 실행)

Figma 데스크탑에서:
1. 캡처된 프레임 선택
2. **Plugins → Development → Pretendard Font Converter**
3. "Pretendard로 변환" 클릭
4. `N개 노드 수정 완료` 메시지 확인

플러그인 위치: `C:\Users\Jey\Desktop\claude-project\figma-plugin\manifest.json`
플러그인 재생성 방법: `figma-plugin/CLAUDE.md` 참조

**주의:** use_figma/figma-rest-api로 플러그인 원격 실행 불가. 반드시 사용자가 Figma 데스크탑에서 직접 실행해야 함.

---

### 5단계: 변환 결과 검증 (use_figma)

```javascript
// 텍스트 노드 fontName 전수 확인
const allPretendard = results.every(r => r.fontName.family === 'Pretendard')
// → { allPretendard: true, wrongNodes: [] } 확인
```

**[필수] 검증 항목:**
- [ ] `allPretendard: true` 확인
- [ ] weight별 variant(Light/Regular/SemiBold/Bold/ExtraBold 등) 올바르게 매핑됐는지 확인
- [ ] Figma 뷰 모드(더블클릭 없이)에서 Pretendard로 렌더링되는지 시각 확인

---

## 금지 사항 요약

| 시도 | 결과 | 이유 |
|------|------|------|
| use_figma로 Pretendard 적용 | 실패 | 클라우드에서 loadFontAsync 불가 |
| HTML에 Pretendard 직접 사용 후 캡처 | fontStyle 누락 | Figma 클라우드가 로컬 폰트 인식 못함 |
| Noto Sans KR 사용 | 불완전 매핑 | 200/600/800 weight 없음 |
| figma-rest-api로 폰트 수정 | 불가 | 텍스트 노드 font 속성 수정 API 없음 |
