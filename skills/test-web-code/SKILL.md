---
name: test-web-code
description: |
  Generate and run Unit + Integration + E2E tests for a web project. Every scenario ships
  Happy / Sad / Bad / Ugly variants, so happy-path-only suites are impossible.
  Production-first E2E with no automation bypasses — no fake-green tests.
  웹 프로젝트 통합 테스트 생성·실행. 해피패스 금지, 우회 없는 프로덕션 E2E.
---

# test-web-code — 웹 프로젝트 통합 테스트 생성/실행기

현재 프로젝트의 스택을 분석해서 **Unit + Integration + E2E** 테스트를 한 번에 셋업하고 실행한다. **E2E에 집중**하되 피라미드 비율(70/20/10)을 의식한다.

2026년 업계 데이터 기반으로 다음을 전제로 한다:
- AI 생성 테스트는 mutation 테스팅에서 약 **50%가 실제 버그 탐지 실패** → 의식적으로 negative/edge 케이스 생성
- happy path만 작성하면 거짓 안정감 (false green) 발생
- 프로덕션 환경 차이(DNS, SSL, 시간대, 캐시)는 로컬에선 안 잡힘
- 셀렉터 취약성, hardcoded wait, POM 부재가 첫 실행 실패의 주범

---

## 핵심 원칙 (반드시 지킨다)

### 1. Happy path 금지 — Edge/Error/Negative 명시
모든 시나리오는 다음 4가지 변형을 함께 생성:
- **Happy**: 정상 흐름
- **Sad**: 유효성 위반, 빈 값, 초과 길이, 잘못된 타입
- **Bad**: 권한 거부, 401/403, 인증 만료
- **Ugly**: XSS(`<script>alert(1)</script>`), SQL injection(`'; DROP TABLE`), 이모지/한글 인코딩, 동시 편집 conflict

### 2. 셀렉터 우선순위
1. `data-testid` (가장 안정적) — 없으면 추가 권장
2. `role` + accessible name (`getByRole`)
3. `getByText` (정확 매치, regex 지양)
4. CSS 셀렉터 (최후 수단)
- **금지**: nth-child, 깊은 descendant selector, 클래스명 의존

### 3. Wait 전략
- ✅ `await page.waitForLoadState('networkidle')`
- ✅ `await expect(locator).toBeVisible({ timeout: 10000 })` (auto-waiting)
- ❌ `await page.waitForTimeout(3000)` (절대 금지, hardcoded sleep)
- ❌ `setTimeout` / sleep 패턴

### 4. 격리 (Isolation)
- 각 테스트가 자기 데이터만 생성/정리
- DB 시드 데이터에 의존 X — 테스트가 자체 생성
- ID는 timestamp + random suffix (`e2e-${Date.now()}-${Math.random().toString(36).slice(2,7)}`)
- afterEach에서 본인이 만든 것 cleanup

### 5. 프로덕션 우선
- `BASE_URL` 환경변수 지원 (기본 localhost, 프로덕션 검증 시 prod URL 주입)
- 배포 직후 Smoke 자동 실행
- 단, 프로덕션 DB 오염 방지: 테스트 데이터는 prefix(`e2e-`) + 마지막에 본인 삭제

### 6. Retry & Trace
- `@playwright/test` 러너 사용 (스크립트 직접 구현 지양)
- `retries: 2` (CI), `0` (로컬)
- `trace: 'on-first-retry'` — 실패 디버깅용
- `screenshot: 'only-on-failure'`, `video: 'retain-on-failure'`

### 7. Auth 처리
- 한 번 로그인 → `storageState` 저장 → 모든 테스트에서 재사용
- 매 테스트마다 prompt/로그인 반복 금지
- Basic Auth는 `extraHTTPHeaders` 또는 `httpCredentials` 사용

---

## 실행 흐름

### Step 1 — 프로젝트 분석

다음을 순서대로 수집한다 (Read/Bash 도구로):

```bash
# 스택 감지
cat package.json    # framework, deps, scripts
ls src/             # 디렉토리 구조
git remote -v       # 프로젝트 식별
```

