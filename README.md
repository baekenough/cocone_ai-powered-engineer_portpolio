# AI Powered Engineer Portfolio

> 코코네엔지니어링 AI Powered Engineer 포지션 지원 포트폴리오

| 항목 | 내용 |
|------|------|
| 지원 포지션 | AI Powered Engineer |
| GitHub | [@baekenough](https://github.com/baekenough) |
| 주요 프로젝트 | [oh-my-customcode](https://github.com/baekenough/oh-my-customcode), NuTalk (비공개) |
| 기술 스택 | TypeScript, Python, Rust, Go, Flutter, Claude API, GitHub Actions |

---

## 프로젝트 소개

### oh-my-customcode

**GitHub**: [github.com/baekenough/oh-my-customcode](https://github.com/baekenough/oh-my-customcode)

Claude Code를 커스터마이징하는 npm 패키지. 41개 에이전트, 55개 스킬, 19개 규칙, 22개 가이드 — 총 **136개 컴포넌트**로 구성된 AI 에이전트 프레임워크입니다. 오케스트레이터 패턴으로 메인 대화가 라우팅하고 서브에이전트가 실행합니다.

### NuTalk (비공개)

**데모**: [nutalk-one.vercel.app](https://nutalk-one.vercel.app/)

현대적인 UI/UX 감성 데이팅 앱. Flutter 크로스플랫폼(Android/iOS) + Go 1.25 마이크로서비스 백엔드(5개 서비스: auth, user, chat, point, moderation)로 구성됩니다. PostgreSQL+PostGIS, Redis 7, NATS JetStream을 활용하며, oh-my-customcode 에이전트 시스템(46개 에이전트, 71개 스킬)으로 개발 전 과정을 AI 오케스트레이션합니다.

| 기술 | 용도 |
|------|------|
| Flutter (Dart) | Android / iOS 크로스플랫폼 앱 |
| Go 1.25 | 마이크로서비스 5개 (auth, user, chat, point, moderation) |
| PostgreSQL 16 + PostGIS | 관계형 DB + 위치 기반 쿼리 |
| Redis 7 | 세션, OTP, 캐시 |
| NATS JetStream | 이벤트 기반 서비스 간 통신 |
| gRPC + HTTP | 서비스 간 / 클라이언트 통신 |

---

## 목차

| # | 항목 | 코코네 가이드 항목 | 핵심 |
|---|------|-------------------|------|
| 1 | [AI 활용 문제 해결 사례](docs/01-ai-problem-solving.md) | 항목 1 | 136개 컴포넌트 자동 동기화, mgr-sauron 8라운드 검증 |
| 2 | [프롬프트 템플릿](docs/02-prompt-templates.md) | 항목 2 | Claude API 구조화 프롬프트, JSON 스키마 응답 강제 |
| 3 | [AI 산출물 전후 비교](docs/03-before-after-comparison.md) | 항목 3 | codex-exec 20라운드 크로스 모델 검증, 7개 버그 수정 |
| 4 | [개발 워크플로 통합](docs/04-workflow-integration.md) | 항목 4 | 12 GitHub Actions + 훅 시스템 |
| 5 | [AI 한계 극복 경험](docs/05-ai-limitations.md) | 항목 5 | 드리프트/결함/보안 3종 대응 체계 |
| 6 | [LLM 기반 기능 개발](docs/06-llm-feature-development.md) | 항목 7 | ontology-rag 하이브리드 검색 + mcp-sage MCP 서버 |

---

## 정량 성과 하이라이트

| 메트릭 | 값 |
|--------|-----|
| 관리 컴포넌트 | 136개 (에이전트 + 스킬 + 규칙 + 가이드) |
| 자동 검증 라운드 | 8 (Manager 5 + Deep Review 3) |
| 수동 검증 시간 절감 | 30분+ → 0분 |
| CI/CD 워크플로우 | 12개 |
| 크로스 모델 검증 | 20라운드 (Codex xhigh 10 + Claude Opus 10) |
| 토큰 절감율 | 75-95% (ontology-rag) |
| NuTalk 마이크로서비스 | 5개 Go 서비스 + Flutter 앱 |
| NuTalk 에이전트 | 46개 (oh-my-customcode 기반) |

---

## 기술 스택 요약

| 영역 | 기술 |
|------|------|
| 언어 | TypeScript, Python, Rust, Go, Dart, Bash |
| AI/LLM | Claude API (Anthropic SDK), OpenAI Codex CLI |
| 모바일 | Flutter (Android + iOS) |
| 백엔드 | Go 마이크로서비스, gRPC, NATS JetStream |
| 데이터베이스 | PostgreSQL + PostGIS, Redis |
| 프레임워크 | Bun, MCP (Model Context Protocol) |
| CI/CD | GitHub Actions (12 workflows) |
| 검색 엔진 | Hybrid Search (keyword + graph + community + PageRank) |
| 패키지 관리 | npm, PyPI |

---

## 핵심 역량

1. **AI 시스템 설계**: 41개 에이전트 오케스트레이션, 136개 컴포넌트 자동 동기화
2. **프롬프트 엔지니어링**: 구조화된 JSON 스키마 응답, 크로스 모델 검증
3. **AI 품질 관리**: 8라운드 자동 검증, 훅 기반 실시간 규칙 강제, 0 vulnerability 보안
4. **풀스택 AI 개발**: Flutter + Go 마이크로서비스, AI 오케스트레이션 적용 실서비스 개발

---

*이 포트폴리오의 모든 공개 코드는 GitHub 레포지토리에서 확인할 수 있습니다:*

- **oh-my-customcode**: [github.com/baekenough/oh-my-customcode](https://github.com/baekenough/oh-my-customcode)
- **baekenough-skills**: [github.com/baekenough/baekenough-skills](https://github.com/baekenough/baekenough-skills)
- **NuTalk 데모**: [nutalk-one.vercel.app](https://nutalk-one.vercel.app/) (비공개 프로젝트)
