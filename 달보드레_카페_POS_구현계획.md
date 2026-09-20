# 달보드레 카페 POS 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 중학교 특수학급 카페 '달보드레'에서 학생이 주문을 받고, 음료를 만들고, 손님을 호출하며, 선생님이 메뉴·재료·기록을 직접 관리하는 단일 파일 웹앱을 만든다.

**Architecture:** 빌드 도구 없는 단일 `index.html`. 내부는 `CONFIG → 순수 로직 → 백엔드 어댑터 → 화면` 순으로 구획하고, 각 구획을 한국어 주석 배너로 구분한다. 백엔드 어댑터는 6개 메서드로 된 하나의 인터페이스를 로컬(localStorage)과 Firestore 두 구현이 공유한다. 화면은 기기 역할(주문/제조/호출/관리)에 따라 하나만 그려진다.

**Tech Stack:** Vanilla JS (ES modules via CDN), Firebase Firestore 12.x (CDN), Web Speech API, Canvas API, GitHub Pages

**Spec:** `달보드레_카페_POS_PRD.md`

**참고 구현:** `../304 Ramen/index.html` — 같은 사람이 만든 축제 POS. 백엔드 어댑터 이중화, 트랜잭션 발번, `onSnapshot` 구독, 역할 선택, 모달·토스트 패턴을 여기서 가져온다. 결제·거스름돈 관련 코드는 가져오지 않는다.

## 테스트 전략 (중요 — 먼저 읽을 것)

PRD가 **단일 `index.html` · 빌드 도구 없음**을 요구하므로 pytest·vitest 같은 외부 러너를 쓸 수 없다. 대신:

| 대상 | 방법 | 실행 |
|---|---|---|
| 순수 로직 (계산·판정·변환) | `index.html` 안에 내장한 자가 테스트 | `http://localhost:8320/?selftest=1` → 화면에 `PASS n/n` |
| 화면·조작 | 브라우저에서 직접 확인 | 각 태스크의 "확인" 단계에 무엇을 보면 성공인지 명시 |
| Firestore 동기화 | 탭 2개를 띄워 상호 반영 확인 | Task 11 |

자가 테스트 코드는 배포 파일에 함께 남는다. `?selftest=1`이 없으면 실행되지 않으므로 운영에 영향이 없고, 선생님이 나중에 앱이 멀쩡한지 확인할 때도 쓸 수 있다.

**로컬 서버 띄우기 (모든 태스크에서 동일):**

```bash
python -m http.server 8320
```

## Global Constraints

아래는 PRD에서 그대로 옮긴 프로젝트 전역 제약이다. 모든 태스크의 요구사항에 암묵적으로 포함된다.

- **단일 `index.html`** — 빌드 도구 없음. Vanilla JS. 외부 라이브러리는 CDN only
- **코드 주석은 한국어**
- 화면 하단 푸터: **`Made by 정우쌤`**
- Firebase 인스턴스 이름: **`dallbodrae-pos`** (다른 앱과 캐시가 섞이지 않게)
- 데이터 경로: **`dallbodrae_pos/{spaceId}/`** 아래에만 저장. 초기화도 이 경로 안에서만 동작
- 사진: **400×400 JPEG, 품질 0.7**, data URL로 Firestore 문서에 저장. 문서 1MB 제한을 넘지 않게 검사
- 재료 차감은 **`increment()` 원자적 연산**, 주문번호 발번은 **트랜잭션**
- 문서 전체 덮어쓰기 금지 → **필드 단위 merge**
- **주문 목록의 정렬과 식별은 항상 `createdAt`·`orderId` 기준**. `orderNo`는 초기화로 인해 같은 날 중복될 수 있다
- 메뉴 타일 최소 **160×180px**, 타일 간 간격 **14px 이상**
- 호출 기본 문구: **`"{번호}번 손님, 음료 나왔습니다."`**
- `priceMode` 기본값 **`'off'`**. 단 `price` 필드는 P0부터 데이터에 존재한다
- 관리 화면은 **4자리 PIN**으로 잠근다
- **접근성 8원칙(PRD 5장)은 기능보다 우선한다.** 특히: 색에만 의존하지 않기, 누를 수 없는 버튼은 누르게 두지 않기, 모든 동작은 되돌릴 수 있게

---

## File Structure

```
Dallbodrae/
  index.html                      ← 앱 전체 (배포되는 유일한 파일)
  README.md                       ← 설정·배포·운영 안내
  firestore.rules                 ← 기존 규칙에 "추가"할 블록
  .gitignore
  달보드레_카페_POS_PRD.md          ← 이미 있음
  달보드레_카페_POS_구현계획.md      ← 이 문서
```

`index.html` 내부 구획 (이 순서와 주석 배너를 지킬 것):

```
<head>  스타일 (CSS 변수 · 글자 크기 3단계 · 공통 컴포넌트)
<body>  상단바 / main / 푸터

<script type="module">
  // ===== 1. 설정 (CONFIG) =====
  // ===== 2. 순수 로직 =====            ← 자가 테스트 대상. DOM·네트워크를 만지지 않는다
  // ===== 3. 백엔드 어댑터 =====        ← 로컬 / Firestore
  // ===== 4. 공통 UI 부품 =====         ← 모달 · 토스트 · 확인창
  // ===== 5. 화면 =====                 ← 역할선택 / 주문 / 제조 / 호출 / 관리
  // ===== 6. 자가 테스트 =====          ← ?selftest=1 일 때만 실행
  // ===== 7. 시작 =====
</script>
```

**2번 구획(순수 로직)에는 DOM 접근과 네트워크 호출을 절대 넣지 않는다.** 이 규칙이 깨지면 자가 테스트가 무력해진다.

---

# Phase 1 — P0: 여기까지만 해도 카페가 돌아간다

---

### Task 1: 프로젝트 뼈대 + 자가 테스트 러너 + CONFIG

**Files:**
- Create: `index.html`
- Create: `.gitignore`

**Interfaces:**
- Consumes: 없음 (첫 태스크)
- Produces:
  - `CONFIG` — 전역 설정 객체
  - `test(name, fn)` — 테스트 등록
  - `eq(actual, expected, msg?)` — 깊은 비교 단언. 불일치 시 `Error` 발생
  - `runSelfTest()` — 등록된 테스트를 모두 실행하고 결과를 `<pre id="selftest-result">`에 출력

- [ ] **Step 1: git 저장소 초기화**

```bash
git init
git branch -M main
```

- [ ] **Step 2: `.gitignore` 작성**

```
.DS_Store
Thumbs.db
*.log
```

- [ ] **Step 3: 실패하는 테스트를 포함한 `index.html` 뼈대 작성**

아래 내용으로 `index.html`을 만든다. 6번 구획의 테스트 2개는 아직 통과하지 않는다 (`CONFIG`가 없으므로).

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no">
<title>달보드레</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@400;700;900&display=swap" rel="stylesheet">
<style>
  /* 달보드레 — '달콤하고 부드럽다'는 이름에 맞춘 부드러운 색감 */
  :root {
    --bg: #fdf7f2;
    --panel: #ffffff;
    --ink: #2a2320;
    --muted: #7a6b60;
    --line: #e8dcd2;
    --brand: #b5651d;
    --brand-dark: #8c4a12;
    --ok: #2f7d4f;
    --warn: #d98a00;
    --danger: #c0392b;
    --radius: 18px;
    /* 글자 크기 3단계 — Task 5에서 실제로 전환한다 */
    --scale: 1;
  }
  * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
  html, body { margin: 0; height: 100%; }
  body {
    background: var(--bg); color: var(--ink);
    font-family: 'Noto Sans KR', system-ui, sans-serif;
    font-size: calc(16px * var(--scale));
    display: flex; flex-direction: column; min-height: 100vh;
    user-select: none; -webkit-user-select: none;
  }
  input, textarea, select { user-select: text; -webkit-user-select: text; font: inherit; }
  button { font: inherit; cursor: pointer; border: none; touch-action: manipulation; }
  button:disabled { cursor: not-allowed; opacity: .4; }
  main { flex: 1; padding: 16px; min-height: 0; }
  footer { text-align: center; padding: 10px; color: var(--muted); font-size: calc(13px * var(--scale)); }
  #selftest-result { padding: 20px; font-size: 15px; white-space: pre-wrap; line-height: 1.7; }
</style>
</head>
<body>
<main id="app"></main>
<footer>Made by 정우쌤</footer>

<script type="module">
// ===== 1. 설정 (CONFIG) =====
// (Step 5에서 채운다)

// ===== 2. 순수 로직 =====

// ===== 3. 백엔드 어댑터 =====

// ===== 4. 공통 UI 부품 =====

// ===== 5. 화면 =====

// ===== 6. 자가 테스트 =====
// DOM·네트워크를 건드리지 않는 순수 로직만 여기서 검증한다.
const TESTS = [];
function test(name, fn) { TESTS.push({ name, fn }); }

// 깊은 비교 단언. 다르면 Error를 던진다.
function eq(actual, expected, msg) {
  const a = JSON.stringify(actual), e = JSON.stringify(expected);
  if (a !== e) throw new Error(`${msg ? msg + ' — ' : ''}기대: ${e} / 실제: ${a}`);
}

function runSelfTest() {
  const fails = [];
  let pass = 0;
  for (const t of TESTS) {
    try { t.fn(); pass++; }
    catch (err) { fails.push(`✗ ${t.name}\n    ${err.message}`); }
  }
  const head = fails.length === 0 ? `PASS ${pass}/${TESTS.length}` : `FAIL ${pass}/${TESTS.length}`;
  document.body.innerHTML =
    `<pre id="selftest-result">${head}\n\n${fails.join('\n\n') || '모든 테스트를 통과했습니다.'}</pre>`;
}

test('테스트 러너가 불일치를 잡아낸다', () => {
  let caught = false;
  try { eq(1, 2); } catch (err) { caught = true; }
  if (!caught) throw new Error('eq가 불일치를 통과시켰습니다');
});

test('CONFIG에 필수 값이 있다', () => {
  eq(CONFIG.dataRoot, 'dallbodrae_pos');
  eq(typeof CONFIG.spaceId, 'string');
  eq(CONFIG.appName, 'dallbodrae-pos');
});

// ===== 7. 시작 =====
if (new URLSearchParams(location.search).has('selftest')) {
  runSelfTest();
} else {
  document.getElementById('app').textContent = '준비 중';
}
</script>
</body>
</html>
```

- [ ] **Step 4: 테스트가 실패하는지 확인**

한 터미널에서 서버를 띄운다.

```bash
python -m http.server 8320
```

브라우저로 `http://localhost:8320/?selftest=1` 접속.
기대: `FAIL 1/2` 과 `✗ CONFIG에 필수 값이 있다 — CONFIG is not defined` 비슷한 메시지.

- [ ] **Step 5: CONFIG를 작성해 테스트를 통과시킨다**

1번 구획을 아래로 채운다.

```js
// ===== 1. 설정 (CONFIG) =====
// 여기 값만 바꾸면 다른 학급·다른 학년도에서도 그대로 쓸 수 있다.
const CONFIG = {
  // Firebase 콘솔 > 프로젝트 설정 > 내 앱 > (웹 앱 추가) 의 값을 붙여넣는다.
  // 비워두면 로컬 모드로 동작한다 (같은 기기의 탭끼리만 동기화 · 연습용).
  firebase: {
    apiKey: '',
    authDomain: '',
    projectId: '',
    storageBucket: '',
    messagingSenderId: '',
    appId: '',
  },
  firebaseVersion: '12.19.0',
  appName: 'dallbodrae-pos',     // Firebase 인스턴스 이름 — 다른 앱과 캐시가 섞이지 않게
  databaseId: '',                // 비우면 (default) 데이터베이스
  dataRoot: 'dallbodrae_pos',    // 최상위 컬렉션 — 기존 앱이 쓰지 않는 이름이어야 한다
  spaceId: 'cafe-2026',          // 운영 단위. 새로 시작하고 싶으면 이 값만 바꾼다
  cafeName: '달보드레',

  defaults: {
    priceMode: 'off',            // 'off' | 'show' | 'practice'
    fontScale: 'normal',         // 'normal' | 'large' | 'xlarge'
    ttsText: '{번호}번 손님, 음료 나왔습니다.',
    ttsRate: 1.0,
    ttsVolume: 1.0,
    chime: true,
    lateMinutes: 5,              // 제조 지연 강조 기준 (분)
    pin: '1234',                 // 관리 화면 PIN (설정에서 변경 가능)
  },

  photoMaxPx: 400,               // 사진 리사이즈 한 변 (px)
  photoQuality: 0.7,             // JPEG 품질
  photoMaxBytes: 300 * 1024,     // 이보다 크면 품질을 낮춰 재시도
  undoSeconds: 6,                // "완료" 되돌리기 가능 시간 (초)
  recentOrderCount: 12,          // 주문 화면 최근 주문 표시 개수
  displayRecentCount: 4,         // 호출 화면에 함께 보여줄 직전 번호 개수

  localStorageKey: 'dallbodrae-local-db',
  roleKey: 'dallbodrae-role',
  fontScaleKey: 'dallbodrae-font-scale',
};

const FONT_SCALES = { normal: 1, large: 1.25, xlarge: 1.55 };
```

- [ ] **Step 6: 테스트 통과 확인**

브라우저를 새로고침한다 (`http://localhost:8320/?selftest=1`).
기대: `PASS 2/2`

- [ ] **Step 7: 커밋**

```bash
git add index.html .gitignore 달보드레_카페_POS_PRD.md 달보드레_카페_POS_구현계획.md
git commit -m "$(cat <<'MSG'
feat: 달보드레 POS 뼈대와 자가 테스트 러너

단일 index.html 구조, CONFIG, ?selftest=1 자가 테스트 모드를 추가했다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 2: 순수 로직 — 영업일 · 주문번호 · 초기화 판정

PRD 2.3(영업일 자동 전환), 3.1.4(주문번호 발번), 3.6.1(주문번호 초기화)의 계산 부분이다.
DOM도 네트워크도 없는 함수들이므로 여기서 확실히 잡아두면 나중에 화면에서 디버깅할 일이 줄어든다.

**Files:**
- Modify: `index.html` (2번 구획 · 6번 구획)

**Interfaces:**
- Consumes: `CONFIG`, `test`, `eq` (Task 1)
- Produces:
  - `businessDateOf(date?) → 'YYYY-MM-DD'` — 기기 로컬 시간 기준
  - `nextOrderNo(counterValue) → number` — 카운터는 "마지막으로 발번한 번호"를 담는다
  - `resetPlan(counterValue, startFrom, waitingCount) → { ok, needsConfirm, message, newCounterValue }`

- [ ] **Step 1: 실패하는 테스트를 6번 구획에 추가**

```js
// ---- 영업일 ----
test('영업일은 로컬 시간 기준 YYYY-MM-DD 이다', () => {
  eq(businessDateOf(new Date(2026, 8, 20, 9, 30)), '2026-09-20');
  eq(businessDateOf(new Date(2026, 0, 5, 23, 59)), '2026-01-05', '한 자리 월·일은 0으로 채운다');
});

// ---- 주문번호 발번 ----
test('주문번호는 카운터 다음 값이다', () => {
  eq(nextOrderNo(0), 1, '하루의 첫 주문은 1번');
  eq(nextOrderNo(7), 8);
});

// ---- 주문번호 초기화 판정 ----
test('대기 주문이 없으면 확인 한 번으로 초기화한다', () => {
  const p = resetPlan(57, 1, 0);
  eq(p.ok, true);
  eq(p.needsConfirm, false);
  eq(p.newCounterValue, 0, '1번부터 시작하려면 카운터는 0');
});