감지 항목:
- **Framework**: Next.js / Astro / Remix / Vite+React / Vue / Nuxt / Express / NestJS
- **언어**: TypeScript / JavaScript
- **패키지 매니저**: pnpm / npm / yarn / bun
- **기존 테스트**: `tests/`, `__tests__/`, `*.test.ts`, `*.spec.ts` 유무
- **DB**: Supabase / Neon / Prisma / Drizzle / TypeORM
- **인증**: Basic Auth / NextAuth / Clerk / Supabase Auth / 자체 JWT
- **배포**: Vercel / Cloudflare / Netlify (URL 추출)

기존에 `playwright.config.ts`나 `vitest.config.ts`가 있으면 **재활용**, 없으면 새로 생성.

### Step 2 — Critical Flows 식별

사용자에게 질문 (1회만, 답 받으면 진행):

```
프로젝트 분석 결과:
- Framework: {감지된 스택}
- 기존 테스트: {유무}
- 배포 URL: {감지된 URL}

테스트할 핵심 사용자 흐름을 알려줘 (예: "회원가입 → 로그인 → 게시글 작성 → 결제").
모르면 "auto" 라고 하면 라우트 파일 보고 자동 추론할게.
```

`auto` 응답 시: `src/pages/`, `app/`, `routes/` 탐색해서 public/admin/API 분류 후 상위 5개 flow 자동 선정.

### Step 3 — 테스트 도구 셋업

기본 스택 (사용자가 다른 거 안 정하면 이걸로):

| 레이어 | 도구 | 이유 |
|---|---|---|
| **Unit** | Vitest | Vite 호환, 빠름, ESM 네이티브 |
| **Integration** | Vitest + supertest (Node API) | 같은 러너 사용 |
| **E2E** | @playwright/test | retry/trace/병렬 무료, 1.56+ 자체 AI agent |
| **Mutation (옵션)** | Stryker | JS/TS 표준 |
| **Visual (옵션)** | @playwright/test + toHaveScreenshot | 추가 의존성 없음 |
| **A11y (옵션)** | axe-playwright | 통합 간단 |

설치 명령 (해당 PM 사용):
```bash
pnpm add -D @playwright/test vitest @vitest/ui supertest @types/supertest
pnpm exec playwright install chromium
# 옵션
pnpm add -D @stryker-mutator/core @stryker-mutator/vitest-runner
pnpm add -D axe-playwright
```

### Step 4 — 파일 구조 생성

```
tests/
├── unit/
│   └── *.test.ts          # 순수 함수, 유틸, 로직
├── integration/
│   └── *.test.ts          # API route, DB 어댑터
├── e2e/
│   ├── fixtures/
│   │   ├── auth.ts        # storageState 생성
│   │   └── test-data.ts   # factory 함수
│   ├── pages/             # POM (Page Object Model)
│   │   └── *.page.ts
│   ├── flows/
│   │   ├── *.spec.ts      # 핵심 사용자 시나리오
│   │   └── *.smoke.ts     # 배포 직후 1분 안에 끝나는 검증
│   └── helpers/
│       └── cleanup.ts     # 테스트 데이터 정리
├── playwright.config.ts
└── vitest.config.ts
```

### Step 5 — 코드 생성 패턴

#### Unit 테스트 템플릿
```ts
import { describe, it, expect } from 'vitest';
import { formatDate } from '@/lib/utils';

describe('formatDate', () => {
  it('formats ISO string to YYYY-MM-DD', () => {
    expect(formatDate('2026-05-22T10:00:00Z')).toBe('2026-05-22');
  });

  // Sad path — 의식적으로 작성
  it('throws on invalid input', () => {
    expect(() => formatDate('not-a-date')).toThrow();
  });

  it('handles null/undefined', () => {
    expect(formatDate(null)).toBe('');
    expect(formatDate(undefined)).toBe('');
  });
});
```

#### Integration 테스트 템플릿
```ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import request from 'supertest';
import app from '@/app';

describe('POST /api/posts', () => {
  let createdId: string;

  afterEach(async () => {
    if (createdId) {
      await request(app).delete(`/api/posts/${createdId}`).set('Authorization', authHeader());
      createdId = '';
    }
  });

  it('creates a post with valid input', async () => {
    const res = await request(app)
      .post('/api/posts')
      .set('Authorization', authHeader())
      .send({ title: `e2e-${Date.now()}`, content: 'test' });
    expect(res.status).toBe(201);
    createdId = res.body.id;
  });

  // Negative cases
  it('returns 401 without auth', async () => {
    const res = await request(app).post('/api/posts').send({ title: 'x' });
    expect(res.status).toBe(401);
  });

  it('returns 400 on empty title', async () => {
    const res = await request(app).post('/api/posts').set('Authorization', authHeader()).send({ title: '' });
    expect(res.status).toBe(400);
  });

  it('sanitizes XSS in title', async () => {
    const res = await request(app)
      .post('/api/posts')
      .set('Authorization', authHeader())
      .send({ title: '<script>alert(1)</script>', content: 'x' });
    if (res.status === 201) {
      createdId = res.body.id;
      expect(res.body.title).not.toContain('<script>');
    } else {
      expect(res.status).toBe(400);
    }
  });
});
```

