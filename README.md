# AI Powered Engineer Portfolio

> 코코네엔지니어링 AI Powered Engineer 포지션 지원 포트폴리오

| 항목 | 내용 |
|------|------|
| 지원 포지션 | AI Powered Engineer |
| GitHub | [@baekenough](https://github.com/baekenough) |
| 주요 프로젝트 | [oh-my-customcode](https://github.com/baekenough/oh-my-customcode) |
| 기술 스택 | TypeScript, Python, Rust, Claude API, GitHub Actions |

---

## 목차

| # | 항목 | 코코네 가이드 항목 |
|---|------|-------------------|
| 1 | [AI 활용 문제 해결 사례](#1-ai-활용-문제-해결-사례) | 항목 1 |
| 2 | [프롬프트 템플릿](#2-프롬프트-템플릿) | 항목 2 |
| 3 | [AI 산출물 전후 비교](#3-ai-산출물-전후-비교) | 항목 3 |
| 4 | [개발 워크플로 통합](#4-개발-워크플로-통합) | 항목 4 |
| 5 | [AI 한계 극복 경험](#5-ai-한계-극복-경험) | 항목 5 |
| 6 | [LLM 기반 기능 개발](#6-llm-기반-기능-개발) | 항목 7 |

---

## 1. AI 활용 문제 해결 사례

### 배경: oh-my-customcode 에이전트 프레임워크

**GitHub**: https://github.com/baekenough/oh-my-customcode

Claude Code를 커스터마이징하는 npm 패키지로, 41개 에이전트, 55개 스킬, 19개 규칙, 22개 가이드 — 총 **136개 컴포넌트**로 구성된 AI 에이전트 프레임워크입니다.

오케스트레이터 패턴을 채택하여 메인 대화가 라우팅을 담당하고 서브에이전트가 실행을 담당합니다. 4개의 라우팅 스킬(`secretary-routing`, `dev-lead-routing`, `de-lead-routing`, `qa-lead-routing`)이 요청을 적절한 전문가 에이전트로 위임합니다.

```
Main Conversation (orchestrator)
  ├─ secretary-routing  → mgr-creator, mgr-updater, mgr-gitnerd, ...
  ├─ dev-lead-routing   → lang-golang-expert, be-fastapi-expert, fe-vercel-agent, ...
  ├─ de-lead-routing    → de-airflow-expert, de-kafka-expert, de-spark-expert, ...
  └─ qa-lead-routing    → qa-planner, qa-writer, qa-engineer
```

---

### 문제: 136개 컴포넌트 간 구조적 드리프트

프레임워크가 성장하면서 **구조적 드리프트(structural drift)** 문제가 부각되었습니다.

에이전트를 추가하거나 삭제할 때 다음 요소들이 연쇄적으로 영향을 받습니다.

| 영향 범위 | 내용 |
|-----------|------|
| 라우팅 패턴 | `secretary-routing`, `dev-lead-routing` 등 4개 스킬의 에이전트 목록 |
| 스킬 참조 | 에이전트 frontmatter의 `skills:` 필드가 실제 디렉토리와 일치하는지 |
| 메모리 스코프 | `memory:` 필드가 `project`, `user`, `local` 세 값 중 하나인지 |
| CLAUDE.md 요약 테이블 | 에이전트 수량 및 목록의 최신화 여부 |

수동으로 이를 검증하려면 **30분 이상**이 소요되며, 누락 가능성이 상존했습니다. 컴포넌트 수가 늘어날수록 검증 비용은 선형적으로 증가하는 구조였습니다.

---

### 해결: mgr-sauron 자동 검증 시스템 (R017)

규칙 `MUST-sync-verification.md`(R017)로 **8라운드 자동 검증** 체계를 구축했습니다.

**Phase 1 — Manager Verification (5 rounds)**

| 라운드 | 검증 항목 |
|--------|-----------|
| 1-2 | `mgr-supplier:audit` (의존성), `mgr-updater:docs` (문서 동기화) 실행 및 이슈 수정 |
| 3-4 | audit + docs 재검증, 잔여 이슈 수정 |
| 5 | 최종 확인: 컴포넌트 수 일치, frontmatter 유효성, 스킬 참조 존재 여부, 메모리 스코프 유효성, 라우팅 패턴 최신화 |

`mgr-claude-code-bible:verify`도 병행 실행하여 Claude Code 공식 스펙 준수 여부를 확인합니다.

**Phase 2 — Deep Review (3 rounds)**

| 라운드 | 검토 초점 |
|--------|-----------|
| 1 | Workflow alignment: 라우팅 스킬의 에이전트 매핑 완전성 |
| 2 | References: 고아 참조, 순환 참조, 유효하지 않은 스킬/메모리 참조 |
| 3 | Philosophy: R006(관심사 분리), R009(병렬 실행), R010(오케스트레이터 위임), R007/R008(에이전트 식별) |

**Phase 3 이후**: 발견된 이슈 전체 수정 → `mgr-gitnerd`를 통한 커밋 → `mgr-sauron` 통과 후에만 push 허용

---

### Quick Verification Commands

8라운드 검증의 핵심이 되는 자동화 쉘 스크립트입니다.

```bash
# Agent count check
ls .claude/agents/*.md | wc -l

# Skill count check
find .claude/skills -name "SKILL.md" | wc -l

# Frontmatter validation — 헤더 누락 파일 탐지
for f in .claude/agents/*.md; do
  head -1 "$f" | grep -q "^---" || echo "MISSING FRONTMATTER: $f"
done

# Invalid skill reference check — 존재하지 않는 스킬 참조 탐지
for f in .claude/agents/*.md; do
  grep "^skills:" -A 10 "$f" | grep "  - " | sed 's/.*- //' | while read skill; do
    [ -f ".claude/skills/$skill/SKILL.md" ] || echo "INVALID SKILL REF in $f: $skill"
  done
done

# Memory scope validation — 유효하지 않은 스코프 탐지
for f in .claude/agents/*.md; do
  mem=$(grep "^memory:" "$f" | awk '{print $2}')
  if [ -n "$mem" ] && [ "$mem" != "project" ] && [ "$mem" != "user" ] && [ "$mem" != "local" ]; then
    echo "INVALID MEMORY SCOPE in $f: $mem"
  fi
done
```

---

### 정량 지표

| 메트릭 | 이전 | 이후 |
|--------|------|------|
| 수동 검증 시간 | 30분 이상 | 0분 (완전 자동화) |
| 검증 라운드 | 없음 | 8라운드 (5 + 3) |
| GitHub Actions 워크플로우 | 0 | 12개 통합 |
| 주간 컴플라이언스 체크 | 수동 | 자동 |
| 드리프트 탐지 지연 | 며칠~수주 | 즉시 (커밋 전) |

---

### AI 기여도 분석

| 작업 영역 | 인간 기여 | AI 기여 |
|-----------|-----------|---------|
| 시스템 설계 (아키텍처, 규칙 체계) | 100% | 0% |
| 에이전트 파일 작성 (41개) | 30% | 70% (Claude 지원) |
| 검증 로직 구현 | 50% | 50% (Claude 지원) |

핵심 인사이트: AI가 반복적인 파일 작성을 지원하는 동안, 인간은 시스템의 철학과 규칙 설계에 집중할 수 있었습니다. **"전문가가 없으면 만들고, 지식을 연결하고, 사용한다"**는 프레임워크 철학 자체가 AI와의 협업으로 완성되었습니다.

---

## 2. 프롬프트 템플릿

### 배경: GitHub 이슈 자동 분석 파이프라인

**파일**: `.github/scripts/analyze-issue.ts`

oh-my-customcode 프로젝트의 GitHub Actions 워크플로우에서 새 이슈가 등록될 때 Claude API를 호출해 자동으로 분석하고 레이블, 우선순위, 기술적 접근 방향을 코멘트로 게시하는 시스템입니다.

---

### Before: 비구조적 프롬프트의 문제

```
"이 GitHub 이슈를 분석해줘: {title} - {body}"
```

이 접근법은 세 가지 근본적인 문제를 안고 있었습니다.

1. **응답 형식 불일치**: 매 호출마다 다른 구조의 자유 텍스트 반환
2. **파싱 불가**: 타입 안전한 downstream 처리 불가능
3. **에러 처리 불가**: 누락 필드에 대한 방어 로직 없음

---

### After: 구조화된 프롬프트 시스템

**1단계 — 응답 스키마 정의 (AnalysisResult interface)**

프롬프트보다 먼저 _기대 출력의 타입_을 정의합니다. 이 타입이 프롬프트 설계의 기준이 됩니다.

```typescript
interface AnalysisResult {
  summary: string;
  type: string;           // Bug, Feature Request, Enhancement, Documentation, ...
  priority: string;       // High / Medium / Low
  priority_reason: string;
  technical_points: string[];
  challenges: string[];
  suggested_approach: string[];
  related_areas: string[];
  questions: string[];
}
```

**2단계 — 시스템 컨텍스트 주입 (CONFIG + DEFAULT_PROJECT_CONTEXT)**

모델이 프로젝트 구조를 이해하고 일관된 판단을 내리도록 고정 컨텍스트를 주입합니다.

```typescript
const CONFIG = {
  anthropicApiKey: process.env.ANTHROPIC_API_KEY,
  githubToken: process.env.GITHUB_TOKEN,
  githubRepo: process.env.GITHUB_REPOSITORY,
  model: process.env.CLAUDE_MODEL || 'claude-sonnet-4-20250514',
  maxTokens: 8000,
};

const DEFAULT_PROJECT_CONTEXT = `oh-my-customcode is an npm package for customizing Claude Code.
Key components: Agents (42), Skills (55), Rules (19), Guides (22).
Commands: omcustom init, list, doctor.
Tech: TypeScript/Bun, GitHub Actions, npm.`;
```

**3단계 — 프롬프트 4계층 구조**

```
┌─────────────────────────────────────────┐
│  System Context                          │
│  ├── Project description                 │
│  └── Component inventory                 │
├─────────────────────────────────────────┤
│  Issue Details                           │
│  ├── Number, Title, Author, Labels       │
│  └── Body (full text)                    │
├─────────────────────────────────────────┤
│  Analysis Instructions                   │
│  ├── 8 structured fields                 │
│  ├── JSON response format                │
│  └── Language rules (한/영 혼용)          │
├─────────────────────────────────────────┤
│  Output Format                           │
│  └── Typed JSON (AnalysisResult)         │
└─────────────────────────────────────────┘
```

**4단계 — 응답 파싱 및 방어적 검증**

```typescript
// Robust JSON parsing: 마크다운 코드 블록 래핑 제거
let jsonStr = rawContent.trim();
const jsonMatch = jsonStr.match(/```json?\s*\n([\s\S]*?)\n```/);
if (jsonMatch) jsonStr = jsonMatch[1];

const analysis: AnalysisResult = JSON.parse(jsonStr);

// Array field validation guard: 모델이 빈 배열 대신 null 반환 시 방어
const arrayFields = [
  'technical_points',
  'challenges',
  'suggested_approach',
  'related_areas',
  'questions',
] as const;

for (const field of arrayFields) {
  if (!Array.isArray(analysis[field])) {
    analysis[field] = [];
  }
}
```

---

### 핵심 개선점 비교

| 항목 | Before | After |
|------|--------|-------|
| 응답 형식 | 자유 텍스트 | JSON 스키마 강제 |
| 파싱 | 불가 | 타입 안전 파싱 (`AnalysisResult`) |
| 에러 처리 | 없음 | 배열 필드 가드 + 코드 블록 제거 |
| 컨텍스트 | 없음 | 프로젝트 구조 + 컴포넌트 목록 주입 |
| 재현성 | 낮음 | 높음 (모델/토큰/프롬프트 고정) |
| 다운스트림 처리 | 수동 파싱 필요 | `analysis.priority` 직접 참조 가능 |

---

### 설계 원칙

이 패턴에서 도출한 구조화 프롬프트 설계의 3가지 원칙입니다.

1. **출력 타입 먼저 정의**: 프롬프트보다 `interface`를 먼저 작성하면 요구사항이 명확해집니다.
2. **컨텍스트는 고정 주입**: 매 호출마다 동일한 시스템 컨텍스트를 주입하여 모델의 판단 기준을 안정화합니다.
3. **방어적 파싱 필수**: 모델 응답은 언제나 예상을 벗어날 수 있습니다. `try-catch`, 배열 가드, 코드 블록 제거를 기본 장착합니다.

---

## 3. AI 산출물 전후 비교

### 배경: codex-exec 래퍼 — 20라운드 크로스 모델 검증

**저장소**: https://github.com/baekenough/baekenough-skills

oh-my-customcode의 `codex-exec` 스킬을 독립 레포로 분리하여 skills.sh 마켓플레이스에 배포했습니다. 배포 전 **20라운드의 크로스 모델 검증** (Codex xhigh 10라운드 + Claude Opus 10라운드)을 통해 7개 버그를 발견하고 전량 수정했습니다.

---

### 검증 결과 요약

| 메트릭 | 값 |
|--------|-----|
| 총 검증 라운드 | 20 (Codex xhigh 10 + Claude Opus 10) |
| 발견된 이슈 | 7개 |
| 수정 완료 | 7/7 (100%) |
| 크로스 플랫폼 지원 | macOS, Linux, Windows |
| 마켓플레이스 배포 | skills.sh 라이브 |

---

### 핵심 Before/After 비교

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

### 크로스 모델 검증의 의의

단일 모델 검증으로는 모델 특유의 사각지대가 존재합니다. **Codex xhigh**는 코드 실행 관점의 엣지 케이스에 강하고, **Claude Opus**는 설계 의도와 철학적 일관성 검토에 강합니다. 두 모델을 교차 검증함으로써 더 넓은 결함 탐지 범위를 확보했습니다.

| 모델 | 강점 | 발견 이슈 유형 |
|------|------|---------------|
| Codex xhigh | 코드 실행 엣지 케이스 | SIGKILL 타이밍, stdin hang, exit code 혼입 |
| Claude Opus | 설계 일관성, cross-platform | Windows 경로 처리, agent-coupled 언어 제거 |

7개 이슈 전량 수정 후 skills.sh 마켓플레이스에 라이브 배포하였으며, `npx skills add baekenough/codex-exec` 명령으로 즉시 설치 가능합니다.

---

## 4. 개발 워크플로 통합

AI를 단순한 코드 생성 도구가 아니라 **개발 파이프라인의 능동적 참여자**로 통합한 경험입니다.
oh-my-customcode는 12개의 GitHub Actions 워크플로우와 훅 시스템을 통해 AI 에이전트가 CI/CD 전 주기에 관여합니다.

### 4.1 GitHub Actions 워크플로우 구성 (12개)

| 번호 | 워크플로우 | 역할 | AI 관여 |
|------|-----------|------|---------|
| 1 | `ci.yml` | lint, test, build | - |
| 2 | `release.yml` | npm 릴리스 자동화 | - |
| 3 | `issue-analyzer.yml` | 이슈 자동 분석 및 코멘트 | Claude API |
| 4 | `claude-native-check.yml` | Claude Code 네이티브 호환성 검증 | Claude API |
| 5 | `security-audit.yml` | 보안 취약점 감사 | - |
| 6 | `docs-sync.yml` | 문서 동기화 | - |
| 7 | `docs-validator.yml` | 문서 유효성 검증 | - |
| 8 | `release-notes.yml` | 릴리스 노트 자동 생성 | Claude API |
| 9 | `deploy-test.yml` | 배포 테스트 | - |
| 10 | `reusable-claude-native.yml` | 재사용 가능 Claude 네이티브 체크 | Claude API |
| 11 | `reusable-daily-report.yml` | 재사용 가능 일일 리포트 | Claude API |
| 12 | `reusable-issue-analyzer.yml` | 재사용 가능 이슈 분석 | Claude API |

### 4.2 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│  GitHub Events                                               │
│  ├── push/PR -> ci.yml + claude-native-check.yml            │
│  ├── issue created -> issue-analyzer.yml (Claude API)       │
│  ├── release tag -> release.yml + release-notes.yml         │
│  └── schedule -> security-audit.yml + docs-sync.yml         │
├─────────────────────────────────────────────────────────────┤
│  Reusable Workflows (DRY)                                    │
│  ├── reusable-claude-native.yml                              │
│  ├── reusable-daily-report.yml                               │
│  └── reusable-issue-analyzer.yml                             │
├─────────────────────────────────────────────────────────────┤
│  Local Hooks (.claude/hooks/hooks.json)                      │
│  ├── PreToolUse: stage-blocker, git-delegation-guard         │
│  ├── PostToolUse: prettier, tsc, gofmt, ruff, console.log   │
│  └── Stop: final console.log audit                           │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 로컬 훅 시스템

CI/CD 워크플로우 외에도, AI 에이전트가 코드를 작성하는 **로컬 편집 단계**에서도 자동화가 작동합니다.
훅은 AI가 파일을 수정할 때마다 실행되어 포매팅, 타입 검사, 규칙 준수를 실시간으로 강제합니다.

**훅 종류 및 역할**:

| 훅 | 트리거 | 역할 |
|----|--------|------|
| `stage-blocker` | Write/Edit | 개발 단계별 도구 사용 제한 |
| `git-delegation-guard` | Task spawn | R010 git 위임 규칙 준수 검증 |
| `dev-server-blocker` | Bash (`npm run dev`) | tmux 외부 dev 서버 실행 차단 |
| `prettier` / `gofmt` / `ruff` | PostToolUse Edit | 언어별 자동 포매팅 |
| `tsc check` | PostToolUse Edit (`.ts`) | TypeScript 타입 체크 |
| `console.log detector` | PostToolUse / Stop | `console.log` 잔류 경고 |

**훅 구현 예시 — `git-delegation-guard.sh`**:

```bash
#!/bin/bash
# R010 git-delegation-guard hook
# Warns when git operations are delegated to a non-mgr-gitnerd agent via Task tool.

input=$(cat)
agent_type=$(echo "$input" | jq -r '.tool_input.subagent_type // ""')
prompt=$(echo "$input" | jq -r '.tool_input.prompt // ""')

if [ "$agent_type" != "mgr-gitnerd" ]; then
  git_keywords=("git commit" "git push" "git revert" "git merge" "git rebase"
                 "git checkout" "git branch" "git reset" "git cherry-pick" "git tag")

  for keyword in "${git_keywords[@]}"; do
    if echo "$prompt" | grep -qi "$keyword"; then
      echo "[Hook] WARNING: R010 violation - git op delegated to '$agent_type'" >&2
      echo "[Hook] All git ops MUST go to mgr-gitnerd per R010" >&2
      break
    fi
  done
fi

echo "$input"  # Always pass through (advisory only)
```

이 훅은 AI 에이전트가 git 작업을 잘못된 에이전트에 위임할 때 실시간으로 경고를 출력합니다.
`echo "$input"`으로 항상 통과(pass-through)시켜 경고가 작업 흐름을 차단하지 않도록 설계했습니다.

### 4.4 핵심 개선 효과

- **이슈 트리아지 자동화**: 이슈 생성 즉시 Claude API가 분석 코멘트를 달아 분류 시간 단축
- **릴리스 노트 자동 생성**: 태그 push 시 커밋 히스토리 기반 릴리스 노트 자동 작성
- **보안 지속 감시**: 주기적 `npm audit` CI + 자동 보안 PR 생성
- **규칙 실시간 강제**: 코딩 규칙 위반을 훅이 편집 시점에 즉각 감지

---

## 5. AI 한계 극복 경험

AI를 실제 프로덕션 시스템에 통합하면서 마주친 **3가지 반복 패턴의 AI 한계**와 이를 체계적으로 극복한 경험입니다.
문제별로 탐지 메커니즘 → 수정 프로세스 → 예방 조치의 3단계 대응 체계를 구축했습니다.

### 5.1 구조적 드리프트 (Structural Drift)

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

### 5.2 코드 결함 (Code Defects)

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

### 5.3 보안 취약점 (Security Vulnerabilities)

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

### 5.4 AI 오류 유형별 탐지/수정 프로세스 요약

| AI 오류 유형 | 탐지 메커니즘 | 수정 프로세스 | 예방 조치 |
|-------------|-------------|-------------|----------|
| 구조적 드리프트 | mgr-sauron 8라운드 검증 | 자동 수정 + 재검증 | R017 push 전 필수 검증 |
| 코드 결함 | 크로스 모델 검증 (Codex + Claude) | Before/After 비교 수정 | multi-model-verification 스킬 |
| 보안 취약점 | security-audit CI + hooks | `npm audit fix` + overrides | R001/R002 강제 규칙 |
| 규칙 위반 | git-delegation-guard hook | 실시간 경고 + 흐름 차단 | R010 오케스트레이터 규칙 |
| 문서 불일치 | docs-sync CI + mgr-updater | 자동 동기화 | docs-validator 검증 |

---

## 6. LLM 기반 기능 개발

LLM을 단순 사용하는 것을 넘어, **LLM이 더 잘 작동하도록 돕는 인프라**를 직접 설계하고 구현한 경험입니다.
두 가지 핵심 시스템을 소개합니다.

### 6.1 ontology-rag — 하이브리드 검색 엔진

oh-my-customcode의 핵심 컨텍스트 검색 엔진입니다.
136개 컴포넌트(에이전트, 스킬, 규칙, 가이드)를 그래프로 모델링하고, 쿼리에 대해 가장 적합한 컨텍스트를 동적으로 선택해 LLM에 제공합니다.
전체 컨텍스트를 프롬프트에 넣는 대신, 관련성 높은 부분만 선별 로딩해 토큰을 75-95% 절감합니다.

**4-signal 하이브리드 검색 가중치**:

```python
class HybridSearcher:
    # Scoring weights
    KEYWORD_WEIGHT = 0.50    # 키워드 매칭
    GRAPH_WEIGHT = 0.30      # 그래프 근접도 (BFS depth)
    COMMUNITY_WEIGHT = 0.15  # 커뮤니티 관련성 (Jaccard)
    IMPORTANCE_WEIGHT = 0.05 # 전역 중요도 (PageRank)
```

**최종 점수 계산 공식**:

```
final_score = 0.50 * keyword_score
            + 0.30 * graph_score
            + 0.15 * community_score
            + 0.05 * importance_score
```

**각 시그널 설명**:

| 시그널 | 가중치 | 알고리즘 | 역할 |
|--------|--------|---------|------|
| Keyword | 50% | 역인덱스 매칭 | 직접적 텍스트 유사도 |
| Graph | 30% | BFS 깊이 기반 `1/(depth+1)` | 구조적 근접도 |
| Community | 15% | Jaccard 유사도 | 관련 커뮤니티 멤버십 |
| Importance | 5% | PageRank 정규화 | 전역 노드 중요도 |

가중치 설계 근거: 대부분의 쿼리는 키워드가 명확하므로 Keyword에 높은 비중을 두되, 그래프 구조가 없으면 에이전트-스킬 간 연결 관계를 놓치므로 Graph를 30%로 유지했습니다.

**토큰 예산 시스템 (`schema.yaml` loading_levels)**:

쿼리 복잡도에 따라 5단계로 로딩 범위를 조절해 불필요한 토큰 소비를 방지합니다.

```yaml
loading_levels:
  level_0:
    description: "Category summaries only"
    target_tokens: 500
  level_1:
    description: "Relevant entity summaries"
    target_tokens: 800
  level_2:
    description: "Selected entity full content"
    target_tokens: 1500
  level_3:
    description: "Skill details"
    target_tokens: 600
  level_4:
    description: "Full skill content (on-demand)"
    target_tokens: 2000
```

**성능 최적화: Python + Rust 하이브리드**

대규모 노드 스코어링에서 Python의 속도 한계를 보완하기 위해 Rust 백엔드를 도입했습니다.

```python
# Python에서 Rust 배치 스코어링 호출
scores = _rust_backend.batch_hybrid_score(
    node_ids, keyword_hits, graph_depths, community_memberships
)
# Rust 미탑재 환경에서는 Python fallback 자동 전환
```

---

### 6.2 mcp-sage — 가이드라인 MCP 서버

AI 에이전트가 코딩 가이드라인을 실시간으로 조회할 수 있도록 **MCP(Model Context Protocol) 서버**를 직접 구현했습니다.
시스템 프롬프트에 모든 규칙을 정적으로 포함시키는 방식 대신, 에이전트가 필요한 규칙만 동적으로 쿼리하는 구조입니다.

**MCP 도구 테이블**:

| 도구 | 설명 | 활용 예시 |
|------|------|----------|
| `list_guidelines` | 모든 가이드라인 목록 반환 | 가용 가이드라인 전체 탐색 |
| `get_index` | 가이드라인 목차 반환 | 카테고리 구조 파악 |
| `get_category` | 카테고리 상세 정보 반환 | 특정 도메인 규칙 일괄 조회 |
| `get_rule` | 개별 규칙 상세 내용 반환 | 정밀 규칙 참조 |
| `search_rules` | 키워드 기반 규칙 검색 | 상황별 관련 규칙 탐색 |

**설계 철학**: 정적 시스템 프롬프트에 규칙 전체를 포함하면 토큰 비용이 고정적으로 발생합니다.
mcp-sage는 에이전트가 필요한 시점에 필요한 규칙만 pull하므로, 단순 작업 시 가이드라인 관련 토큰 소비가 0에 가깝습니다.

---

### 6.3 정량 성과 요약

| 메트릭 | 값 | 비교 기준 |
|--------|-----|----------|
| 토큰 절감율 | 75-95% | 전체 컨텍스트 로딩 대비 |
| 검색 시그널 수 | 4개 | keyword, graph, community, PageRank |
| 로딩 레벨 | 5단계 | 500 ~ 2,000 토큰 |
| Rust 가속 배율 | ~10x | Python fallback 대비 |
| MCP 도구 수 | 5개 | list, index, category, rule, search |
| 관리 컴포넌트 | 136개 | 에이전트 + 스킬 + 규칙 + 가이드 전체 |

---

## 마무리

### 기술 스택 요약

| 영역 | 기술 |
|------|------|
| 언어 | TypeScript, Python, Rust, Bash |
| AI/LLM | Claude API (Anthropic SDK), OpenAI Codex CLI |
| 프레임워크 | Bun, MCP (Model Context Protocol) |
| CI/CD | GitHub Actions (12 workflows) |
| 검색 엔진 | Hybrid Search (keyword + graph + community + PageRank) |
| 패키지 관리 | npm, PyPI |
| 프로토콜 | MCP (Model Context Protocol) |

---

### 핵심 역량

1. **AI 시스템 설계**: 41개 에이전트 오케스트레이션, 136개 컴포넌트 자동 동기화
2. **프롬프트 엔지니어링**: 구조화된 JSON 스키마 응답, 크로스 모델 검증
3. **AI 품질 관리**: 8라운드 자동 검증, 훅 기반 실시간 규칙 강제, 0 vulnerability 보안

---

*이 포트폴리오의 모든 코드는 공개 GitHub 레포지토리에서 확인할 수 있습니다:*

- **oh-my-customcode**: [github.com/baekenough/oh-my-customcode](https://github.com/baekenough/oh-my-customcode)
- **baekenough-skills**: [github.com/baekenough/baekenough-skills](https://github.com/baekenough/baekenough-skills)
