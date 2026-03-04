# 5. AI 한계 극복 경험

> ← [목차로 돌아가기](../README.md)

AI를 실제 프로덕션 시스템에 통합하면서 마주친 **주요 AI 한계**와 이를 체계적으로 극복한 경험입니다.
문제별로 탐지 메커니즘 → 수정 프로세스 → 예방 조치의 3단계 대응 체계를 구축했습니다.

## 5.1 구조적 드리프트 (Structural Drift)

**문제 현상**

AI가 새 에이전트나 스킬을 추가할 때, 연관된 참조(라우팅 패턴, 문서 카운트, CLAUDE.md 테이블)를 누락하는 패턴이 반복됐습니다.
예를 들어, 새 에이전트 파일을 `.claude/agents/`에 생성하면서 라우팅 스킬의 `agent-triggers.yaml`과 `CLAUDE.md`의 에이전트 목록을 갱신하지 않는 경우입니다.

**대처: mgr-sauron 8라운드 검증 시스템**

```
Phase 1 — Manager Verification (5 rounds)
  Round 1-2: mgr-supplier:audit + mgr-updater:docs 동기화 검증 + 이슈 수정
  Round 3-4: 재검증 (수량, frontmatter, 스킬 참조, memory scope) + 잔여 수정
  Round 5:   최종 확인 — 모든 카운트 일치, 라우팅 패턴 갱신 완료

Phase 2 — Deep Review (3 rounds)
  Round 1: 워크플로우 정합성 — 라우팅 스킬이 모든 에이전트를 커버하는가
  Round 2: 참조 무결성 — 고아 참조, 순환 참조, 유효하지 않은 스킬 ref 없는가
  Round 3: 설계 철학 — R006 관심사 분리, R009 병렬, R010 위임, R007/R008 식별 준수
```

**결과**: R017 규칙으로 공식화, 모든 `git push` 전 sauron 검증 의무화

---

## 5.2 코드 결함 (Code Defects)

**문제 현상**

AI 생성 코드는 정상 경로에서는 잘 동작하지만, 엣지 케이스에서 예상치 못한 버그를 포함하는 경우가 있었습니다.
대표 사례로는 Codex CLI 래퍼 스킬(`codex-exec`) 개발 중 발생한 3가지 버그입니다:

| 버그 | 증상 | 단일 모델 탐지 여부 |
|------|------|-------------------|
| SIGKILL 타이밍 오류 | `child.killed`가 SIGTERM 직후 `true` → SIGKILL 분기에서 오탐 | 미탐지 |
| stdin 행 (hang) | stdin이 닫히지 않아 프로세스가 입력 대기 상태로 진입 | 미탐지 |
| exit code 비표준 | 타임아웃 시 `null` exit code 반환 → 호출자 혼란 | 탐지 |

**대처: 크로스 모델 검증 (multi-model-verification)**

단일 모델 리뷰의 맹점을 극복하기 위해 Codex와 Claude Opus를 병렬로 동원했습니다.

```
Step 1: Codex (xhigh reasoning effort)로 코드 구현
Step 2: Claude Opus로 독립적 코드 리뷰 (다른 관점)
Step 3: 두 모델의 발견 사항 교차 검증
Step 4: 불일치 항목 집중 수정
Step 5: 수정 후 재검증 (총 20라운드)
```

**결과**: 단일 모델 대비 발견 이슈 2배 이상 (3-4개 → 7개), `baekenough-skills` 마켓플레이스 배포 품질 달성

---

## 5.3 보안 취약점 (Security Vulnerabilities)

**문제 현상**

AI는 기능 구현에 집중하는 경향이 있어 의존성 버전 관리와 민감 데이터 노출 같은 보안 이슈를 간과하는 경우가 있습니다.
실제로 `vite` 취약점이 `npm audit`에서 발견된 사례가 있었으며, 디버깅 코드로 작성된 `console.log`가 민감 컨텍스트를 출력할 위험도 있었습니다.

**대처: 다층 보안 체계**

| 계층 | 도구 | 역할 |
|------|------|------|
| CI 계층 | `security-audit.yml` | 주기적 `npm audit`, 취약 의존성 자동 PR |
| 편집 계층 | `console.log detector` 훅 | 파일 저장 시 `console.log` 잔류 즉시 경고 |
| 규칙 계층 | R001 Safety + R002 Permission | AI가 `.env`, `.git/config` 등 민감 파일 접근 금지 |
| 패키지 계층 | `overrides` + `npm update` | 간접 의존성 취약점을 `package.json` overrides로 고정 |

**결과**: `npm audit 0 vulnerabilities` 상시 유지, API 키 노출 0건

---

## 5.4 AI 오류 유형별 탐지/수정 프로세스 요약

| AI 오류 유형 | 탐지 메커니즘 | 수정 프로세스 | 예방 조치 |
|-------------|-------------|-------------|----------|
| 구조적 드리프트 | mgr-sauron 8라운드 검증 | 자동 수정 + 재검증 | R017 push 전 필수 검증 |
| 코드 결함 | 크로스 모델 검증 (Codex + Claude) | Before/After 비교 수정 | multi-model-verification 스킬 |
| 보안 취약점 | security-audit CI + hooks | `npm audit fix` + overrides | R001/R002 강제 규칙 |
| 규칙 위반 | git-delegation-guard hook | 실시간 경고 (advisory, 흐름 비차단) | R010 오케스트레이터 규칙 |
| 문서 불일치 | docs-sync CI + mgr-updater | 자동 동기화 | docs-validator 검증 |