#### E2E 테스트 템플릿 (@playwright/test)
```ts
import { test, expect } from '@playwright/test';

const baseURL = process.env.BASE_URL || 'http://localhost:3000';

test.describe('블로그 글 작성 흐름', () => {
  test.use({
    httpCredentials: process.env.ADMIN_USERNAME && process.env.ADMIN_PASSWORD
      ? { username: process.env.ADMIN_USERNAME, password: process.env.ADMIN_PASSWORD }
      : undefined,
  });

  let createdSlug: string;

  test.afterEach(async ({ request }) => {
    if (createdSlug) {
      // cleanup
      await request.delete(`/api/posts/${createdSlug}`).catch(() => {});
      createdSlug = '';
    }
  });

  test('Happy: 정상 작성 → 발행 → 공개 페이지 노출', async ({ page }) => {
    createdSlug = `e2e-${Date.now()}`;
    await page.goto('/admin/blog');
    await page.getByTestId('new-post-button').click();
    await page.getByTestId('post-title').fill('테스트 제목');
    await page.getByTestId('post-slug').fill(createdSlug);
    await page.getByTestId('post-content').fill('본문');
    await page.getByTestId('publish-button').click();
    await expect(page.getByText('발행 완료')).toBeVisible();

    // 공개 페이지 노출 확인
    await page.goto(`/blog/${createdSlug}`);
    await expect(page.getByRole('heading', { name: '테스트 제목' })).toBeVisible();
  });

  test('Sad: 빈 제목 제출 시 에러 표시', async ({ page }) => {
    await page.goto('/admin/blog');
    await page.getByTestId('new-post-button').click();
    await page.getByTestId('publish-button').click();
    await expect(page.getByText(/제목.*필수/)).toBeVisible();
  });

  test('Sad: 중복 slug 시 거부', async ({ page, request }) => {
    const slug = `e2e-dup-${Date.now()}`;
    createdSlug = slug;
    // 첫 번째 생성
    await request.post('/api/posts', { data: { slug, title: 'first' } });
    // UI에서 같은 slug 시도
    await page.goto('/admin/blog');
    await page.getByTestId('new-post-button').click();
    await page.getByTestId('post-slug').fill(slug);
    await page.getByTestId('publish-button').click();
    await expect(page.getByText(/중복|이미 존재/)).toBeVisible();
  });

  test('Bad: 인증 없이 admin 접근 시 401', async ({ browser }) => {
    const ctx = await browser.newContext({ httpCredentials: undefined });
    const page = await ctx.newPage();
    const res = await page.goto('/admin/blog');
    expect(res?.status()).toBe(401);
    await ctx.close();
  });

  test('Ugly: XSS payload 입력 시 sanitize 또는 거부', async ({ page }) => {
    createdSlug = `e2e-xss-${Date.now()}`;
    await page.goto('/admin/blog');
    await page.getByTestId('new-post-button').click();
    await page.getByTestId('post-title').fill('<script>window.__pwned=true</script>');
    await page.getByTestId('post-slug').fill(createdSlug);
    await page.getByTestId('publish-button').click();

    await page.goto(`/blog/${createdSlug}`);
    const pwned = await page.evaluate(() => (window as any).__pwned);
    expect(pwned).toBeUndefined();
  });
});
```

#### playwright.config.ts 템플릿
```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 2 : undefined,
  reporter: [['html', { open: 'never' }], ['list']],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'mobile-chrome', use: { ...devices['Pixel 7'] } },
  ],
  webServer: process.env.BASE_URL ? undefined : {
    command: 'pnpm dev',
    url: 'http://localhost:3000',
    reuseExistingServer: true,
    timeout: 120_000,
  },
});
```

