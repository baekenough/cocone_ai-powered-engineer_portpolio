# 3. AI 산출물 전후 비교

> ← [목차로 돌아가기](../README.md)

## 배경: codex-exec 래퍼 — 20라운드 크로스 모델 검증

**저장소**: https://github.com/baekenough/baekenough-skills

oh-my-customcode의 `codex-exec` 스킬을 독립 레포로 분리하여 skills.sh 마켓플레이스에 배포했습니다. 배포 전 **20라운드의 크로스 모델 검증** (Codex xhigh 10라운드 + Claude Opus 10라운드)을 통해 7개 버그를 발견하고 전량 수정했습니다.

---

## 검증 결과 요약

| 메트릭 | 값 |
|--------|-----|
| 총 검증 라운드 | 20 (Codex xhigh 10 + Claude Opus 10) |
| 발견된 이슈 | 7개 |
| 수정 완료 | 7/7 (100%) |
| 크로스 플랫폼 지원 | macOS, Linux, Windows |
| 마켓플레이스 배포 | skills.sh 라이브 |

---

## 핵심 Before/After 비교

**비교 1 — SIGKILL 타이밍 버그**

프로세스 타임아웃 처리 로직에서 발견된 버그입니다. `child.killed`가 SIGTERM 전송 즉시 `true`가 되는 Node.js 동작을 간과했습니다.

Before:
```javascript
// Bug: child.killed becomes true immediately after SIGTERM
// so the SIGKILL branch never executes — process may become zombie
setTimeout(() => {
  if (!child.killed) {  // <- SIGTERM 후 항상 true이므로 이 블록 진입 불가
    child.kill('SIGKILL');
  }
}, KILL_GRACE_PERIOD_MS);
```

After:
```javascript
// Fix: always attempt SIGKILL, use try-catch for already-exited process
setTimeout(() => {
  try {
    child.kill('SIGKILL');
  } catch (e) {
    // Process already terminated — ignore
  }
}, KILL_GRACE_PERIOD_MS);
```

**교훈**: `child.killed`는 프로세스 실제 종료가 아닌 시그널 전송 여부를 나타냅니다. 타임아웃 강제 종료에는 조건 분기보다 `try-catch`가 더 안전한 패턴입니다.

---

**비교 2 — stdin 무시 누락**

non-interactive CLI 래퍼에서 stdin 처리 누락으로 자식 프로세스가 hang하는 문제입니다.

Before:
```javascript
const child = spawn(binary, args, {
  cwd: workingDir || process.cwd(),
  env: process.env,
  // stdio 미설정 -> stdin 기본값 'pipe' -> 자식 프로세스가 입력 대기로 hang
});
```

After:
```javascript
const child = spawn(binary, args, {
  cwd: workingDir || process.cwd(),
  env: process.env,
  stdio: ['ignore', 'pipe', 'pipe'],  // stdin closed -> hang 없음
});
```

**교훈**: non-interactive 래퍼에서 stdin을 `'ignore'`로 설정하지 않으면 자식 프로세스가 터미널 입력을 기다리며 무한 대기 상태가 됩니다. CLI 자동화에서 `stdio` 명시는 필수입니다.

---

**비교 3 — exit code 정규화**

시그널 기반 종료 코드가 소비자 입장에서 의미 불명확한 값으로 전달되는 문제입니다.

Before:
```javascript
// Non-standard exit codes passed through as-is
result.exit_code = execResult.exitCode;
// 137 (SIGKILL), 143 (SIGTERM), 126, 127 등 소비자가 처리 불가한 값 혼입
```

After:
```javascript
// Normalized to 3 standard codes
// 0: success, 1: execution error, 2: validation error
result.exit_code = execResult.exitCode === 0
  ? 0
  : (execResult.exitCode === 2 ? 2 : 1);
```

**교훈**: 시그널 기반 종료 코드(137=SIGKILL, 143=SIGTERM)는 OS 레벨의 정보로, API 소비자 입장에서 의미가 불명확합니다. 3-tier 정규화(0/1/2)로 일관된 에러 처리 인터페이스를 제공합니다.

---

## 크로스 모델 검증의 의의

단일 모델 검증으로는 모델 특유의 사각지대가 존재합니다. **Codex xhigh**는 코드 실행 관점의 엣지 케이스에 강하고, **Claude Opus**는 설계 의도와 철학적 일관성 검토에 강합니다. 두 모델을 교차 검증함으로써 더 넓은 결함 탐지 범위를 확보했습니다.

| 모델 | 강점 | 발견 이슈 유형 |
|------|------|---------------|
| Codex xhigh | 코드 실행 엣지 케이스 | SIGKILL 타이밍, stdin hang, exit code 혼입 |
| Claude Opus | 설계 일관성, cross-platform | Windows 경로 처리, agent-coupled 언어 제거 |

7개 이슈 전량 수정 후 skills.sh 마켓플레이스에 라이브 배포하였으며, `npx skills add baekenough/codex-exec` 명령으로 즉시 설치 가능합니다.