test('시작 번호를 지정할 수 있다', () => {
  eq(resetPlan(57, 100, 0).newCounterValue, 99);
});

test('대기 주문이 있으면 중복 경고와 함께 한 번 더 확인받는다', () => {
  const p = resetPlan(57, 1, 3);
  eq(p.ok, true);
  eq(p.needsConfirm, true);
  if (!p.message.includes('3건')) throw new Error(`대기 건수가 안내에 없습니다: ${p.message}`);
  if (!p.message.includes('두 번')) throw new Error(`번호 중복 경고가 없습니다: ${p.message}`);
});

test('시작 번호가 1 미만이거나 정수가 아니면 거부한다', () => {
  eq(resetPlan(57, 0, 0).ok, false);
  eq(resetPlan(57, -3, 0).ok, false);
  eq(resetPlan(57, 1.5, 0).ok, false);
  eq(resetPlan(57, NaN, 0).ok, false);
});
```

- [ ] **Step 2: 실패 확인**

`http://localhost:8320/?selftest=1` 새로고침.
기대: `FAIL 2/8`, 새 테스트 6개가 `businessDateOf is not defined` 등으로 실패.

- [ ] **Step 3: 2번 구획에 구현 작성**

```js
// ===== 2. 순수 로직 =====
// 이 구획에는 DOM 접근과 네트워크 호출을 넣지 않는다 (자가 테스트 대상).

// ---- 영업일 ----
// 상시 운영이므로 날짜가 바뀌면 주문번호가 저절로 1번부터 다시 시작한다.
function businessDateOf(date = new Date()) {
  const p = (n) => String(n).padStart(2, '0');
  return `${date.getFullYear()}-${p(date.getMonth() + 1)}-${p(date.getDate())}`;
}

// ---- 주문번호 ----
// counters/{영업일}.value 는 "마지막으로 발번한 번호"를 담는다. 없으면 0으로 본다.
function nextOrderNo(counterValue) {
  return (counterValue || 0) + 1;
}

// ---- 주문번호 초기화 (PRD 3.6.1) ----
// 아직 받아가지 않은 주문이 남은 채로 번호를 되돌리면 같은 번호가 두 번 불린다.
// 그래서 대기 건수를 보여주고 한 번 더 확인받는다.
function resetPlan(counterValue, startFrom, waitingCount) {
  if (!Number.isInteger(startFrom) || startFrom < 1) {
    return {
      ok: false,
      needsConfirm: false,
      message: '시작 번호는 1 이상의 정수여야 합니다.',
      newCounterValue: counterValue,
    };
  }
  const newCounterValue = startFrom - 1;
  if (waitingCount > 0) {
    return {
      ok: true,
      needsConfirm: true,
      message: `아직 받지 않은 주문이 ${waitingCount}건 있어요.\n지금 번호를 되돌리면 같은 번호가 두 번 불릴 수 있습니다.`,
      newCounterValue,
    };
  }
  return {
    ok: true,
    needsConfirm: false,
    message: `주문번호를 ${startFrom}번부터 다시 시작합니다.`,
    newCounterValue,
  };
}
```

- [ ] **Step 4: 통과 확인**

새로고침. 기대: `PASS 8/8`

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 영업일·주문번호·초기화 판정 로직

주문번호 초기화는 대기 주문이 남아 있으면 번호 중복 경고와 함께
한 번 더 확인받는다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 3: 사진 리사이즈

PRD 6.3. 선생님이 휴대폰으로 찍은 3~5MB 사진을 그대로 저장하면 Firestore 문서 1MB 제한에 걸린다.
업로드 순간 400×400 JPEG로 줄이고, 그래도 크면 품질을 낮춰 재시도한다.

**Files:**
- Modify: `index.html` (2번 구획 · 6번 구획)

**Interfaces:**
- Consumes: `CONFIG` (Task 1)
- Produces:
  - `dataUrlBytes(dataUrl) → number` — base64 data URL의 실제 바이트 수 (순수 함수)
  - `fitBox(w, h, maxPx) → { w, h }` — 비율 유지 축소 크기 (순수 함수)
  - `async shrinkImage(file, opts?) → dataUrl` — 캔버스를 쓰므로 자가 테스트 대상이 아니다

- [ ] **Step 1: 실패하는 테스트 추가**

```js
// ---- 사진 ----
test('data URL 바이트 수를 센다', () => {
  // 'AAAA' 는 base64 4글자 → 3바이트
  eq(dataUrlBytes('data:image/jpeg;base64,AAAA'), 3);
  // 패딩 '=' 은 바이트에서 뺀다
  eq(dataUrlBytes('data:image/jpeg;base64,AAA='), 2);
  eq(dataUrlBytes('data:image/jpeg;base64,AA=='), 1);
});

test('사진은 비율을 지키며 한 변이 maxPx 이하가 된다', () => {
  eq(fitBox(1200, 900, 400), { w: 400, h: 300 });
  eq(fitBox(900, 1200, 400), { w: 300, h: 400 });
  eq(fitBox(400, 400, 400), { w: 400, h: 400 });
  eq(fitBox(200, 100, 400), { w: 200, h: 100 }, '원본이 작으면 키우지 않는다');
});
```

- [ ] **Step 2: 실패 확인** — `FAIL 8/10`

- [ ] **Step 3: 2번 구획에 순수 함수, 그리고 캔버스를 쓰는 `shrinkImage` 작성**

```js
// ---- 사진 리사이즈 (PRD 6.3) ----
// Firebase Storage를 쓰지 않고 Firestore 문서에 data URL로 넣기 때문에
// 업로드 시점에 반드시 줄여야 한다.

// base64 data URL의 실제 바이트 수
function dataUrlBytes(dataUrl) {
  const b64 = dataUrl.slice(dataUrl.indexOf(',') + 1);
  const padding = (b64.match(/=+$/) || [''])[0].length;
  return Math.floor(b64.length * 3 / 4) - padding;
}

// 비율을 지키며 한 변이 maxPx 이하가 되는 크기. 원본이 더 작으면 그대로 둔다.
function fitBox(w, h, maxPx) {
  const ratio = Math.min(1, maxPx / Math.max(w, h));
  return { w: Math.round(w * ratio), h: Math.round(h * ratio) };
}

// File → 줄인 JPEG data URL. 목표 용량을 넘으면 품질을 낮춰 최대 3번 재시도한다.
async function shrinkImage(file, opts = {}) {
  const maxPx = opts.maxPx ?? CONFIG.photoMaxPx;
  const maxBytes = opts.maxBytes ?? CONFIG.photoMaxBytes;
  let quality = opts.quality ?? CONFIG.photoQuality;

  // imageOrientation: 휴대폰으로 세로로 찍은 사진이 눕지 않게 EXIF 회전을 반영한다
  const bitmap = await createImageBitmap(file, { imageOrientation: 'from-image' });
  const { w, h } = fitBox(bitmap.width, bitmap.height, maxPx);
  const canvas = document.createElement('canvas');
  canvas.width = w; canvas.height = h;
  const ctx = canvas.getContext('2d');
  ctx.drawImage(bitmap, 0, 0, w, h);
  bitmap.close?.();

  let dataUrl = canvas.toDataURL('image/jpeg', quality);
  for (let i = 0; i < 3 && dataUrlBytes(dataUrl) > maxBytes; i++) {
    quality = Math.max(0.3, quality - 0.15);
    dataUrl = canvas.toDataURL('image/jpeg', quality);
  }
  return dataUrl;
}
```

- [ ] **Step 4: 통과 확인** — `PASS 10/10`

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 사진 리사이즈 (400px JPEG, 용량 초과 시 품질 재시도)

Firebase Storage 없이 Firestore 문서에 data URL로 넣기 위해
업로드 시점에 줄인다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 4: 백엔드 어댑터 — 로컬 모드

앱 전체가 쓰는 데이터 인터페이스를 여기서 확정한다. Firestore 구현(Task 11)은 **같은 6개 메서드**를 제공한다.
로컬 모드를 먼저 만드는 이유: Firebase 설정 없이도 이후 모든 화면 태스크를 개발·확인할 수 있다.

**Files:**
- Modify: `index.html` (3번 구획 · 6번 구획)

**Interfaces:**
- Consumes: `CONFIG`, `nextOrderNo`, `businessDateOf`
- Produces: `createLocalBackend() → backend`, 그리고 **모든 백엔드가 지키는 인터페이스**:

```js
backend.mode                                  // 'local' | 'firestore'
backend.subscribe(name, cb) → unsubscribe     // name: 'menus'|'ingredients'|'orders'|'stockLogs'|'counterResets'|'settings'
                                              // cb(docs) — docs는 [{ id, ...data }] 배열
backend.add(name, data) → Promise<id>
backend.set(name, id, patch) → Promise        // 필드 단위 merge (덮어쓰기 아님)
backend.remove(name, id) → Promise
backend.issueOrderNo(businessDate) → Promise<number>          // 트랜잭션 발번
backend.resetOrderNo(businessDate, value) → Promise           // 카운터를 value로 설정
backend.bump(name, id, fields) → Promise      // { current: -150 } 원자적 증감
```

- [ ] **Step 1: 실패하는 테스트 추가**

로컬 백엔드는 localStorage를 쓰므로 테스트용 키를 따로 주고, 테스트가 끝나면 지운다.

```js
// ---- 로컬 백엔드 ----
test('로컬 백엔드가 추가·수정·삭제를 한다', () => {
  const key = '__test-db-1';
  localStorage.removeItem(key);
  const be = createLocalBackend(key);
  let seen = null;
  be.subscribe('menus', (docs) => { seen = docs; });

  const id = be.addSync('menus', { name: '아메리카노', price: 1000 });
  eq(seen.length, 1);
  eq(seen[0].name, '아메리카노');

  be.setSync('menus', id, { soldOut: true });
  eq(seen[0].name, '아메리카노', 'set은 덮어쓰지 않고 합친다');
  eq(seen[0].soldOut, true);

  be.removeSync('menus', id);
  eq(seen.length, 0);
  localStorage.removeItem(key);
});

test('로컬 백엔드가 주문번호를 순서대로 발번한다', () => {
  const key = '__test-db-2';
  localStorage.removeItem(key);
  const be = createLocalBackend(key);
  eq(be.issueOrderNoSync('2026-09-20'), 1);
  eq(be.issueOrderNoSync('2026-09-20'), 2);
  eq(be.issueOrderNoSync('2026-09-21'), 1, '날짜가 바뀌면 1번부터');
  be.resetOrderNoSync('2026-09-20', 0);
  eq(be.issueOrderNoSync('2026-09-20'), 1, '초기화하면 다시 1번');
  localStorage.removeItem(key);
});

test('로컬 백엔드가 필드를 원자적으로 증감한다', () => {
  const key = '__test-db-3';
  localStorage.removeItem(key);
  const be = createLocalBackend(key);
  const id = be.addSync('ingredients', { name: '우유', current: 1000 });
  be.bumpSync('ingredients', id, { current: -150 });
  be.bumpSync('ingredients', id, { current: -150 });
  let seen = null;
  be.subscribe('ingredients', (docs) => { seen = docs; });
  eq(seen[0].current, 700);
  localStorage.removeItem(key);
});
```

- [ ] **Step 2: 실패 확인** — `FAIL 10/13`

- [ ] **Step 3: 3번 구획에 로컬 백엔드 작성**

비동기 메서드는 동기 구현을 감싸기만 한다. 테스트는 동기 버전(`~Sync`)을 쓰고, 앱은 비동기 버전을 쓴다.

```js
// ===== 3. 백엔드 어댑터 =====
// 로컬 모드와 Firestore 모드가 아래 메서드를 똑같이 제공한다.
//   subscribe / add / set / remove / issueOrderNo / resetOrderNo / bump

const COLLECTIONS = ['menus', 'ingredients', 'orders', 'stockLogs', 'counterResets', 'settings'];

// ---- 로컬 모드 ----
// CONFIG.firebase.apiKey 가 비어 있을 때 쓴다.
// 같은 기기의 탭끼리만 동기화된다 (storage 이벤트). 연습·시연용.
function createLocalBackend(storageKey = CONFIG.localStorageKey) {
  const listeners = {};   // { 컬렉션명: [cb, ...] }

  function load() {
    try { return JSON.parse(localStorage.getItem(storageKey)) || {}; }
    catch { return {}; }
  }
  function save(db) {
    localStorage.setItem(storageKey, JSON.stringify(db));
  }
  function docsOf(db, name) {
    return Object.entries(db[name] || {}).map(([id, data]) => ({ id, ...data }));
  }
  function notify(db, name) {
    for (const cb of listeners[name] || []) cb(docsOf(db, name));
  }
  function notifyAll(db) {
    for (const name of Object.keys(listeners)) notify(db, name);
  }

  // 다른 탭에서 바뀌면 이쪽에도 반영한다
  window.addEventListener('storage', (e) => {
    if (e.key === storageKey) notifyAll(load());
  });

  const api = {
    mode: 'local',

    subscribe(name, cb) {
      (listeners[name] ||= []).push(cb);
      cb(docsOf(load(), name));   // 구독 즉시 현재 상태를 한 번 준다
      return () => {
        listeners[name] = listeners[name].filter((f) => f !== cb);
      };
    },

    addSync(name, data) {
      const db = load();
      const id = `${Date.now().toString(36)}-${Math.random().toString(36).slice(2, 8)}`;
      (db[name] ||= {})[id] = data;
      save(db); notify(db, name);
      return id;
    },

    setSync(name, id, patch) {
      const db = load();
      const prev = (db[name] ||= {})[id] || {};
      db[name][id] = { ...prev, ...patch };   // 필드 단위 merge
      save(db); notify(db, name);
    },

    removeSync(name, id) {
      const db = load();
      if (db[name]) delete db[name][id];
      save(db); notify(db, name);
    },

    bumpSync(name, id, fields) {
      const db = load();
      const prev = (db[name] ||= {})[id] || {};
      const next = { ...prev };
      for (const [k, delta] of Object.entries(fields)) next[k] = (next[k] || 0) + delta;
      db[name][id] = next;
      save(db); notify(db, name);
    },

    issueOrderNoSync(businessDate) {
      const db = load();
      const counters = (db.counters ||= {});
      const value = nextOrderNo(counters[businessDate] || 0);
      counters[businessDate] = value;
      save(db);
      return value;
    },

    resetOrderNoSync(businessDate, value) {
      const db = load();
      (db.counters ||= {})[businessDate] = value;
      save(db);
    },

    clearSync() {
      localStorage.removeItem(storageKey);
      notifyAll({});
    },
  };

  // 앱이 쓰는 비동기 인터페이스
  api.add = async (n, d) => api.addSync(n, d);
  api.set = async (n, i, p) => api.setSync(n, i, p);
  api.remove = async (n, i) => api.removeSync(n, i);
  api.bump = async (n, i, f) => api.bumpSync(n, i, f);
  api.issueOrderNo = async (bd) => api.issueOrderNoSync(bd);
  api.resetOrderNo = async (bd, v) => api.resetOrderNoSync(bd, v);
  api.clear = async () => api.clearSync();
  return api;
}
```

- [ ] **Step 4: 통과 확인** — `PASS 13/13`

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 로컬 모드 백엔드 어댑터

Firestore 구현이 따를 6개 메서드 인터페이스를 확정했다.
Firebase 설정 없이도 모든 화면을 개발·확인할 수 있다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 5: 역할 선택 · 라우팅 · 글자 크기 3단계

PRD 4장(화면 구성), 3.1.7(글자 크기). 여기서 앱이 처음으로 "화면"이 된다.

**Files:**
- Modify: `index.html` (4번 구획 · 5번 구획 · 7번 구획, `<body>`)

