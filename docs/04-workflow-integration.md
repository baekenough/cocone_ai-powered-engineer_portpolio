# 4. 개발 워크플로 통합

> ← [목차로 돌아가기](../README.md)

AI를 단순한 코드 생성 도구가 아니라 **개발 파이프라인의 능동적 참여자**로 통합한 경험입니다.
oh-my-customcode는 12개의 GitHub Actions 워크플로우와 훅 시스템을 통해 AI 에이전트가 CI/CD 전 주기에 관여합니다.

## 4.1 GitHub Actions 워크플로우 구성 (12개)

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

## 4.2 전체 아키텍처

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

## 4.3 로컬 훅 시스템

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

## 4.4 핵심 개선 효과

- **이슈 트리아지 자동화**: 이슈 생성 즉시 Claude API가 분석 코멘트를 달아 분류 시간 단축
- **릴리스 노트 자동 생성**: 태그 push 시 커밋 히스토리 기반 릴리스 노트 자동 작성
- **보안 지속 감시**: 주기적 `npm audit` CI + 자동 보안 PR 생성
- **규칙 실시간 강제**: 코딩 규칙 위반을 훅이 편집 시점에 즉각 감지
