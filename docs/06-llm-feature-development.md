# 6. LLM 기반 기능 개발

> ← [목차로 돌아가기](../README.md)

LLM을 단순 사용하는 것을 넘어, **LLM이 더 잘 작동하도록 돕는 인프라**를 직접 설계하고 구현한 경험입니다.
두 가지 핵심 시스템을 소개합니다.

## 6.1 ontology-rag — 하이브리드 검색 엔진

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

## 6.2 mcp-sage — 가이드라인 MCP 서버

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

## 6.3 정량 성과 요약

| 메트릭 | 값 | 비교 기준 |
|--------|-----|----------|
| 토큰 절감율 | 75-95% | 전체 컨텍스트 로딩 대비 |
| 검색 시그널 수 | 4개 | keyword, graph, community, PageRank |
| 로딩 레벨 | 5단계 | 500 ~ 2,000 토큰 |
| Rust 가속 배율 | ~10x | Python fallback 대비 |
| MCP 도구 수 | 5개 | list, index, category, rule, search |
| 관리 컴포넌트 | 136개 | 에이전트 + 스킬 + 규칙 + 가이드 전체 |