**Interfaces:**
- Consumes: `CONFIG`, `FONT_SCALES`, `createLocalBackend`
- Produces:
  - `state` — `{ role, backend, menus, ingredients, orders, settings, fontScale }`
  - `setRole(role)` / `changeRole()`
  - `setFontScale(key)` — `'normal' | 'large' | 'xlarge'`
  - `render()` — 현재 역할의 화면을 그린다
  - `VIEWS` — `{ order, barista, display, admin }` 각각 `render(el)` 함수를 가진 객체
  - `toast(text, opts?)`, `confirmModal({ title, msg, okText?, okClass? }) → Promise<boolean>`

- [ ] **Step 1: 역할 선택 화면과 라우팅 구현**

`<body>`를 상단바 + main + 푸터 구조로 바꾸고, 5번 구획에 `VIEWS`를 만든다.

```html
<body>
<div class="topbar" id="topbar" hidden>
  <h1 id="topbar-title">달보드레</h1>
  <span class="role-tag" id="role-tag"></span>
  <span class="staff" id="staff-tag"></span>
  <span class="spacer"></span>
  <span class="net-tag" id="net-tag"></span>
  <div class="font-switch" role="group" aria-label="글자 크기">
    <button data-scale="normal">가</button>
    <button data-scale="large">가</button>
    <button data-scale="xlarge">가</button>
  </div>
  <button id="btn-change-role">역할 바꾸기</button>
</div>
<main id="app"></main>
<footer>Made by 정우쌤</footer>
```

```js
// ===== 5. 화면 =====
const state = {
  role: null,
  backend: null,
  menus: [], ingredients: [], orders: [], settings: {},
  fontScale: 'normal',
};

const ROLE_LABEL = { order: '주문', barista: '제조', display: '호출', admin: '관리' };

// ---- 글자 크기 (PRD 3.1.7) ----
// 학생마다 편차가 커서 설정 깊숙이 넣지 않고 상단바에 상시 노출한다.
function setFontScale(key) {
  state.fontScale = FONT_SCALES[key] ? key : 'normal';
  document.documentElement.style.setProperty('--scale', FONT_SCALES[state.fontScale]);
  localStorage.setItem(CONFIG.fontScaleKey, state.fontScale);
  for (const b of document.querySelectorAll('.font-switch button')) {
    b.classList.toggle('on', b.dataset.scale === state.fontScale);
  }
}

// ---- 역할 ----
function setRole(role) {
  state.role = role;
  if (role) localStorage.setItem(CONFIG.roleKey, role);
  else localStorage.removeItem(CONFIG.roleKey);
  render();
}
function changeRole() { setRole(null); }

// ---- 역할 선택 화면 ----
function renderRolePicker(el) {
  el.innerHTML = `
    <div class="role-grid">
      <button data-role="order"><span>🧾</span>주문<small>캐셔 학생</small></button>
      <button data-role="barista"><span>☕</span>제조<small>바리스타 학생</small></button>
      <button data-role="display"><span>📣</span>호출<small>손님이 보는 화면</small></button>
      <button data-role="admin"><span>⚙️</span>관리<small>선생님</small></button>
    </div>`;
  for (const b of el.querySelectorAll('[data-role]')) {
    b.onclick = () => setRole(b.dataset.role);
  }
}

// ---- 라우팅 ----
const VIEWS = {
  order:   { render: (el) => { el.textContent = '주문 화면 준비 중'; } },
  barista: { render: (el) => { el.textContent = '제조 화면 준비 중'; } },
  display: { render: (el) => { el.textContent = '호출 화면 준비 중'; } },
  admin:   { render: (el) => { el.textContent = '관리 화면 준비 중'; } },
};

function render() {
  const el = document.getElementById('app');
  const bar = document.getElementById('topbar');
  if (!state.role) {
    bar.hidden = true;
    renderRolePicker(el);
    return;
  }
  bar.hidden = state.role === 'display';   // 호출 화면은 손님이 보므로 상단바를 감춘다
  document.getElementById('role-tag').textContent = ROLE_LABEL[state.role];
  el.innerHTML = '';
  VIEWS[state.role].render(el);
}
```

- [ ] **Step 2: 공통 UI 부품(토스트·확인창)을 4번 구획에 작성**

```js
// ===== 4. 공통 UI 부품 =====

// 되돌릴 수 있는 알림. 접근성 원칙 4번(모든 동작은 되돌릴 수 있다)의 손발이다.
function toast(text, { actionText, onAction, ms = 3000 } = {}) {
  const el = document.createElement('div');
  el.className = 'toast';
  el.innerHTML = `<span></span>`;
  el.firstChild.textContent = text;
  if (actionText) {
    const b = document.createElement('button');
    b.textContent = actionText;
    b.onclick = () => { el.remove(); onAction?.(); };
    el.appendChild(b);
  }
  document.body.appendChild(el);
  setTimeout(() => el.remove(), ms);
}

// 확인창. 파괴적인 동작 앞에 반드시 둔다.
function confirmModal({ title, msg, okText = '확인', okClass = 'primary', cancelText = '취소' }) {
  return new Promise((resolve) => {
    const back = document.createElement('div');
    back.className = 'modal-back';
    back.innerHTML = `
      <div class="modal">
        <h3></h3><p></p>
        <div class="modal-actions">
          <button class="btn ghost" data-no></button>
          <button class="btn ${okClass}" data-yes></button>
        </div>
      </div>`;
    back.querySelector('h3').textContent = title;
    back.querySelector('p').textContent = msg;       // \n 을 살리려면 CSS white-space: pre-line
    back.querySelector('[data-no]').textContent = cancelText;
    back.querySelector('[data-yes]').textContent = okText;
    back.querySelector('[data-no]').onclick = () => { back.remove(); resolve(false); };
    back.querySelector('[data-yes]').onclick = () => { back.remove(); resolve(true); };
    document.body.appendChild(back);
  });
}
```

`.modal p { white-space: pre-line; }` 를 스타일에 추가한다 (`resetPlan`의 메시지가 두 줄이다).

- [ ] **Step 3: 시작 코드를 7번 구획에 작성**

```js
// ===== 7. 시작 =====
if (new URLSearchParams(location.search).has('selftest')) {
  runSelfTest();
} else {
  // Firebase 키가 비어 있으면 로컬 모드로 동작한다 (PRD 6.2)
  state.backend = createLocalBackend();

  // 저장된 데이터가 깨져 있으면 로컬 백엔드는 조용히 빈 상태로 시작한다.
  // 선생님 눈에는 "메뉴가 이유 없이 사라진" 것으로 보이므로 반드시 알린다.
  const raw = localStorage.getItem(CONFIG.localStorageKey);
  if (raw) {
    try { JSON.parse(raw); }
    catch {
      toast('저장된 자료를 읽지 못했어요. 메뉴를 다시 등록해 주세요.', { ms: 8000 });
    }
  }

  for (const name of ['menus', 'ingredients', 'orders']) {
    state.backend.subscribe(name, (docs) => { state[name] = docs; render(); });
  }
  state.backend.subscribe('settings', (docs) => {
    state.settings = { ...CONFIG.defaults, ...(docs.find((d) => d.id === 'app') || {}) };
    render();
  });

  // 지금 로컬인지 클라우드인지 상단바에 표시한다.
  // Task 11에서 Firestore를 붙이면 '연결됨'으로 바뀐다.
  document.getElementById('net-tag').textContent = '연습 모드';

  setFontScale(localStorage.getItem(CONFIG.fontScaleKey) || CONFIG.defaults.fontScale);
  for (const b of document.querySelectorAll('.font-switch button')) {
    b.onclick = () => setFontScale(b.dataset.scale);
  }
  document.getElementById('btn-change-role').onclick = changeRole;

  setRole(localStorage.getItem(CONFIG.roleKey) || null);
}
```

- [ ] **Step 4: 자가 테스트가 여전히 통과하는지 확인**

`http://localhost:8320/?selftest=1` → `PASS 13/13`
(화면 코드를 추가해도 순수 로직 테스트는 깨지면 안 된다.)

- [ ] **Step 5: 화면 확인**

`http://localhost:8320/` 접속.

| 확인할 것 | 기대 |
|---|---|
| 첫 화면 | 역할 카드 4개(주문/제조/호출/관리)가 큰 타일로 보인다 |
| 주문 카드 클릭 | "주문 화면 준비 중" + 상단바에 `주문` 배지 |
| 새로고침 | 역할 선택 화면으로 돌아가지 않고 주문 화면이 유지된다 |
| 상단바 `가 가 가` | 누를 때마다 글자가 커진다. 새로고침해도 유지된다 |
| `역할 바꾸기` | 역할 선택 화면으로 돌아간다 |
| 호출 역할 선택 | 상단바가 사라진다 |
| 푸터 | 모든 화면 아래에 `Made by 정우쌤` |

- [ ] **Step 6: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 역할 선택·라우팅·글자 크기 3단계

글자 크기는 학생별 편차를 고려해 설정이 아닌 상단바에 상시 노출한다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 6: 관리 화면 — PIN · 메뉴 등록(사진) · 품절 · 오늘의 담당

PRD 3.6. 메뉴가 없으면 주문 화면을 만들 수 없으므로 관리 화면을 먼저 만든다.
**이 태스크가 "선생님이 직접 커스터마이징한다"는 이 앱의 핵심 요구를 실현하는 곳이다.**

**Files:**
- Modify: `index.html` (2번 · 5번 · 6번 구획)

**Interfaces:**
- Consumes: `state`, `shrinkImage`, `confirmModal`, `toast`, `backend`
- Produces:
  - `sortedMenus(menus) → menu[]` — `order` 오름차순, 같으면 이름순 (순수 함수)
  - `saveSettings(patch) → Promise`
  - `VIEWS.admin.render(el)`

- [ ] **Step 1: 실패하는 테스트 추가**

```js
// ---- 메뉴 정렬 ----
test('메뉴는 order 오름차순, 같으면 이름순으로 정렬된다', () => {
  const list = [
    { id: 'c', name: '핫초코', order: 2 },
    { id: 'a', name: '아메리카노', order: 1 },
    { id: 'b', name: '녹차라떼', order: 1 },
  ];
  eq(sortedMenus(list).map((m) => m.id), ['b', 'a', 'c']);
});

test('order가 없는 메뉴는 맨 뒤로 간다', () => {
  const list = [
    { id: 'x', name: '신메뉴' },
    { id: 'a', name: '아메리카노', order: 1 },
  ];
  eq(sortedMenus(list).map((m) => m.id), ['a', 'x']);
});
```

- [ ] **Step 2: 실패 확인** — `FAIL 13/15`

- [ ] **Step 3: `sortedMenus`를 2번 구획에 작성**

```js
// ---- 메뉴 정렬 ----
// order가 없는 메뉴는 맨 뒤에 둔다 (새로 만든 메뉴가 중간에 끼어들지 않게).
function sortedMenus(menus) {
  return [...menus].sort((a, b) => {
    const ao = a.order ?? Number.MAX_SAFE_INTEGER;
    const bo = b.order ?? Number.MAX_SAFE_INTEGER;
    if (ao !== bo) return ao - bo;
    return String(a.name).localeCompare(String(b.name), 'ko');
  });
}
```

- [ ] **Step 4: 통과 확인** — `PASS 15/15`

- [ ] **Step 5: 관리 화면 구현**

5번 구획의 `VIEWS.admin`을 채운다. 구성:

**(a) PIN 잠금** — 관리 화면에 처음 들어갈 때 4자리 PIN을 묻는다. 맞으면 이 탭에서는 다시 묻지 않는다 (`sessionStorage`). 틀리면 흔들림 애니메이션 + 다시 입력.

```js
// 학생이 실수로 들어가 메뉴를 지우는 일을 막는 잠금장치다.
// 보안이 목적이 아니므로 세션 단위로만 기억한다.
let adminUnlocked = sessionStorage.getItem('dallbodrae-admin') === 'ok';
let pinInput = '';

function renderPinGate(el) {
  el.innerHTML = `
    <div class="card pin-gate">
      <h2>선생님 확인</h2>
      <p>관리 화면에 들어가려면 비밀번호 4자리를 눌러 주세요.</p>
      <div class="pin-dots" id="pin-dots"></div>
      <div class="keypad" id="pin-pad"></div>
    </div>`;

  const dots = el.querySelector('#pin-dots');
  const pad = el.querySelector('#pin-pad');

  function drawDots() {
    dots.innerHTML = '';
    for (let i = 0; i < 4; i++) {
      const d = document.createElement('span');
      d.className = 'pin-dot' + (i < pinInput.length ? ' on' : '');
      dots.appendChild(d);
    }
  }

  function push(ch) {
    if (pinInput.length >= 4) return;
    pinInput += ch;
    drawDots();
    if (pinInput.length === 4) setTimeout(check, 150);
  }

  function check() {
    const expected = String(state.settings.pin || CONFIG.defaults.pin);
    if (pinInput === expected) {
      sessionStorage.setItem('dallbodrae-admin', 'ok');
      adminUnlocked = true;
      pinInput = '';
      render();
    } else {
      dots.classList.add('shake');
      setTimeout(() => dots.classList.remove('shake'), 400);
      pinInput = '';
      drawDots();
    }
  }

  for (const label of ['1','2','3','4','5','6','7','8','9','','0','지우기']) {
    const b = document.createElement('button');
    b.textContent = label;
    b.className = 'key' + (label === '지우기' ? ' wide' : '');
    if (label === '') { b.disabled = true; b.className = 'key blank'; }
    else if (label === '지우기') b.onclick = () => { pinInput = pinInput.slice(0, -1); drawDots(); };
    else b.onclick = () => push(label);
    pad.appendChild(b);
  }

  pinInput = '';
  drawDots();
}
```

```css
.pin-dots { display: flex; gap: 14px; justify-content: center; margin: 24px 0; }
.pin-dot { width: 20px; height: 20px; border-radius: 50%; border: 3px solid var(--line); }
.pin-dot.on { background: var(--brand); border-color: var(--brand); }
.pin-dots.shake { animation: shake .4s; }
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  25% { transform: translateX(-10px); }
  75% { transform: translateX(10px); }
}
.keypad { display: grid; grid-template-columns: repeat(3, 1fr); gap: 12px; max-width: 320px; margin: 0 auto; }
.key { min-height: 68px; font-size: 26px; font-weight: 700; border-radius: 14px; background: var(--panel); border: 2px solid var(--line); }
.key.blank { visibility: hidden; }
```

`VIEWS.admin.render`는 맨 위에서 잠금을 확인한다.

```js
VIEWS.admin = {
  render(el) {
    if (!adminUnlocked) { renderPinGate(el); return; }
    renderAdminTabs(el);
  },
};
```

**(b) 탭** — `메뉴` / `담당` / `설정`. (재료·기록은 Phase 2에서 탭을 추가한다.)

**(c) 메뉴 탭**
- 메뉴 목록: 각 행에 썸네일, 이름, 가격, `위/아래` 순서 버튼, `품절` 토글, `수정`, `삭제`
- `＋ 메뉴 추가` → 모달: 이름 / 가격 / 이모지 / **사진 선택**
- 사진 선택 시 `shrinkImage(file)` → 미리보기 표시 → 저장 시 `photo` 필드에 data URL
- **저장 전 용량 확인** — `shrinkImage`는 3회 재시도 후에도 목표 용량을 못 맞추면 그냥 큰
  data URL을 돌려준다. 그대로 Firestore에 쓰면 1MB 문서 제한에 걸려 **아무 안내 없이
  저장이 실패한다.** 저장 직전에 확인하고, 초과하면 선생님이 알아들을 말로 알린다:

```js
if (photo && dataUrlBytes(photo) > CONFIG.photoMaxBytes) {
  toast('사진 용량이 너무 커요. 다른 사진으로 다시 올려 주세요.', { ms: 5000 });
  return;   // 저장하지 않는다
}
```
- 사진이 없으면 `emoji` 필드를 쓴다 (기본값 `☕`)
- 삭제는 `confirmModal`로 확인 (`okClass: 'danger'`)
- 저장은 `backend.set('menus', id, patch)` — 필드 단위 merge

메뉴 문서 모양 (PRD 7장과 일치시킬 것):

```js
{
  name: '아이스 아메리카노',
  photo: 'data:image/jpeg;base64,...' || null,
  emoji: '☕',
  price: 1000,
  order: 1,
  visible: true,
  soldOut: false,
  recipe: [],        // Phase 2에서 채운다. 비어 있으면 재고를 차감하지 않는다
}
```

**(d) 담당 탭** — 오늘의 캐셔·바리스타 이름을 입력한다. `settings/app.todayStaff = { cashier, barista }`.
빈칸이면 상단바에 아무것도 표시하지 않는다.

**(e) 설정 탭** — PIN 변경, 기본 글자 크기, 제조 지연 강조 기준(분), 저장 위치 표시(`모드: 로컬 / 경로: dallbodrae_pos/cafe-2026`).

```js
async function saveSettings(patch) {
  await state.backend.set('settings', 'app', patch);
}
```

상단바에 담당 학생을 표시한다 (`render()` 안):

```js
const staff = state.settings.todayStaff || {};
const who = state.role === 'order' ? staff.cashier : state.role === 'barista' ? staff.barista : '';
const label = state.role === 'order' ? '오늘의 캐셔' : '오늘의 바리스타';
document.getElementById('staff-tag').textContent = who ? `${label}: ${who}` : '';
```

- [ ] **Step 6: 화면 확인**

`http://localhost:8320/` → 관리 역할 선택.

| 확인할 것 | 기대 |
|---|---|
| 관리 화면 진입 | PIN 키패드가 먼저 나온다 |
| 틀린 PIN | 흔들리고 입력이 비워진다. 들어가지지 않는다 |
| `1234` 입력 | 관리 화면이 열린다. 새로고침해도 다시 묻지 않는다 (같은 탭) |
| 메뉴 추가 → 사진 선택 | 큰 사진을 골라도 미리보기가 뜨고 저장된다 |
| 저장 후 | 브라우저 개발자도구 → Application → Local Storage 에서 `photo` 값이 **30~50KB 수준**인지 확인 |
| 순서 위/아래 | 목록 순서가 바뀐다 |
| 품절 토글 | 배지가 켜진다 |
| 삭제 | 확인창이 뜬 뒤에 지워진다 |
| 담당 탭에 이름 입력 후 주문 역할로 전환 | 상단바에 `오늘의 캐셔: ○○` |

- [ ] **Step 7: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 관리 화면 (PIN, 메뉴 등록·사진·순서·품절, 오늘의 담당)

선생님이 코드를 열지 않고 메뉴를 바꿀 수 있는 경로를 완성했다.
사진은 업로드 시점에 400px JPEG로 줄여 저장한다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 7: 주문 화면

PRD 3.1. AAC 원칙(사진 + 글자 병기)이 실제로 구현되는 화면이다.

**Files:**
- Modify: `index.html` (2번 · 5번 · 6번 구획)

**Interfaces:**
- Consumes: `sortedMenus`, `businessDateOf`, `backend.issueOrderNo`, `backend.add`
- Produces:
  - `cartAdd(cart, menu) → cart` / `cartSetQty(cart, menuId, qty) → cart` / `cartTotal(cart) → number` (순수 함수)
  - `cart` 모양: `[{ menuId, name, photo, emoji, price, qty }]`
  - `submitOrder() → Promise<string>` — 저장된 주문의 `orderId`를 반환한다 (Task 13이 쓴다)
  - `VIEWS.order.render(el)`

- [ ] **Step 1: 실패하는 테스트 추가**

```js
// ---- 장바구니 ----
test('같은 메뉴를 담으면 수량이 늘어난다', () => {
  const m = { id: 'a', name: '아메리카노', price: 1000, emoji: '☕', photo: null };
  let cart = cartAdd([], m);
  cart = cartAdd(cart, m);
  eq(cart.length, 1);
  eq(cart[0].qty, 2);
});

test('담는 순서가 유지된다', () => {
  const a = { id: 'a', name: '아메리카노', price: 1000 };
  const b = { id: 'b', name: '핫초코', price: 1500 };
  let cart = cartAdd(cartAdd([], a), b);
  cart = cartAdd(cart, a);
  eq(cart.map((c) => c.menuId), ['a', 'b'], '이미 담긴 메뉴는 제자리에 머문다');
});

test('수량을 0으로 만들면 목록에서 빠진다', () => {
  const a = { id: 'a', name: '아메리카노', price: 1000 };
  const cart = cartSetQty(cartAdd([], a), 'a', 0);
  eq(cart.length, 0);
});

test('합계를 계산한다', () => {
  const cart = [
    { menuId: 'a', price: 1000, qty: 2 },
    { menuId: 'b', price: 1500, qty: 1 },
  ];
  eq(cartTotal(cart), 3500);
  eq(cartTotal([]), 0);
});
```

- [ ] **Step 2: 실패 확인** — `FAIL 15/19`

- [ ] **Step 3: 2번 구획에 장바구니 로직 작성**

```js
// ---- 장바구니 ----
// 화면에 사진을 다시 조회하지 않아도 되게, 담는 시점의 표시 정보를 함께 복사해 둔다.
function cartAdd(cart, menu) {
  const found = cart.find((c) => c.menuId === menu.id);
  if (found) return cart.map((c) => (c.menuId === menu.id ? { ...c, qty: c.qty + 1 } : c));
  return [...cart, {
    menuId: menu.id,
    name: menu.name,
    photo: menu.photo || null,
    emoji: menu.emoji || '☕',
    price: menu.price || 0,
    qty: 1,
  }];
}

function cartSetQty(cart, menuId, qty) {
  if (qty <= 0) return cart.filter((c) => c.menuId !== menuId);
  return cart.map((c) => (c.menuId === menuId ? { ...c, qty } : c));
}

function cartTotal(cart) {
  return cart.reduce((sum, c) => sum + (c.price || 0) * c.qty, 0);
}
```

- [ ] **Step 4: 통과 확인** — `PASS 19/19`

- [ ] **Step 5: 주문 화면 구현**

`VIEWS.order`를 2단 레이아웃(왼쪽 메뉴 타일 / 오른쪽 담은 내용)으로 만든다.

**메뉴 타일** — 접근성 원칙 1·2·3번이 모두 걸린 곳이다.

```html
<button class="tile" data-menu-id="...">
  <div class="tile-photo"><img src="{photo}"> 또는 <span class="emoji">☕</span></div>
  <div class="tile-name">아이스 아메리카노</div>
  <!-- 품절이면 -->
  <div class="tile-soldout" aria-hidden="true">✕</div>
</button>
```

```css
.tile { min-width: 160px; min-height: 180px; border-radius: 20px; ... }
.menu-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(160px, 1fr)); gap: 14px; }
.tile-photo { aspect-ratio: 1; }                    /* 정사각 */
.tile-photo img { width: 100%; height: 100%; object-fit: cover; border-radius: 14px; }
.tile-name { font-size: calc(19px * var(--scale)); font-weight: 900; }
/* 품절: 색만으로 구분하지 않는다 — 흐림 + 큰 ✕ */
.tile.soldout { opacity: .35; }
.tile.soldout .tile-soldout { display: grid; font-size: 72px; color: var(--danger); }
```

- 품절 메뉴는 `disabled`로 둔다 (누를 수 없는 버튼을 누르게 두지 않는다)
- `visible === false`인 메뉴는 아예 그리지 않는다
- 탭 시 `cartAdd` + 짧은 효과음/진동: `navigator.vibrate?.(30)`

**담은 내용** — 사진 카드로 크게. 각 카드에 `－ 수량 ＋`, 삭제(✕).
`전체 비우기`는 `confirmModal`.

**금액 표시** — `state.settings.priceMode === 'off'`이면 가격·합계를 **그리지 않는다** (DOM에서 제외).
Task 17에서 `'show'`, `'practice'`를 붙인다.

**주문 완료**

```js
async function submitOrder() {
  if (cart.length === 0) return;
  const businessDate = businessDateOf();
  const orderNo = await state.backend.issueOrderNo(businessDate);   // 트랜잭션
  const staff = state.settings.todayStaff || {};
  // 반환된 id는 Task 13(재료 차감 기록)에서 stockLogs.orderId로 쓴다
  const orderId = await state.backend.add('orders', {
    orderNo,
    businessDate,
    items: cart,
    total: cartTotal(cart),
    status: 'waiting',
    cashier: staff.cashier || '',      // 기록만 해 둔다 (이번 버전에서는 집계하지 않는다)
    barista: staff.barista || '',
    createdAt: Date.now(),
  });
  showOrderNumber(orderNo);            // 초대형 표시 + 음성 안내, 3초 뒤 자동 복귀
  cart = [];
}
```

**번호 표시 화면** — 화면을 덮는 큰 숫자 + `{번호}번이에요` 음성. 3초 후 자동으로 주문 화면 복귀.
접속 직후 음성이 막히는 브라우저가 있으므로, 실패해도 조용히 넘어간다 (`try/catch`).

- [ ] **Step 6: 화면 확인**

관리 화면에서 메뉴 3개(사진 1개, 이모지 2개)를 등록한 뒤 주문 역할로 전환.

| 확인할 것 | 기대 |
|---|---|
| 타일 | 사진(또는 이모지) + 이름이 **함께** 보인다 |
| 타일 탭 | 오른쪽에 사진 카드로 담긴다. 다시 탭하면 수량 2 |
| `－ ＋` | 수량이 바뀐다. 0이 되면 카드가 사라진다 |
| 품절 메뉴 | 흐려지고 큰 ✕. 눌러도 아무 일도 안 일어난다 |
| 주문 완료 | 큰 번호가 뜨고 3초 뒤 빈 주문 화면으로 돌아온다 |
| 번호 | 첫 주문이 **1번**, 다음이 2번 |
| 금액 | 어디에도 보이지 않는다 (`priceMode: 'off'`) |
| 글자 크기 `아주 크게` | 타일 이름과 담은 내용 글자가 같이 커지고 레이아웃이 깨지지 않는다 |

- [ ] **Step 7: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 주문 화면 (사진+글자 타일, 장바구니, 번호 발번)

품절 메뉴는 흐림과 큰 ✕로 표시하고 비활성화한다 — 색에만 의존하지 않고,
누를 수 없는 버튼을 누르게 두지 않는다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 8: 제조 화면

PRD 3.2.

**Files:**
- Modify: `index.html` (2번 · 5번 · 6번 구획)

**Interfaces:**
- Consumes: `state`, `backend.set`, `toast`, `CONFIG.undoSeconds`
- Produces:
  - `waitingOrders(orders) → order[]` — `status === 'waiting'`, `createdAt` 오름차순 (순수 함수)
  - `isLate(order, now, lateMinutes) → boolean` (순수 함수)
  - `VIEWS.barista.render(el)`

- [ ] **Step 1: 실패하는 테스트 추가**

```js
// ---- 제조 대기 목록 ----
test('대기 주문만 오래된 순으로 보여준다', () => {
  const orders = [
    { id: '3', status: 'waiting',   createdAt: 300 },
    { id: '1', status: 'waiting',   createdAt: 100 },
    { id: '2', status: 'ready',     createdAt: 200 },
    { id: '4', status: 'cancelled', createdAt: 400 },
  ];
  eq(waitingOrders(orders).map((o) => o.id), ['1', '3']);
});

test('주문번호가 겹쳐도 createdAt 기준으로 줄을 세운다', () => {
  // 주문번호 초기화 후에는 같은 날 1번이 두 개 있을 수 있다
  const orders = [
    { id: 'new', status: 'waiting', orderNo: 1, createdAt: 900 },
    { id: 'old', status: 'waiting', orderNo: 1, createdAt: 100 },
  ];
  eq(waitingOrders(orders).map((o) => o.id), ['old', 'new']);
});

test('기준 시간을 넘긴 주문을 지연으로 본다', () => {
  const now = 10 * 60 * 1000;
  eq(isLate({ createdAt: 0 }, now, 5), true,  '10분 경과 > 기준 5분');
  eq(isLate({ createdAt: 8 * 60 * 1000 }, now, 5), false, '2분 경과 < 기준 5분');
});
```

- [ ] **Step 2: 실패 확인** — `FAIL 19/22`

- [ ] **Step 3: 2번 구획에 구현 작성**

```js
// ---- 제조 대기 목록 ----
// 정렬은 반드시 createdAt 기준이다.
// 주문번호는 초기화(3.6.1) 때문에 같은 날 중복될 수 있으므로 정렬 키로 쓰면 안 된다.
function waitingOrders(orders) {
  return orders
    .filter((o) => o.status === 'waiting')
    .sort((a, b) => a.createdAt - b.createdAt);
}

function isLate(order, now, lateMinutes) {
  return now - order.createdAt > lateMinutes * 60 * 1000;
}
```

- [ ] **Step 4: 통과 확인** — `PASS 22/22`

- [ ] **Step 5: 제조 화면 구현**

카드 그리드. 카드 하나에:

```html
<div class="order-card late?">
  <div class="order-no">3</div>
  <div class="order-items">
    <div class="oi"><img src="{photo}"><span>아이스 아메리카노</span><b>×2</b></div>
  </div>
  <div class="order-elapsed">4분 경과</div>
  <button class="btn ok big">완료</button>
</div>
```

- 경과 시간은 10초마다 갱신한다 (`setInterval`). 역할이 바뀌면 `clearInterval`
- 지연 카드는 **색 + 두꺼운 테두리 + ⏰ 아이콘** (색에만 의존하지 않는다)
- 카드마다 메뉴 **품절 토글** 단축 버튼을 둔다 (재료가 떨어진 걸 먼저 아는 사람이 바리스타)

**완료 + 되돌리기**

```js
async function markReady(order) {
  await state.backend.set('orders', order.id, { status: 'ready', readyAt: Date.now() });
  toast(`${order.orderNo}번 호출했어요`, {
    actionText: '되돌리기',
    ms: CONFIG.undoSeconds * 1000,
    onAction: () => state.backend.set('orders', order.id, { status: 'waiting', readyAt: null }),
  });
}
```

- [ ] **Step 6: 화면 확인**

탭 두 개를 띄운다 — 하나는 주문, 하나는 제조.

| 확인할 것 | 기대 |
|---|---|
| 주문 탭에서 주문 완료 | 제조 탭에 **자동으로** 카드가 나타난다 (새로고침 없이) |
| 카드 | 번호가 가장 크고, 메뉴 사진과 수량이 보인다 |
| 여러 건 | 오래된 주문이 위에 있다 |
| 지연 (관리에서 기준을 1분으로 낮춰 확인) | 1분 후 카드에 색 + 테두리 + ⏰ |
| `완료` | 카드가 사라지고 `되돌리기` 토스트가 6초간 뜬다 |
| `되돌리기` | 카드가 다시 나타난다 |
| 품절 토글 | 주문 탭의 해당 타일이 바로 흐려진다 |

- [ ] **Step 7: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 제조 화면 (대기 카드, 지연 강조, 완료·되돌리기)

정렬은 createdAt 기준이다. 주문번호는 초기화로 중복될 수 있어
정렬 키로 쓰지 않는다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 9: 호출 화면

PRD 3.3. 손님이 보는 화면이므로 조작 요소가 없다.

