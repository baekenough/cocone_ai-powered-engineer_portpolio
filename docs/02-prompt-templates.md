# 2. 프롬프트 템플릿

> ← [목차로 돌아가기](../README.md)

## 배경: GitHub 이슈 자동 분석 파이프라인

**파일**: `.github/scripts/analyze-issue.ts`

oh-my-customcode 프로젝트의 GitHub Actions 워크플로우에서 새 이슈가 등록될 때 Claude API를 호출해 자동으로 분석하고 레이블, 우선순위, 기술적 접근 방향을 코멘트로 게시하는 시스템입니다.

---

## Before: 비구조적 프롬프트의 문제

```
"이 GitHub 이슈를 분석해줘: {title} - {body}"
```

이 접근법은 세 가지 근본적인 문제를 안고 있었습니다.

1. **응답 형식 불일치**: 매 호출마다 다른 구조의 자유 텍스트 반환
2. **파싱 불가**: 타입 안전한 downstream 처리 불가능
3. **에러 처리 불가**: 누락 필드에 대한 방어 로직 없음

---

## After: 구조화된 프롬프트 시스템

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

## 핵심 개선점 비교

| 항목 | Before | After |
|------|--------|-------|
| 응답 형식 | 자유 텍스트 | JSON 스키마 강제 |
| 파싱 | 불가 | 타입 안전 파싱 (`AnalysisResult`) |
| 에러 처리 | 없음 | 배열 필드 가드 + 코드 블록 제거 |
| 컨텍스트 | 없음 | 프로젝트 구조 + 컴포넌트 목록 주입 |
| 재현성 | 낮음 | 높음 (모델/토큰/프롬프트 고정) |
| 다운스트림 처리 | 수동 파싱 필요 | `analysis.priority` 직접 참조 가능 |

---

## 설계 원칙

이 패턴에서 도출한 구조화 프롬프트 설계의 3가지 원칙입니다.

1. **출력 타입 먼저 정의**: 프롬프트보다 `interface`를 먼저 작성하면 요구사항이 명확해집니다.
2. **컨텍스트는 고정 주입**: 매 호출마다 동일한 시스템 컨텍스트를 주입하여 모델의 판단 기준을 안정화합니다.
3. **방어적 파싱 필수**: 모델 응답은 언제나 예상을 벗어날 수 있습니다. `try-catch`, 배열 가드, 코드 블록 제거를 기본 장착합니다.
