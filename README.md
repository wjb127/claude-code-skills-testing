# Claude Code Skills — Testing & Tracking Verification

웹 개발 외주/사이드 프로젝트에서 반복적으로 필요한 **테스트 자동화**와 **트래킹 픽셀 검증** 워크플로우를 Claude Code Skill로 패키징한 모음.

AI(특히 바이브 코딩)가 짠 코드의 약점 — 가짜 그린 테스트, 픽셀 누락, 듀얼 GTM/gtag 중복, 동의 신호 누락 — 을 잡아내기 위한 실전 체크리스트와 자동 검증 루프가 들어 있다.

## 포함된 스킬

### 1. `test-web-code` — 웹 코드 테스트 생성기 (Unit + Integration + E2E)
- 테스트 피라미드 비율(70/20/10) 자동 적용
- Vitest + Playwright (라이브러리 + Test Runner) 양쪽 지원
- Happy / Sad / Bad / Ugly 4가지 변종 템플릿
- `data-testid` 우선, `waitForTimeout` 금지 등 8가지 안티패턴 차단
- **프로덕션 URL 기반 E2E 실행** (`BASE_URL` env)
- 실패 시 trace.zip + 스크린샷 자동 생성
- 단순 생성에서 끝나지 않고 실제 테스트까지 돌리고 결과를 정직하게 보고

→ [`skills/test-web-code/SKILL.md`](skills/test-web-code/SKILL.md)

### 2. `verify-pixels` — 광고 픽셀 / GA4 검증기
- GA4, Google Ads, GTM, Meta Pixel, TikTok, Naver, Kakao, Bing, Hotjar, Clarity 10개 플랫폼 자동 감지
- Playwright `page.on('request')`로 실제 네트워크 호출 캡처 (단순 스크립트 존재 여부 X)
- Consent Mode V2 4신호 검증 (`ad_storage`, `analytics_storage`, `ad_user_data`, `ad_personalization`)
- CAPI 중복 제거(`event_id`) 체크
- **스크린샷 증거 + HTML 리포트** 자동 생성 → 클라이언트 반박 차단용
- **Auto-Fix Loop**: 수정 → 빌드 → 배포 → 재검증 반복 (최대 5회, 30분 타임아웃, STUCK 감지)
- 9가지 자동 수정 가능 / 8가지 수동 개입 필요 카테고리 구분

→ [`skills/verify-pixels/SKILL.md`](skills/verify-pixels/SKILL.md)

## 설치

```bash
# ~/.claude/skills/ 아래에 복사
cp -r skills/test-web-code ~/.claude/skills/
cp -r skills/verify-pixels ~/.claude/skills/

# Claude Code 재시작 후
# /test-web-code <프로젝트설명>
# /verify-pixels --loop
```

또는 심볼릭 링크:
```bash
ln -s "$(pwd)/skills/test-web-code" ~/.claude/skills/test-web-code
ln -s "$(pwd)/skills/verify-pixels" ~/.claude/skills/verify-pixels
```

## 배경

이 스킬들이 잡으려는 대표적인 문제들:

- AI가 생성한 E2E 테스트가 mutation 50%를 못 잡고 그린 — "테스트 통과"가 안전 보장이 아님
- 클라이언트가 "픽셀 아직 설치 안 된 것 같아요" 라고 했을 때 즉시 증거 제출이 안 됨
- gtag + GTM 듀얼 설치로 PageView 2배 카운트
- Consent Mode V2 미구현으로 Google Ads 리마케팅 누락
- 모달 닫기 같은 UX 깨짐을 E2E가 못 잡고 그린

각 스킬 SKILL.md 안에 위 케이스들의 구체적 진단 + 자동 수정 패턴이 들어 있다.

## 라이선스

MIT