**Files:**
- Modify: `index.html` (2번 · 5번 · 6번 구획)

**Interfaces:**
- Consumes: `state`, `CONFIG.displayRecentCount`
- Produces:
  - `recentCalls(orders, count) → order[]` — `status === 'ready'`, `readyAt` 내림차순 (순수 함수)
  - `sinoKorean(n) → string` — 숫자를 한자어 수사 한글로 (순수 함수)
  - `callText(template, orderNo) → string` — **화면용**, 숫자 그대로 (순수 함수)
  - `speechText(template, orderNo) → string` — **음성용**, 한자어 수사로 (순수 함수)
  - `playChime()`, `speak(text)`
  - `VIEWS.display.render(el)`

> **왜 음성용을 따로 두는가** — 한국어 TTS는 `3번`을 세는 단위로 읽어 **[세 번]**으로
> 발음한다. 주문번호는 횟수가 아니라 이름표이므로 **[삼 번]**이 맞다.
> 화면에는 숫자를 크게 보여주고(학생·손님이 식별해야 한다), 음성으로 넘길 때만
> 한글 수사로 바꾼다.

- [ ] **Step 1: 실패하는 테스트 추가**

```js
// ---- 호출 ----
test('최근 호출을 최신 순으로 자른다', () => {
  const orders = [
    { id: 'a', status: 'ready', readyAt: 100 },
    { id: 'b', status: 'ready', readyAt: 300 },
    { id: 'c', status: 'waiting', readyAt: null },
    { id: 'd', status: 'ready', readyAt: 200 },
  ];
  eq(recentCalls(orders, 2).map((o) => o.id), ['b', 'd']);
});

test('호출 문구의 {번호}를 바꾼다', () => {
  eq(callText('{번호}번 손님, 음료 나왔습니다.', 7), '7번 손님, 음료 나왔습니다.');
  eq(callText('{번호}번요! {번호}번!', 3), '3번요! 3번!', '여러 번 나와도 모두 바꾼다');
  eq(callText('음료 나왔습니다.', 3), '음료 나왔습니다.', '{번호}가 없어도 그대로 쓴다');
});

// ---- 숫자를 한자어 수사로 (음성용) ----
test('한 자리 수를 읽는다', () => {
  eq(sinoKorean(1), '일');
  eq(sinoKorean(3), '삼');
  eq(sinoKorean(6), '육');
  eq(sinoKorean(9), '구');
});

test('십 단위에서 앞의 1을 생략한다', () => {
  eq(sinoKorean(10), '십', '십일이 아니라 십');
  eq(sinoKorean(11), '십일');
  eq(sinoKorean(15), '십오');
  eq(sinoKorean(20), '이십');
  eq(sinoKorean(25), '이십오');
  eq(sinoKorean(99), '구십구');
});

test('백·천 단위도 같은 규칙을 따른다', () => {
  eq(sinoKorean(100), '백', '일백이 아니라 백');
  eq(sinoKorean(101), '백일');
  eq(sinoKorean(110), '백십');
  eq(sinoKorean(120), '백이십');
  eq(sinoKorean(999), '구백구십구');
  eq(sinoKorean(1000), '천');
  eq(sinoKorean(1234), '천이백삼십사');
});

test('음성 문구는 번호를 한글 수사로 바꾼다', () => {
  eq(speechText('{번호}번 손님, 음료 나왔습니다.', 3),
     '삼번 손님, 음료 나왔습니다.', 'TTS가 [세 번]으로 읽지 않게 한다');
  eq(speechText('{번호}번 손님, 음료 나왔습니다.', 25),
     '이십오번 손님, 음료 나왔습니다.');
  eq(speechText('음료 나왔습니다.', 3), '음료 나왔습니다.');
});
```

- [ ] **Step 2: 실패 확인** — `FAIL 22/28`

- [ ] **Step 3: 2번 구획에 구현 작성**

```js
// ---- 호출 ----
function recentCalls(orders, count) {
  return orders
    .filter((o) => o.status === 'ready' && o.readyAt)
    .sort((a, b) => b.readyAt - a.readyAt)
    .slice(0, count);
}

// 화면용 — 숫자를 그대로 쓴다. 손님과 학생이 번호표와 눈으로 맞춰야 한다.
function callText(template, orderNo) {
  return String(template).split('{번호}').join(String(orderNo));
}

// ---- 숫자를 한자어 수사로 (음성용) ----
// 한국어 TTS는 '3번'을 세는 단위로 보고 [세 번]이라고 읽는다.
// 주문번호는 횟수가 아니라 이름표이므로 [삼 번]이 맞다.
const SINO_DIGITS = ['', '일', '이', '삼', '사', '오', '육', '칠', '팔', '구'];
const SINO_UNITS = ['', '십', '백', '천'];

function sinoKorean(n) {
  const num = Math.floor(Math.abs(Number(n) || 0));
  if (num === 0) return '영';

  let out = '';
  const digits = String(num).split('').reverse();   // 1의 자리부터
  for (let i = digits.length - 1; i >= 0; i--) {
    const d = Number(digits[i]);
    if (d === 0) continue;
    // 십·백·천 앞의 1은 읽지 않는다 (10 → '십', 100 → '백')
    const head = (d === 1 && i > 0) ? '' : SINO_DIGITS[d];
    out += head + (SINO_UNITS[i] || '');
  }
  return out;
}

// 음성용 — 번호만 한글 수사로 바꾼다.
function speechText(template, orderNo) {
  return String(template).split('{번호}').join(sinoKorean(orderNo));
}
```

> `sinoKorean`은 만 단위 이상을 다루지 않는다. 하루 주문번호가 9,999를 넘을 일이 없고,
> 넘더라도 `SINO_UNITS`가 없는 자리는 단위 없이 읽혀 알아들을 수는 있다.

- [ ] **Step 4: 통과 확인** — `PASS 28/28`

- [ ] **Step 5: 호출 화면 구현**

```js
// 차임음 — 오디오 파일 없이 WebAudio로 두 음을 낸다 (기내 안내방송 느낌)
function playChime() {
  if (!state.settings.chime) return;
  const ctx = (window.__ac ||= new (window.AudioContext || window.webkitAudioContext)());
  const now = ctx.currentTime;
  for (const [i, freq] of [880, 660].entries()) {
    const osc = ctx.createOscillator(), gain = ctx.createGain();
    osc.frequency.value = freq;
    osc.type = 'sine';
    gain.gain.setValueAtTime(0.0001, now + i * 0.35);
    gain.gain.exponentialRampToValueAtTime(0.3, now + i * 0.35 + 0.03);
    gain.gain.exponentialRampToValueAtTime(0.0001, now + i * 0.35 + 0.5);
    osc.connect(gain).connect(ctx.destination);
    osc.start(now + i * 0.35);
    osc.stop(now + i * 0.35 + 0.55);
  }
}

function speak(text) {
  try {
    const u = new SpeechSynthesisUtterance(text);
    u.lang = 'ko-KR';
    u.rate = state.settings.ttsRate;
    u.volume = state.settings.ttsVolume;
    speechSynthesis.speak(u);
  } catch { /* 음성이 없는 기기에서는 조용히 넘어간다 */ }
}
```

**화면 구성**
- 상단: `달보드레` (큰 제목)
- 중앙: 가장 최근 호출 번호를 **초대형**으로 (`font-size: clamp(120px, 40vmin, 400px)`), 확대·깜빡임 애니메이션
- 하단: 직전 호출 번호 3~4개를 작게 나열
- 첫 진입 시 **"화면을 한 번 눌러 주세요"** 오버레이 — 브라우저 자동재생 정책상 사용자 제스처가 있어야 소리가 난다. 누르면 오버레이가 사라지고 전체화면 요청

**새 호출 감지** — 구독 콜백에서 `readyAt`이 가장 큰 주문의 id가 바뀌면 연출한다.

```js
let lastCalledId = null;
function onDisplayData() {
  const [top] = recentCalls(state.orders, 1);
  if (top && top.id !== lastCalledId) {
    lastCalledId = top.id;
    playChime();
    setTimeout(() => speak(speechText(state.settings.ttsText, top.orderNo)), 900);
  }
}
```

첫 로딩 때 과거 주문으로 갑자기 호출이 울리지 않도록, 구독 첫 콜백에서는 `lastCalledId`만 채우고 연출을 건너뛴다.

- [ ] **Step 5-1: Task 7의 주문 완료 음성도 함께 고친다**

Task 7이 주문 완료 후 말하는 `` `${orderNo}번이에요` `` 도 같은 문제를 갖는다
(TTS가 [세 번]으로 읽는다). 이 태스크에서 `sinoKorean`을 만들었으므로 그 호출 지점도
함께 바꾼다.

```js
// 변경 전: new SpeechSynthesisUtterance(`${orderNo}번이에요`)
// 변경 후:
new SpeechSynthesisUtterance(`${sinoKorean(orderNo)}번이에요`)
```

화면에 크게 뜨는 숫자는 그대로 둔다 — 학생과 손님이 번호표와 눈으로 맞춰야 한다.

- [ ] **Step 6: 화면 확인**

탭 세 개 — 주문 / 제조 / 호출.

| 확인할 것 | 기대 |
|---|---|
| 호출 화면 첫 진입 | "화면을 한 번 눌러 주세요" 안내. 누르면 사라진다 |
| 제조에서 `완료` | 호출 화면에 차임음 → 음성 → 번호가 크게 뜬다 |
| 문구 | "3번 손님, 음료 나왔습니다." |
| 여러 번 호출 | 아래에 직전 번호들이 작게 쌓인다 |
| 호출 화면 새로고침 | 과거 주문 때문에 소리가 울리지 않는다 |
| 상단바 | 호출 화면에는 없다 |

- [ ] **Step 7: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 호출 화면 (초대형 번호, WebAudio 차임음, 한국어 TTS)

첫 구독 콜백에서는 연출을 건너뛰어 새로고침 시 과거 주문이
갑자기 호출되지 않게 한다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 10: 주문번호 초기화

PRD 3.6.1. 판정 로직은 Task 2에서 이미 테스트까지 끝냈다. 여기서는 화면과 이력 기록을 붙인다.

**Files:**
- Modify: `index.html` (5번 구획)

**Interfaces:**
- Consumes: `resetPlan`, `businessDateOf`, `waitingOrders`, `confirmModal`, `backend.resetOrderNo`, `backend.add`
- Produces: `resetOrderNoFlow() → Promise<void>` (관리 화면 설정 탭의 버튼이 호출)

- [ ] **Step 1: 관리 화면 설정 탭에 초기화 UI 추가**

```html
<div class="card">
  <h3>주문번호 초기화</h3>
  <p class="muted">
    날이 바뀌면 저절로 1번부터 시작합니다.
    연습 주문을 지우고 시작하거나, 번호가 너무 커졌을 때 여기서 되돌립니다.
  </p>
  <p>지금 번호: <b id="reset-current">—</b></p>
  <label>몇 번부터 시작할까요? <input type="number" id="reset-start" value="1" min="1" step="1"></label>
  <label class="check">
    <input type="checkbox" id="reset-clear-display" checked>
    호출 화면의 최근 호출 목록도 비우기
  </label>
  <button class="btn danger" id="btn-reset-no">주문번호 다시 시작</button>
</div>
```

- [ ] **Step 2: 초기화 흐름 구현**

```js
async function resetOrderNoFlow() {
  const businessDate = businessDateOf();
  const startFrom = Number(document.getElementById('reset-start').value);
  const clearDisplay = document.getElementById('reset-clear-display').checked;
  const waiting = waitingOrders(state.orders.filter((o) => o.businessDate === businessDate));

  // 카운터는 구독 대상이 아니므로 그때그때 읽는다
  const counterValue = await state.backend.getCounter(businessDate);
  const plan = resetPlan(counterValue, startFrom, waiting.length);

  if (!plan.ok) { toast(plan.message); return; }

  // 대기 주문이 있으면 여기서 한 번 더 확인받는다 (같은 번호가 두 번 불릴 수 있다)
  const ok = await confirmModal({
    title: plan.needsConfirm ? '번호가 겹칠 수 있어요' : '주문번호를 되돌릴까요?',
    msg: plan.message,
    okText: '되돌리기',
    okClass: plan.needsConfirm ? 'danger' : 'primary',
  });
  if (!ok) return;

  await state.backend.resetOrderNo(businessDate, plan.newCounterValue);

  // 이력을 남긴다 — 나중에 "왜 3번이 두 개지?"에 답할 근거가 된다
  await state.backend.add('counterResets', {
    businessDate,
    beforeValue: counterValue,
    startFrom,
    waitingCount: waiting.length,
    clearedDisplay: clearDisplay,
    createdAt: Date.now(),
  });

  if (clearDisplay) {
    // 호출 화면을 비운다 — 주문 기록은 지우지 않고 표시만 끈다
    for (const o of state.orders.filter((x) => x.status === 'ready')) {
      await state.backend.set('orders', o.id, { hiddenFromDisplay: true });
    }
  }

  toast(`주문번호를 ${startFrom}번부터 다시 시작합니다.`);
}
```

`recentCalls`가 `hiddenFromDisplay`를 걸러내도록 Task 9의 필터를 고친다:

```js
function recentCalls(orders, count) {
  return orders
    .filter((o) => o.status === 'ready' && o.readyAt && !o.hiddenFromDisplay)
    .sort((a, b) => b.readyAt - a.readyAt)
    .slice(0, count);
}
```

Task 9의 테스트에 한 줄을 더한다.

```js
test('호출 화면에서 숨긴 주문은 빼고 보여준다', () => {
  const orders = [
    { id: 'a', status: 'ready', readyAt: 100 },
    { id: 'b', status: 'ready', readyAt: 300, hiddenFromDisplay: true },
  ];
  eq(recentCalls(orders, 5).map((o) => o.id), ['a']);
});
```

카운터 현재 값은 화면에 보여야 하므로 백엔드에 조회 메서드를 하나 더한다.

```js
// 로컬 백엔드에 추가
api.getCounterSync = (bd) => (load().counters || {})[bd] || 0;
api.getCounter = async (bd) => api.getCounterSync(bd);
```

- [ ] **Step 3: 자가 테스트 확인** — `PASS 29/29`

- [ ] **Step 4: 화면 확인**

| 확인할 것 | 기대 |
|---|---|
| 주문 3건을 넣고 모두 `완료` 처리 후 초기화 | 확인창 1단계 → 다음 주문이 1번 |
| 주문 2건을 대기 상태로 두고 초기화 | **"아직 받지 않은 주문이 2건 있어요…"** 경고, 확인 버튼이 빨간색 |
| 경고에서 취소 | 번호가 그대로다 |
| 시작 번호 `100` | 다음 주문이 100번 |
| 시작 번호 `0` 또는 빈칸 | "시작 번호는 1 이상의 정수여야 합니다" 토스트, 아무 일도 안 일어남 |
| `최근 호출 목록도 비우기` 체크 | 호출 화면이 비워진다. **관리 화면의 주문 기록은 그대로** |
| 초기화 후 제조 화면 | 기존 대기 주문이 사라지지 않고 그대로 있다 |

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 주문번호 초기화 (대기 주문 경고, 시작 번호 지정, 이력 기록)

판매 기록은 지우지 않고 카운터만 되돌린다.
번호가 중복됐을 때 추적할 수 있도록 counterResets에 이력을 남긴다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 11: Firestore 백엔드 · 보안 규칙 · README · 배포

PRD 6.2, 6.4. 여기까지 끝나면 **기기 3대로 실제 운영이 가능**하다.

**Files:**
- Modify: `index.html` (3번 · 7번 구획)
- Create: `firestore.rules`
- Create: `README.md`