### Step 6 — 실제 실행 (반드시 수행)

생성만 하고 끝내지 말 것. 다음을 순서대로 실행:

```bash
# 1. Unit + Integration
pnpm exec vitest run

# 2. E2E (로컬)
pnpm exec playwright test

# 3. E2E (프로덕션) — 배포 URL이 있으면
BASE_URL=https://<prod-url> pnpm exec playwright test e2e/flows/*.smoke.ts
```

실패 시:
- trace 파일 열어서 (`pnpm exec playwright show-trace trace.zip`) 원인 파악
- 셀렉터 문제면 data-testid 추가 또는 더 안정적인 locator로 수정
- 타이밍 문제면 explicit wait 추가 (hardcoded sleep 금지)
- 환경 변수 문제면 `.env.local` 확인

**3번 재시도 후에도 실패하면 사용자에게 보고 — 무한 재시도 금지**.

### Step 7 — 리포트 생성

실행 완료 후 다음을 정리해서 사용자에게 보고:

```
✅ Unit:  {pass}/{total}  ({duration})
✅ Integ: {pass}/{total}  ({duration})
✅ E2E:   {pass}/{total}  ({duration})

📊 Playwright HTML 리포트: pnpm exec playwright show-report
🎬 실패 trace 보기: pnpm exec playwright show-trace <path>

⚠️ 다음 시나리오는 인프라/환경 한계로 자동화 불가 — 수동 검증 필요:
- {예: 결제 모듈 (Toss 샌드박스 키 필요)}
- {예: 이메일 발송 검증 (메일박스 접근 필요)}

🧪 다음 단계 권장:
- Mutation 검증: pnpm add -D @stryker-mutator/core && stryker run
- Visual diff 베이스라인: pnpm exec playwright test --update-snapshots
- A11y 통합: axe-playwright 추가
```

---

## 정직성 원칙

다음은 명시적으로 **사용자에게 보고**한다:

1. **"통과"의 의미를 정확히 표현**: "X/X 통과"라고만 쓰지 말고 "happy path X/X 통과, edge case Y/Z 커버, mutation 미검증" 형태로.
2. **누락된 영역 명시**: visual, a11y, security, load 같은 영역은 별도 도구가 필요하다는 점을 보고.
3. **Flaky 의심 케이스**: 같은 테스트가 재실행에서 결과 바뀌면 즉시 리포트.
4. **AI 자체 한계 인지**: 비즈니스 룰 추측은 코드 기반 hallucination 가능성 있음. 핵심 비즈니스 가정은 사용자 확인.

---

## 금지 사항

- ❌ `page.waitForTimeout(N)` — 무조건 condition-based wait로
- ❌ 셀렉터로 nth-child, 깊은 CSS descendant
- ❌ 매 테스트마다 로그인 반복 (storageState 사용)
- ❌ 시드 DB 데이터에 의존 (격리 깨짐)
- ❌ Playwright 라이브러리 직접 사용 (`import { chromium } from 'playwright'`) — `@playwright/test` 러너 강제
- ❌ Happy path만 제출 — 항상 Sad/Bad/Ugly 동반
- ❌ 테스트 통과율만 보고하고 mutation/visual/a11y 갭 숨김
- ❌ 3번 재시도 후 실패한 테스트를 임의로 skip 처리 — 사용자 확인 필수

---

## 사용자 요청 변형 처리

| 사용자 입력 | 동작 |
|---|---|
| `/test-web-code` (인자 없음) | Step 1부터 풀 플로우 |
| `/test-web-code e2e only` | Unit/Integration 건너뛰고 E2E만 |
| `/test-web-code smoke` | smoke 테스트만 생성/실행 (배포 직후용) |
| `/test-web-code <flow>` | 특정 flow만 (예: "로그인 흐름") |
| `/test-web-code --prod` | BASE_URL을 프로덕션으로 자동 설정 |
| `/test-web-code --mutation` | Stryker mutation 테스트 추가 실행 |

---

## 출력 톤

- 한국어, 반말, 간결
- 진행 중간 업데이트 1줄씩
- 코드 블록 외 이모지 금지 (사용자 글로벌 규칙 준수)
- 작업 종료 후 ntfy 알림 (사용자 글로벌 규칙)
- implementation-notes.html 생성 (3개 이상 파일 변경 시, 글로벌 규칙)
