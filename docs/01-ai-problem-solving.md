# 1. AI 활용 문제 해결 사례

> ← [목차로 돌아가기](../README.md)

## 배경: oh-my-customcode 에이전트 프레임워크

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

## 문제: 136개 컴포넌트 간 구조적 드리프트

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

## 해결: mgr-sauron 자동 검증 시스템 (R017)

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

## Quick Verification Commands

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

## 정량 지표

| 메트릭 | 이전 | 이후 |
|--------|------|------|
| 수동 검증 시간 | 30분 이상 | 0분 (완전 자동화) |
| 검증 라운드 | 없음 | 8라운드 (5 + 3) |
| GitHub Actions 워크플로우 | 0 | 12개 통합 |
| 주간 컴플라이언스 체크 | 수동 | 자동 |
| 드리프트 탐지 지연 | 며칠~수주 | 즉시 (커밋 전) |

---

## AI 기여도 분석

| 작업 영역 | 인간 기여 | AI 기여 |
|-----------|-----------|---------|
| 시스템 설계 (아키텍처, 규칙 체계) | 100% | 0% |
| 에이전트 파일 작성 (41개) | 30% | 70% (Claude 지원) |
| 검증 로직 구현 | 50% | 50% (Claude 지원) |

핵심 인사이트: AI가 반복적인 파일 작성을 지원하는 동안, 인간은 시스템의 철학과 규칙 설계에 집중할 수 있었습니다. **"전문가가 없으면 만들고, 지식을 연결하고, 사용한다"**는 프레임워크 철학 자체가 AI와의 협업으로 완성되었습니다.