**Interfaces:**
- Consumes: `CONFIG`, `COLLECTIONS`, `nextOrderNo`
- Produces: `async createFirestoreBackend() → backend` — Task 4와 **같은 인터페이스**, `mode: 'firestore'`

- [ ] **Step 1: Firestore 백엔드 작성**

```js
// ---- Firestore 모드 ----
// 경로: {dataRoot}/{spaceId}/{컬렉션}/{문서}
// 인스턴스 이름을 따로 줘서 같은 도메인의 다른 앱과 오프라인 캐시가 섞이지 않게 한다.
async function createFirestoreBackend() {
  const v = CONFIG.firebaseVersion;
  const { initializeApp } = await import(`https://www.gstatic.com/firebasejs/${v}/firebase-app.js`);
  const fs = await import(`https://www.gstatic.com/firebasejs/${v}/firebase-firestore.js`);

  const app = initializeApp(CONFIG.firebase, CONFIG.appName);

  // 오프라인 캐시 (PRD 6.4). 기기 여러 대가 같은 탭 구조로 열려 있으므로 멀티탭 매니저를 쓴다.
  // 구버전 API(enableIndexedDbPersistence)는 폐기되어 조용히 실패할 수 있으므로 신 API를 먼저 시도한다.
  let settings = {};
  if (fs.persistentLocalCache && fs.persistentMultipleTabManager) {
    settings.localCache = fs.persistentLocalCache({
      tabManager: fs.persistentMultipleTabManager(),
    });
  }
  const db = CONFIG.databaseId
    ? fs.initializeFirestore(app, settings, CONFIG.databaseId)
    : fs.initializeFirestore(app, settings);

  // 신 API가 없는 버전이면 구 API로 물러선다
  if (!settings.localCache && fs.enableIndexedDbPersistence) {
    try { await fs.enableIndexedDbPersistence(db); } catch { /* 탭이 여럿이면 실패할 수 있다 */ }
  }

  const col = (name) => fs.collection(db, CONFIG.dataRoot, CONFIG.spaceId, name);
  const ref = (name, id) => fs.doc(db, CONFIG.dataRoot, CONFIG.spaceId, name, id);
  const counterRef = (bd) => ref('counters', bd);

  return {
    mode: 'firestore',

    subscribe(name, cb) {
      return fs.onSnapshot(col(name), (snap) => {
        cb(snap.docs.map((d) => ({ id: d.id, ...d.data() })));
      });
    },

    async add(name, data) {
      const d = await fs.addDoc(col(name), data);
      return d.id;
    },

    // 필드 단위 merge — 문서 전체를 덮어쓰지 않는다
    async set(name, id, patch) {
      await fs.setDoc(ref(name, id), patch, { merge: true });
    },

    async remove(name, id) {
      await fs.deleteDoc(ref(name, id));
    },

    // 재고 증감은 원자적으로 — 두 기기가 동시에 팔아도 유실되지 않는다
    async bump(name, id, fields) {
      const patch = {};
      for (const [k, delta] of Object.entries(fields)) patch[k] = fs.increment(delta);
      await fs.setDoc(ref(name, id), patch, { merge: true });
    },

    // 번호 중복을 막으려면 트랜잭션이어야 한다 (인터넷 연결 필요)
    async issueOrderNo(businessDate) {
      return fs.runTransaction(db, async (tx) => {
        const snap = await tx.get(counterRef(businessDate));
        const value = nextOrderNo(snap.exists() ? snap.data().value : 0);
        tx.set(counterRef(businessDate), { value }, { merge: true });
        return value;
      });
    },

    async resetOrderNo(businessDate, value) {
      await fs.setDoc(counterRef(businessDate), { value }, { merge: true });
    },

    async getCounter(businessDate) {
      const snap = await fs.getDoc(counterRef(businessDate));
      return snap.exists() ? snap.data().value : 0;
    },

    async clear() {
      // 초기화도 이 경로 안에서만 동작한다
      for (const name of COLLECTIONS.concat('counters')) {
        const snap = await fs.getDocs(col(name));
        await Promise.all(snap.docs.map((d) => fs.deleteDoc(d.ref)));
      }
    },
  };
}
```

- [ ] **Step 2: 7번 구획에서 모드를 자동 선택**

```js
// Firebase 키가 있으면 클라우드, 없으면 로컬 (PRD 6.2)
state.backend = CONFIG.firebase.apiKey
  ? await createFirestoreBackend()
  : createLocalBackend();

// 상단바에 모드를 표시한다 — 선생님이 "지금 연결돼 있나?"를 알 수 있어야 한다
document.getElementById('net-tag').textContent =
  state.backend.mode === 'local' ? '연습 모드' : '연결됨';
```

연결이 끊겼을 때 주문 확정이 실패하면 토스트로 알린다.

```js
try {
  await submitOrder();
} catch (err) {
  toast('인터넷 연결을 확인해 주세요. 주문번호를 받지 못했습니다.', { ms: 5000 });
}
```

- [ ] **Step 3: `firestore.rules` 작성**

```
// ---- 달보드레 카페 POS ----
// 기존 규칙을 덮어쓰지 말고, match /databases/{database}/documents { 안쪽
// 닫는 } 바로 위에 아래 블록만 추가한다.
// Firestore 규칙은 "하나라도 허용하면 허용"이므로 기존 앱 규칙은 그대로 유지된다.
match /dallbodrae_pos/{spaceId}/{collection}/{docId} {
  allow read, write: if collection in
    ['menus', 'ingredients', 'orders', 'stockLogs', 'counterResets', 'counters', 'settings'];
}
```

- [ ] **Step 4: `README.md` 작성**

두 독자를 나눠 쓴다.

```markdown
# 달보드레 카페 POS

특수학급 카페 '달보드레'의 주문·제조·호출·기록을 관리하는 웹앱.
`index.html` 하나로 동작합니다.

---

## 선생님께 — 매일 쓰는 법

### 기기 배치
| 기기 | 역할 |
|---|---|
| 태블릿 1 | 주문 (캐셔 학생) |
| 태블릿 2 | 제조 (바리스타 학생) |
| 모니터 | 호출 (손님이 보는 화면 — 시작할 때 화면을 **한 번 터치**해야 소리가 납니다) |
| 선생님 기기 | 관리 |

### 처음 한 번만
1. 각 기기에서 주소를 열고 역할을 고릅니다. 역할은 기억되므로 다시 고를 필요가 없습니다.
2. 관리 → PIN `1234` → 메뉴를 등록합니다 (이름 · 사진).
3. 관리 → 설정에서 PIN을 바꿉니다.

### 메뉴 바꾸기
관리 → 메뉴 → `＋ 메뉴 추가`. 사진은 휴대폰으로 찍은 걸 그대로 올리면 앱이 알아서 줄입니다.

### 오늘 재료가 떨어졌을 때
관리 또는 제조 화면에서 그 메뉴의 `품절`을 켭니다. 주문 화면에서 흐려지고 눌리지 않습니다.

### 주문번호를 다시 1번부터
관리 → 설정 → `주문번호 다시 시작`.
아직 받아가지 않은 주문이 있으면 경고가 뜹니다 — 같은 번호가 두 번 불릴 수 있기 때문입니다.
**판매 기록은 지워지지 않습니다.**

### 글자가 작아 보일 때
화면 오른쪽 위 `가 가 가` 버튼으로 세 단계까지 키울 수 있습니다. 기기마다 따로 기억됩니다.

---

## 관리자용 — 설치와 배포

### 연습 모드로 먼저 확인하기
`CONFIG.firebase.apiKey`가 비어 있으면 로컬 모드로 동작합니다.
같은 기기의 탭끼리만 동기화되므로 화면 확인·연습용입니다.

```bash
python -m http.server 8320
```

### 실제 운영 — 기존 Firebase 프로젝트에 앱 추가

1. **이름 충돌 확인** — Firebase 콘솔 → Firestore Database → 데이터 탭에서
   최상위에 `dallbodrae_pos` 컬렉션이 **없는지** 확인합니다.
   이미 있다면 `index.html`의 `CONFIG.dataRoot`와 아래 규칙의 이름을 함께 바꿉니다.
2. **웹 앱 추가** — 프로젝트 설정 → 일반 → 내 앱 → `</>`, 이름 예: `dallbodrae-pos`.
   Firebase Hosting 체크는 **하지 않습니다.**
   나오는 `firebaseConfig` 값을 `index.html` 상단 `CONFIG.firebase`에 붙여넣습니다.
3. **보안 규칙에 블록 추가 (기존 내용은 그대로 두기!)** — Firestore Database → 규칙 탭.
   기존 규칙의 `match /databases/{database}/documents {` 안쪽, 닫는 `}` 바로 위에
   `firestore.rules`의 블록**만** 붙여넣고 게시합니다.
   - Firestore 규칙은 "하나라도 허용하면 허용"이므로 기존 앱 규칙은 바뀌지 않습니다.
   - 게시 전 **규칙 플레이그라운드**로 기존 앱 경로의 읽기/쓰기가 그대로인지 확인하세요.
   - ⚠️ `firebase deploy`로 규칙을 배포하는 다른 앱이 있다면, 그 프로젝트의
     `firestore.rules` 파일에도 같은 블록을 넣어야 다음 배포 때 지워지지 않습니다.
4. **(해당 시) API 키 제한** — 브라우저 키에 HTTP 리퍼러 제한이 걸려 있다면
   `https://<GitHub아이디>.github.io/*`를 **추가**합니다 (기존 항목 삭제 금지).
5. **(해당 시) App Check** — Firestore App Check를 "적용"으로 켜 두었다면 이 앱 요청이 거부됩니다.
   기존 앱에 영향이 가므로 끄지 말고 별도로 대응하세요.
6. GitHub 저장소에 `index.html` 업로드 → Settings → Pages → `main` 브랜치 배포
7. 관리 → 설정에서 상단에 **`연결됨`**, 저장 위치가 `dallbodrae_pos/cafe-2026`인지 확인
8. 메뉴·재료 등록 → 리허설 → **기록 초기화** → 운영 시작

> ⚠️ 로그인 없이 운영하므로 주소와 설정값을 아는 사람은 `dallbodrae_pos` 경로를
> 읽고 쓸 수 있습니다 (다른 앱 경로는 해당 없음). 관리 화면 PIN은 학생의 실수를 막는
> 장치이지 보안 장치가 아닙니다.

### 무료 한도
하루 읽기 5만 / 쓰기 2만 건. 하루 주문 30건 · 기기 4대 기준으로는 여유가 큽니다.
사진을 문서에 넣으므로 메뉴를 20개 이상으로 늘릴 때만 사용량을 한 번 확인하세요.

### 새 학년도에 새로 시작하기
`CONFIG.spaceId`를 바꾸면 (예: `cafe-2026` → `cafe-2027`) 이전 데이터와 섞이지 않고
빈 상태로 시작합니다. 이전 기록은 그대로 남아 있습니다.

### 앱이 멀쩡한지 확인
주소 뒤에 `?selftest=1`을 붙이면 내부 계산 로직을 자가 점검합니다. `PASS n/n`이면 정상입니다.

---

Made by 정우쌤
```

- [ ] **Step 5: 로컬 모드가 깨지지 않았는지 확인**

`CONFIG.firebase.apiKey`를 비운 채로 `http://localhost:8320/` → 상단바에 `연습 모드`.
Task 5~10의 확인 항목을 한 번 더 훑는다.

- [ ] **Step 6: Firestore 모드 확인**

Firebase 웹 앱을 추가하고 `CONFIG.firebase`를 채운 뒤, 규칙 블록을 게시한다.

| 확인할 것 | 기대 |
|---|---|
| 상단바 | `연결됨` |
| **다른 기기**(또는 시크릿 창)에서 주문 | 제조 화면에 나타난다 |
| Firebase 콘솔 | `dallbodrae_pos/cafe-2026/menus` 아래에만 데이터가 있다. 기존 앱 경로는 그대로 |
| 기존 앱 | 여전히 정상 동작한다 (규칙 플레이그라운드로 읽기/쓰기 확인) |
| 와이파이 끄고 주문 완료 | "인터넷 연결을 확인해 주세요" 토스트. 화면은 캐시로 유지된다 |
| 와이파이 복구 | 자동으로 다시 붙는다 |

- [ ] **Step 7: GitHub Pages 배포 후 커밋**

```bash
git add index.html firestore.rules README.md
git commit -m "$(cat <<'MSG'
feat: Firestore 백엔드·보안 규칙·README

로컬 모드와 같은 6개 메서드 인터페이스를 구현했다.
데이터는 dallbodrae_pos 경로 안에만 저장되어 기존 앱과 섞이지 않는다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

> **Phase 1 완료 지점.** 여기까지가 PRD의 P0이다. 메뉴를 등록하고, 주문받고, 만들고, 부르고, 번호를 되돌릴 수 있다. 실제 운영을 시작해도 되는 상태다.

---

# Phase 2 — P1: 재료 재고와 기록

---

### Task 12: 순수 로직 — 재료 차감 · 복구 · 장볼 목록

PRD 3.4. **레시피가 비어 있으면 아무것도 차감하지 않는다**는 점진 도입 원칙이 이 함수에 박힌다.

**Files:**
- Modify: `index.html` (2번 · 6번 구획)

**Interfaces:**
- Consumes: 없음 (순수 함수)
- Produces:
  - `stockDelta(items, menuMap) → { [ingredientId]: number }` — 판매는 음수
  - `invertDelta(delta) → delta` — 취소 복구용
  - `shoppingList(ingredients) → [{ id, name, unit, current, reorderPoint, shortage }]`
  - `menuLowStock(menu, ingredientMap) → boolean`

- [ ] **Step 1: 실패하는 테스트 추가**

```js
// ---- 재료 차감 ----
const MENUS_FOR_TEST = {
  latte: { id: 'latte', name: '라떼', recipe: [
    { ingredientId: 'milk', qty: 150 },
    { ingredientId: 'bean', qty: 18 },
    { ingredientId: 'cup',  qty: 1 },
  ]},
  ade: { id: 'ade', name: '에이드', recipe: [] },   // 레시피 미등록
};

test('레시피대로 수량만큼 차감한다', () => {
  eq(stockDelta([{ menuId: 'latte', qty: 2 }], MENUS_FOR_TEST),
     { milk: -300, bean: -36, cup: -2 });
});

test('레시피가 비어 있으면 아무것도 차감하지 않는다', () => {
  eq(stockDelta([{ menuId: 'ade', qty: 3 }], MENUS_FOR_TEST), {});
});

test('없는 메뉴는 건너뛴다', () => {
  eq(stockDelta([{ menuId: 'deleted', qty: 1 }], MENUS_FOR_TEST), {});
});

test('여러 메뉴가 같은 재료를 쓰면 합산한다', () => {
  const menus = {
    a: { recipe: [{ ingredientId: 'cup', qty: 1 }] },
    b: { recipe: [{ ingredientId: 'cup', qty: 1 }] },
  };
  eq(stockDelta([{ menuId: 'a', qty: 2 }, { menuId: 'b', qty: 3 }], menus), { cup: -5 });
});

test('취소하면 부호가 뒤집힌다', () => {
  eq(invertDelta({ milk: -300, cup: -2 }), { milk: 300, cup: 2 });
});

// ---- 장볼 목록 ----
test('기준점 이하인 재료만 부족한 만큼과 함께 모은다', () => {
  const ing = [
    { id: 'milk', name: '우유', unit: 'ml', current: 200,  reorderPoint: 1000 },
    { id: 'bean', name: '원두', unit: 'g',  current: 900,  reorderPoint: 500 },
    { id: 'cup',  name: '컵',   unit: '개', current: 10,   reorderPoint: 10 },
  ];
  const list = shoppingList(ing);
  eq(list.map((x) => x.id), ['milk', 'cup'], '기준점과 같아도 포함한다');
  eq(list[0].shortage, 800);
});

test('기준점이 없는 재료는 장볼 목록에 넣지 않는다', () => {
  eq(shoppingList([{ id: 'x', name: '빨대', current: 0 }]).length, 0);
});

test('재료가 부족한 메뉴를 알아낸다', () => {
  const ingMap = { milk: { current: 100, reorderPoint: 1000 } };
  eq(menuLowStock({ recipe: [{ ingredientId: 'milk', qty: 150 }] }, ingMap), true);
  eq(menuLowStock({ recipe: [] }, ingMap), false, '레시피가 없으면 경고하지 않는다');
});
```

- [ ] **Step 2: 실패 확인** — `FAIL 29/37`

- [ ] **Step 3: 2번 구획에 구현 작성**

```js
// ---- 재료 차감 (PRD 3.4) ----
// 레시피가 비어 있으면 차감하지 않는다. 이게 "점진 도입"의 핵심이다 —
// 선생님이 레시피를 채우기 전까지 재고 기능은 조용히 꺼져 있다.
function stockDelta(items, menuMap) {
  const delta = {};
  for (const item of items) {
    const menu = menuMap[item.menuId];
    if (!menu || !Array.isArray(menu.recipe) || menu.recipe.length === 0) continue;
    for (const r of menu.recipe) {
      delta[r.ingredientId] = (delta[r.ingredientId] || 0) - r.qty * item.qty;
    }
  }
  return delta;
}

function invertDelta(delta) {
  return Object.fromEntries(Object.entries(delta).map(([k, v]) => [k, -v]));
}

// ---- 장볼 목록 ----
// 기준점을 정하지 않은 재료는 "관리하지 않겠다"는 뜻으로 본다.
function shoppingList(ingredients) {
  return ingredients
    .filter((i) => typeof i.reorderPoint === 'number' && (i.current || 0) <= i.reorderPoint)
    .map((i) => ({
      id: i.id, name: i.name, unit: i.unit,
      current: i.current || 0,
      reorderPoint: i.reorderPoint,
      shortage: i.reorderPoint - (i.current || 0),
    }));
}

function menuLowStock(menu, ingredientMap) {
  if (!Array.isArray(menu.recipe) || menu.recipe.length === 0) return false;
  return menu.recipe.some((r) => {
    const ing = ingredientMap[r.ingredientId];
    return ing && typeof ing.reorderPoint === 'number' && (ing.current || 0) <= ing.reorderPoint;
  });
}
```

- [ ] **Step 4: 통과 확인** — `PASS 37/37`

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 재료 차감·복구·장볼 목록 로직

레시피가 비어 있으면 차감하지 않는다 — 선생님이 레시피를 채우기 전까지
재고 기능은 꺼진 상태로 둔다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 13: 재료 관리 화면 · 레시피 · 입고 · 자동 차감

**Files:**
- Modify: `index.html` (5번 구획)

**Interfaces:**
- Consumes: `stockDelta`, `shoppingList`, `menuLowStock`, `backend.bump`, `backend.add`
- Produces: 관리 화면의 `재료` 탭, 메뉴 편집 모달의 `레시피` 구역

- [ ] **Step 1: 관리 화면에 `재료` 탭 추가**

- 재료 목록: 이름 / 현재 수량 / 단위 / 발주 기준점 / `입고` 버튼 / 수정 / 삭제
- `＋ 재료 추가` 모달: 이름, 단위(`ml` / `g` / `개` 선택), 현재 수량, 발주 기준점
- **`입고`** → 숫자 입력 모달 → `backend.bump('ingredients', id, { current: +n })` + `stockLogs`에 `type: 'restock'` 기록
- **장볼 목록** 카드: `shoppingList(state.ingredients)` 결과를 표로. `목록 복사` 버튼(클립보드) + `인쇄` 버튼(`window.print()`)

```js
function copyShoppingList() {
  const lines = shoppingList(state.ingredients)
    .map((x) => `${x.name} — ${x.shortage}${x.unit} 이상 (지금 ${x.current}${x.unit})`);
  navigator.clipboard.writeText(lines.join('\n'));
  toast('장볼 목록을 복사했어요');
}
```

- [ ] **Step 2: 메뉴 편집 모달에 레시피 구역 추가**

- `＋ 재료 넣기` → 재료 선택 + 1잔당 사용량 입력 → `recipe` 배열에 추가
- 비어 있을 때 안내: **"재료를 넣지 않으면 재고를 차감하지 않습니다. 나중에 채워도 됩니다."**

- [ ] **Step 3: `stockLogs` 구독 추가**

이 태스크부터 `stockLogs`에 기록을 쌓는다. Task 16(기록 화면)이 이 배열을 읽으므로
7번 구획의 구독 목록에 지금 추가한다 (Task 5에서는 `menus`·`ingredients`·`orders`만 구독했다).

```js
// 7번 구획 — 구독 목록에 stockLogs를 더한다
for (const name of ['menus', 'ingredients', 'orders', 'stockLogs']) {
  state.backend.subscribe(name, (docs) => { state[name] = docs; render(); });
}
```

- [ ] **Step 4: 주문 확정 시 자동 차감 연결**

Task 7의 `submitOrder`에 이어 붙인다.

```js
// 주문 저장 뒤 재료를 차감한다. 레시피가 없는 메뉴는 delta가 비어 아무 일도 없다.
const menuMap = Object.fromEntries(state.menus.map((m) => [m.id, m]));
const delta = stockDelta(cart, menuMap);
for (const [ingredientId, d] of Object.entries(delta)) {
  await state.backend.bump('ingredients', ingredientId, { current: d });
  await state.backend.add('stockLogs', {
    ingredientId, type: 'sale', qty: d, orderId, createdAt: Date.now(),
  });
}
```

- [ ] **Step 5: 주문 화면에 부족 경고 배지**

`menuLowStock(menu, ingredientMap)`가 참이면 타일 모서리에 `재료 부족` 배지. **타일은 계속 눌린다** (품절과 다르다 — 경고일 뿐이다).

- [ ] **Step 6: 자가 테스트 확인** — `PASS 37/37`

- [ ] **Step 7: 화면 확인**

| 확인할 것 | 기대 |
|---|---|
| 재료 `우유 1000ml, 기준점 300` 등록 | 목록에 보인다 |
| 라떼에 `우유 150ml` 레시피 등록 | 저장된다 |
| 라떼 2잔 주문 | 우유가 `700ml`로 줄어든다 |
| 레시피 없는 메뉴 주문 | 재료가 그대로다 |
| 우유를 250ml까지 소진 | 주문 화면 라떼 타일에 `재료 부족` 배지, **여전히 눌린다** |
| 장볼 목록 | 우유가 `50ml 이상`으로 뜬다 |
| `목록 복사` | 클립보드에 텍스트가 들어간다 |
| 입고 `1000` | 우유가 1250ml, 장볼 목록에서 사라진다 |

- [ ] **Step 8: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 재료 관리·레시피·입고·주문 시 자동 차감

재료 부족은 경고 배지로만 알리고 타일은 계속 눌리게 둔다.
판매를 막는 것은 품절 토글뿐이다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 14: 주문 취소 + 재료 복구

PRD 3.1.5.

**Files:**
- Modify: `index.html` (5번 구획)

**Interfaces:**
- Consumes: `invertDelta`, `stockDelta`, `confirmModal`, `backend.set`, `backend.bump`
- Produces: `cancelOrderFlow(orderId) → Promise<void>`, 주문 화면의 `최근 주문` 목록

- [ ] **Step 1: 주문 화면 하단에 최근 주문 목록 추가**

`CONFIG.recentOrderCount`(12)건. `createdAt` 내림차순. 각 행에 번호·메뉴 요약·상태 배지·`취소` 버튼.
이미 취소된 주문은 회색 + `취소됨` 배지, 버튼 없음.

- [ ] **Step 2: 취소 흐름 구현**

```js
async function cancelOrderFlow(orderId) {
  const order = state.orders.find((o) => o.id === orderId);
  if (!order || order.status === 'cancelled') return;

  const ok = await confirmModal({
    title: `${order.orderNo}번 주문을 취소할까요?`,
    msg: '재료는 자동으로 되돌아옵니다. 판매 기록에서는 빠집니다.',
    okText: '주문 취소', okClass: 'danger',
  });
  if (!ok) return;

  await state.backend.set('orders', orderId, { status: 'cancelled', cancelledAt: Date.now() });

  // 재료 복구 — 차감했던 것과 정확히 같은 양을 되돌린다
  const menuMap = Object.fromEntries(state.menus.map((m) => [m.id, m]));
  const back = invertDelta(stockDelta(order.items, menuMap));
  for (const [ingredientId, d] of Object.entries(back)) {
    await state.backend.bump('ingredients', ingredientId, { current: d });
    await state.backend.add('stockLogs', {
      ingredientId, type: 'restore', qty: d, orderId, createdAt: Date.now(),
    });
  }
  toast(`${order.orderNo}번 주문을 취소했어요`);
}
```

> 주의: 레시피가 주문 이후에 바뀌면 복구량이 어긋날 수 있다. 이번 버전에서는 현재 레시피 기준으로 되돌린다. 레시피를 바꾸는 일은 드물고, 바꿨다면 그날의 재고를 선생님이 한 번 맞춰 주는 편이 단순하다.

- [ ] **Step 3: 제조 화면에서도 취소 가능하게**

제조 대기 카드에 `취소` 버튼을 더한다. 같은 `cancelOrderFlow`를 부른다.

- [ ] **Step 4: 화면 확인**

| 확인할 것 | 기대 |
|---|---|
| 라떼 2잔 주문 후 취소 | 우유가 300ml 되돌아온다 |
| 취소한 주문 | 목록에서 회색 `취소됨`, 제조 화면에서 사라진다 |
| 취소 확인창에서 `취소` | 아무 일도 안 일어난다 |
| 이미 취소된 주문 | `취소` 버튼이 없다 |

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 주문 취소와 재료 자동 복구

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 15: 순수 로직 — 집계 · CSV

PRD 3.5.

**Files:**
- Modify: `index.html` (2번 · 6번 구획)

**Interfaces:**
- Produces:
  - `inRange(order, from, to) → boolean` — `from`/`to`는 `'YYYY-MM-DD'`, 양끝 포함
  - `summarize(orders, from, to) → { orderCount, dayCount, cancelledCount, byMenu, byHour, totalQty, totalAmount }`
    - `byMenu`: `[{ menuId, name, qty, amount }]` — qty 내림차순
    - `byHour`: `[{ hour, qty }]` — 0~23 전부, 판매 없는 시간은 0
  - `csvCell(v) → string`
  - `toCsv(rows) → string` — `rows`는 2차원 배열

- [ ] **Step 1: 실패하는 테스트 추가**

```js
// ---- 집계 ----
const ORDERS_FOR_TEST = [
  { id: '1', businessDate: '2026-09-20', status: 'ready', createdAt: new Date(2026,8,20,10,10).getTime(),
    items: [{ menuId: 'a', name: '아메리카노', price: 1000, qty: 2 }], total: 2000 },
  { id: '2', businessDate: '2026-09-20', status: 'ready', createdAt: new Date(2026,8,20,12,30).getTime(),
    items: [{ menuId: 'b', name: '라떼', price: 1500, qty: 1 }], total: 1500 },
  { id: '3', businessDate: '2026-09-21', status: 'waiting', createdAt: new Date(2026,8,21,10,5).getTime(),
    items: [{ menuId: 'a', name: '아메리카노', price: 1000, qty: 1 }], total: 1000 },
  { id: '4', businessDate: '2026-09-21', status: 'cancelled', createdAt: new Date(2026,8,21,10,6).getTime(),
    items: [{ menuId: 'a', name: '아메리카노', price: 1000, qty: 9 }], total: 9000 },
];

test('기간은 양끝을 포함한다', () => {
  eq(inRange(ORDERS_FOR_TEST[0], '2026-09-20', '2026-09-20'), true);
  eq(inRange(ORDERS_FOR_TEST[2], '2026-09-20', '2026-09-20'), false);
  eq(inRange(ORDERS_FOR_TEST[2], '2026-09-20', '2026-09-21'), true);
});

test('취소된 주문은 집계에서 빠진다', () => {
  const s = summarize(ORDERS_FOR_TEST, '2026-09-21', '2026-09-21');
  eq(s.orderCount, 1);
  eq(s.cancelledCount, 1);
  eq(s.totalQty, 1, '취소된 9잔은 세지 않는다');
});

test('메뉴별로 많이 팔린 순으로 모은다', () => {
  const s = summarize(ORDERS_FOR_TEST, '2026-09-20', '2026-09-21');
  eq(s.byMenu.map((m) => [m.menuId, m.qty]), [['a', 3], ['b', 1]]);
  eq(s.byMenu[0].amount, 3000);
});

test('운영일 수는 주문이 있었던 날만 센다', () => {
  eq(summarize(ORDERS_FOR_TEST, '2026-09-20', '2026-09-21').dayCount, 2);
  eq(summarize(ORDERS_FOR_TEST, '2026-09-20', '2026-09-20').dayCount, 1);
});

test('시간대별 분포는 0~23시를 모두 채운다', () => {
  const s = summarize(ORDERS_FOR_TEST, '2026-09-20', '2026-09-20');
  eq(s.byHour.length, 24);
  eq(s.byHour[10].qty, 2);
  eq(s.byHour[12].qty, 1);
  eq(s.byHour[0].qty, 0);
});

test('빈 기간도 안전하게 집계한다', () => {
  const s = summarize([], '2026-09-20', '2026-09-20');
  eq(s.orderCount, 0); eq(s.totalQty, 0); eq(s.byMenu, []); eq(s.dayCount, 0);
});

// ---- CSV ----
test('CSV 셀에 쉼표·따옴표·줄바꿈이 있으면 감싼다', () => {
  eq(csvCell('아메리카노'), '아메리카노');
  eq(csvCell('우유, 원두'), '"우유, 원두"');
  eq(csvCell('12"'), '"12"""');
  eq(csvCell('한 줄\n두 줄'), '"한 줄\n두 줄"');
  eq(csvCell(null), '');
  eq(csvCell(0), '0');
});

test('CSV는 줄바꿈으로 이어붙인다', () => {
  eq(toCsv([['메뉴', '잔'], ['라떼', 3]]), '메뉴,잔\n라떼,3');
});
```

- [ ] **Step 2: 실패 확인** — `FAIL 37/45`

- [ ] **Step 3: 2번 구획에 구현 작성**

```js
// ---- 집계 (PRD 3.5) ----
function inRange(order, from, to) {
  return order.businessDate >= from && order.businessDate <= to;   // 'YYYY-MM-DD' 는 문자열 비교로 충분
}

function summarize(orders, from, to) {
  const picked = orders.filter((o) => inRange(o, from, to));
  const live = picked.filter((o) => o.status !== 'cancelled');

  const menuMap = new Map();
  const byHour = Array.from({ length: 24 }, (_, hour) => ({ hour, qty: 0 }));
  const days = new Set();
  let totalQty = 0, totalAmount = 0;

  for (const o of live) {
    days.add(o.businessDate);
    const hour = new Date(o.createdAt).getHours();
    for (const it of o.items || []) {
      const cur = menuMap.get(it.menuId) || { menuId: it.menuId, name: it.name, qty: 0, amount: 0 };
      cur.qty += it.qty;
      cur.amount += (it.price || 0) * it.qty;
      cur.name = it.name;                       // 이름이 바뀌었으면 최근 것을 쓴다
      menuMap.set(it.menuId, cur);
      byHour[hour].qty += it.qty;
      totalQty += it.qty;
      totalAmount += (it.price || 0) * it.qty;
    }
  }

  return {
    orderCount: live.length,
    cancelledCount: picked.length - live.length,
    dayCount: days.size,
    byMenu: [...menuMap.values()].sort((a, b) => b.qty - a.qty),
    byHour,
    totalQty,
    totalAmount,
  };
}

// ---- CSV ----
function csvCell(v) {
  const s = v === null || v === undefined ? '' : String(v);
  return /[",\n]/.test(s) ? `"${s.replace(/"/g, '""')}"` : s;
}

function toCsv(rows) {
  return rows.map((r) => r.map(csvCell).join(',')).join('\n');
}
```

- [ ] **Step 4: 통과 확인** — `PASS 45/45`

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 판매 집계와 CSV 변환 로직

취소된 주문은 집계에서 제외한다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 16: 기록 화면

**Files:**
- Modify: `index.html` (5번 구획)

**Interfaces:**
- Consumes: `summarize`, `toCsv`, `businessDateOf`, `shoppingList`
- Produces: 관리 화면의 `기록` 탭, `downloadCsv(filename, rows)`

- [ ] **Step 1: 기간 선택 + 요약 카드**

`오늘` / `이번 주` / `이번 달` / `기간 지정` 버튼. 기본 `오늘`.
요약: 주문 건수 · 총 잔 수 · 운영일 수 · 일평균 · 취소 건수. (금액은 `priceMode !== 'off'`일 때만)

- [ ] **Step 2: 메뉴별 판매량**

`byMenu`를 표 + 가로 막대로. 막대는 CSS만으로 그린다 (차트 라이브러리를 추가하지 않는다).

```html
<div class="bar-row">
  <span class="bar-name">아이스 아메리카노</span>
  <span class="bar" style="width: {qty/max*100}%"></span>
  <b class="bar-qty">12잔</b>
</div>
```

막대 길이는 보조 표현이고, **숫자를 반드시 함께 적는다** (색·길이에만 의존하지 않는다).

- [ ] **Step 3: 시간대별 분포**

`byHour`에서 판매가 있는 시간만 세로 막대로. 학교 일과 시간대(8~17시)를 기본 구간으로 한다.

- [ ] **Step 4: 재료 대사표**

재료별로 `현재 잔량` / `기간 내 입고` / `판매 차감` / `폐기`를 `stockLogs`에서 합산해 표로.

- [ ] **Step 5: 취소 내역 목록**

번호 · 시각 · 메뉴 요약.

- [ ] **Step 6: CSV 내보내기**

```js
function downloadCsv(filename, rows) {
  // 엑셀에서 한글이 깨지지 않도록 BOM을 붙인다
  const blob = new Blob(['﻿' + toCsv(rows)], { type: 'text/csv;charset=utf-8;' });
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = filename;
  a.click();
  URL.revokeObjectURL(a.href);
}
```

두 가지를 내보낸다.
- `달보드레_요약_2026-09-20.csv` — 메뉴별 판매량
- `달보드레_주문내역_2026-09-20.csv` — 주문 한 줄씩

- [ ] **Step 7: 화면 확인**

| 확인할 것 | 기대 |
|---|---|
| 주문 몇 건 + 취소 1건 후 `오늘` | 건수·잔 수가 맞고, 취소분이 빠져 있다 |
| 메뉴별 막대 | 많이 팔린 순, 숫자가 막대 옆에 있다 |
| 시간대별 | 주문한 시간에 막대가 선다 |
| CSV 내려받기 | 엑셀에서 열었을 때 **한글이 안 깨진다** |
| `이번 달` | 어제 주문까지 포함된다 |
| 금액 | `priceMode: 'off'`면 안 보인다 |

- [ ] **Step 8: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 기록 화면 (기간별 집계, 메뉴별·시간대별 분포, CSV 내보내기)

막대 그래프는 CSS만으로 그리고 숫자를 항상 함께 적는다.
CSV에 BOM을 붙여 엑셀에서 한글이 깨지지 않게 한다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

> **Phase 2 완료 지점.** PRD의 P1까지 끝났다. 재료가 저절로 줄고, 장볼 목록이 나오고, 학기 기록이 쌓인다.

---

# Phase 3 — P2: 여유가 될 때

---

### Task 17: 금액 모드 3단계

PRD 3.1.6. `price` 필드와 데이터 구조는 Task 6부터 이미 있으므로 여기서는 **표시와 계산기만** 붙인다.

**Files:**
- Modify: `index.html` (2번 · 5번 · 6번 구획)

**Interfaces:**
- Produces:
  - `changeOf(total, received) → { enough, change, shortage }` (순수 함수)
  - `quickCashOptions(total, units) → number[]` — 빠른 입력 버튼 금액

- [ ] **Step 1: 실패하는 테스트 추가**

```js
// ---- 계산 훈련 ----
test('거스름돈을 계산한다', () => {
  eq(changeOf(2500, 5000), { enough: true, change: 2500, shortage: 0 });
  eq(changeOf(2500, 2500), { enough: true, change: 0, shortage: 0 });
});

test('받은 돈이 모자라면 부족액을 알려준다', () => {
  eq(changeOf(2500, 1000), { enough: false, change: 0, shortage: 1500 });
});

test('빠른 입력에는 권종과 딱 맞는 금액이 들어간다', () => {
  const opts = quickCashOptions(2500, [1000, 5000, 10000]);
  if (!opts.includes(2500)) throw new Error('딱 맞는 금액이 없습니다');
  if (!opts.includes(5000)) throw new Error('권종이 없습니다');
  eq(opts, [...opts].sort((a, b) => a - b), '오름차순으로 정렬한다');
  eq(new Set(opts).size, opts.length, '중복이 없다');
});
```

- [ ] **Step 2: 실패 확인** — `FAIL 45/48`

- [ ] **Step 3: 구현**

```js
// ---- 계산 훈련 모드 (PRD 3.1.6) ----
// 실제 결제가 아니다. 쿠폰카드로 값을 받고, 여기서는 계산 연습만 한다.
function changeOf(total, received) {
  const enough = received >= total;
  return {
    enough,
    change: enough ? received - total : 0,
    shortage: enough ? 0 : total - received,
  };
}

function quickCashOptions(total, units) {
  const set = new Set([total, ...units.filter((u) => u >= total)]);
  return [...set].sort((a, b) => a - b);
}
```

- [ ] **Step 4: 통과 확인** — `PASS 48/48`

- [ ] **Step 5: 화면에 3단계 반영**

| `priceMode` | 주문 화면 |
|---|---|
| `'off'` | 가격·합계를 DOM에 넣지 않는다 |
| `'show'` | 타일에 가격, 장바구니에 소계, 하단에 합계 |
| `'practice'` | `'show'` + 받은 돈 숫자 키패드 + 빠른 입력 버튼 + **거스름돈 초대형 표시** |

`'practice'`에서 받은 돈이 모자라면 `주문 완료` 버튼을 `disabled`로 두고 부족 금액을 안내한다.
**받은 돈·거스름돈은 주문 문서에 저장하지 않는다** — 연습이지 결제가 아니다.

관리 → 설정에 라디오 3개를 두고, 각 모드 아래에 한 줄 설명을 적는다.

- [ ] **Step 6: 화면 확인**

| 확인할 것 | 기대 |
|---|---|
| `끔` | 금액이 아무 데도 없다 |
| `가격 표시` | 타일·장바구니·합계에 금액 |
| `계산 훈련` | 키패드가 나오고 거스름돈이 크게 표시된다 |
| 모자란 금액 입력 | `주문 완료`가 눌리지 않고 부족액이 보인다 |
| 주문 후 로컬 스토리지 | 주문 문서에 `received`·`change`가 **없다** |

- [ ] **Step 7: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 금액 모드 3단계 (끔 / 가격 표시 / 계산 훈련)

계산 훈련은 연습이므로 받은 돈과 거스름돈을 주문 문서에 남기지 않는다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 18: 폐기 기록

PRD 3.4. 조리 실패·시음분을 **재고에서만** 빼고 판매량에는 넣지 않는다.

**Files:**
- Modify: `index.html` (5번 구획)

- [ ] **Step 1: 재료 탭에 `폐기` 버튼 추가**

재료 행마다 `폐기` → 수량 + 사유 메모(선택) 모달 → `bump({ current: -n })` + `stockLogs`에 `type: 'waste'`.

- [ ] **Step 2: 제조 화면에 `만들다 실패` 단축 버튼**

메뉴를 고르면 그 레시피만큼 폐기 처리한다.

```js
async function wasteByMenu(menu, memo) {
  const delta = stockDelta([{ menuId: menu.id, qty: 1 }], { [menu.id]: menu });
  for (const [ingredientId, d] of Object.entries(delta)) {
    await state.backend.bump('ingredients', ingredientId, { current: d });
    await state.backend.add('stockLogs', {
      ingredientId, type: 'waste', qty: d, memo: memo || '', createdAt: Date.now(),
    });
  }
}
```

- [ ] **Step 3: 기록 화면 재료 대사표에 `폐기` 열이 채워지는지 확인**

- [ ] **Step 4: 화면 확인**

| 확인할 것 | 기대 |
|---|---|
| 우유 200ml 폐기 | 재고가 줄고 판매량은 그대로 |
| `만들다 실패` → 라떼 | 레시피만큼 빠진다 |
| 기록 → 재료 대사표 | `폐기` 열에 숫자가 들어간다 |

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 폐기 기록 (재고에서만 차감, 판매량 미반영)

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 19: 테마 색상 · 호출 문구 · 데이터 초기화

**Files:**
- Modify: `index.html` (5번 구획)

- [ ] **Step 1: 테마 색상**

관리 → 설정에 색상 프리셋 4~5개 (예: 달보드레 기본 / 민트 / 라벤더 / 숲 / 밤).
고르면 `--brand` 등 CSS 변수를 바꾸고 `settings/app.theme`에 저장한다.

- [ ] **Step 2: 호출 문구·음성 설정**

- 문구 입력칸 (`{번호}`가 실제 번호로 바뀐다는 안내 + 미리보기)
- 속도·볼륨 슬라이더
- 차임음 on/off
- **`호출 테스트`** 버튼 — `playChime()` + `speak(speechText(문구, 3))` (음성은 한자어 수사)

- [ ] **Step 3: 데이터 초기화**

2단계 확인:
1. `confirmModal` — "오늘까지의 모든 주문·기록이 지워집니다. 메뉴와 재료는 남습니다."
2. 카페 이름(`달보드레`)을 직접 입력해야 버튼이 활성화

지울 대상: `orders`, `stockLogs`, `counterResets`, `counters`.
**`menus`·`ingredients`·`settings`는 남긴다** — 리허설 후 초기화할 때 메뉴를 다시 등록하게 만들면 안 된다.

삭제 대상 컬렉션은 모두 구독 중이어야 지울 수 있다. `stockLogs`는 Task 13에서 이미 구독했으므로
여기서는 `counterResets`만 더한다.

```js
// 7번 구획 — 구독 목록에 counterResets를 더한다
for (const name of ['menus', 'ingredients', 'orders', 'stockLogs', 'counterResets']) {
  state.backend.subscribe(name, (docs) => { state[name] = docs; render(); });
}
```

```js
// 기록만 지운다. 메뉴·재료·설정은 남긴다 —
// 리허설 후 초기화할 때 메뉴를 다시 등록하게 만들면 안 된다.
async function clearRecords() {
  for (const name of ['orders', 'stockLogs', 'counterResets']) {
    for (const doc of [...(state[name] || [])]) {
      await state.backend.remove(name, doc.id);
    }
  }
  await state.backend.resetOrderNo(businessDateOf(), 0);
  toast('기록을 초기화했어요. 메뉴와 재료는 그대로입니다.');
}
```

- [ ] **Step 4: 화면 확인**

| 확인할 것 | 기대 |
|---|---|
| 테마 변경 | 색이 바뀌고 새로고침해도 유지된다 |
| `호출 테스트` | 차임음 + "3번 손님, 음료 나왔습니다." |
| 문구를 `{번호}번이요`로 변경 후 테스트 | "3번이요" |
| 초기화 | 이름을 정확히 입력해야만 버튼이 눌린다 |
| 초기화 후 | 주문·기록이 비고, **메뉴와 재료는 그대로**, 다음 주문이 1번 |

- [ ] **Step 5: 커밋**

```bash
git add index.html
git commit -m "$(cat <<'MSG'
feat: 테마 색상·호출 문구 설정·데이터 초기화

초기화는 기록만 지우고 메뉴·재료·설정은 남긴다.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

### Task 20: 시범 운영 점검

**Files:**
- Modify: `README.md`

- [ ] **Step 1: 전체 자가 테스트**

배포된 주소에 `?selftest=1` → `PASS 48/48`

- [ ] **Step 2: 기기 3대 실전 리허설**

PRD 9장의 확인 사항을 하나씩 짚는다.

| 확인할 것 | 기대 |
|---|---|
| 태블릿 2대 + 모니터에서 각각 역할 선택 | 새로고침해도 유지된다 |
| 실제 메뉴·사진·재료·레시피 등록 | 문제없이 저장된다 |
| 연속 주문 10건 | 번호가 1~10, 빠짐없이 제조 화면에 뜬다 |
| 호출 | 모니터에서 소리와 번호가 나온다. 교실 뒤에서 들리는지 볼륨 확인 |
| 와이파이 끊기 | 안내가 뜨고, 복구 시 자동 동기화 |
| 학생 조작 | 실제 학생이 한 번 돌려보고, 글자 크기·타일 크기가 맞는지 관찰 |
| 관리 PIN | 학생이 들어가지지 않는다 |
| 리허설 후 초기화 | 기록만 지워지고 메뉴·재료는 남는다 |

- [ ] **Step 3: 관찰한 것을 README에 반영**

실제로 헷갈렸던 지점, 기기별 주의사항(음성이 안 나오는 기기 등)을 "선생님께" 절에 덧붙인다.

- [ ] **Step 4: 커밋**

```bash
git add README.md
git commit -m "$(cat <<'MSG'
docs: 시범 운영에서 확인한 내용 반영

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

---

## 부록: PRD 요구사항 ↔ 태스크 대응표

| PRD | 태스크 |
|---|---|
| 2.3 영업일 자동 전환 | 2 |
| 3.1.1 사진+글자 타일 | 6(등록), 7(표시) |
| 3.1.2 장바구니 | 7 |
| 3.1.3 품절 처리 | 6(토글), 7(표시), 8(단축) |
| 3.1.4 주문 확정·번호 발번 | 7, 4(로컬), 11(Firestore) |
| 3.1.5 주문 취소·재료 복구 | 14 |
| 3.1.6 금액 3단계 | 17 (`price` 필드는 6부터) |
| 3.1.7 글자 크기 3단계 | 5 |
| 3.2 제조 화면 | 8 |
| 3.3 호출 화면 | 9 |
| 3.4 재료 재고 | 12(로직), 13(화면), 18(폐기) |
| 3.5 기록 화면 | 15(로직), 16(화면) |
| 3.6 관리 화면 | 6, 13, 16, 19 |
| 3.6.1 주문번호 초기화 | 2(판정), 10(화면) |
| 5 접근성 8원칙 | 전 태스크 (5·7·8·16에 명시적 확인 항목) |
| 6.1 단일 파일 | 1 |
| 6.2 Firestore·로컬 폴백 | 4, 11 |
| 6.3 사진 저장 | 3, 6 |
| 6.4 동시성 | 4, 11 |
| 6.5 UI/UX·푸터 | 1, 5 |
| 7 데이터 모델 | 4, 6, 7, 10, 12, 13 |

---

Made by 정우쌤
