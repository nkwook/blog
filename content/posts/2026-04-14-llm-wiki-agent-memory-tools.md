---
title: "Karpathy LLM Wiki 이후 — AI Agent 메모리 도구 6종 비교 (2026)"
date: 2026-04-14T14:00:00+09:00
draft: false
tags: [llm-wiki, agent-memory, knowledge-graph, mem0, basic-memory, graphify, graphiti, lightrag, rag, ai-agent]
categories: [research]
summary: "LLM과 대화할수록 지식이 쌓여야 하는데 실제로는 매번 같은 맥락을 다시 설명한다. 2026년 들어 이 문제를 푸는 도구들이 동시에 터졌다. Karpathy LLM Wiki 패턴 + basic-memory · graphify · Graphiti · mem0 · LightRAG를 카테고리·원리·장단점·인기도 관점에서 비교한 선택 가이드."
description: "2026년 AI Agent 메모리 도구 6종(LLM Wiki, basic-memory, graphify, Graphiti, mem0, LightRAG) 비교. 카테고리 구분, 슬롯별 권장 매트릭스, 선택 질문 4개."
keywords: ["AI Agent Memory", "LLM Wiki", "Karpathy", "mem0", "basic-memory", "graphify", "Graphiti", "Zep", "LightRAG", "Knowledge Graph", "GraphRAG", "2026"]
cover:
  image: ""
  alt: "AI Agent Memory 도구 지형도"
  hidden: true
---

## 대화가 매번 0에서 시작되지 않기를

LLM과 나누는 대화가 **매번 0에서 시작되지 않고, 이전 세션 위에 계속 쌓여서 컨텍스트와 내러티브가 증강되었으면** 하는 생각을 자주 한다. 같은 프로젝트 얘기를 할 때마다 "내가 누구고, 뭐에 관심 있고, 지난 분기에 무슨 결론을 냈는지"를 재설명하지 않는 것. 3개월 전에 쌓아둔 사고가 오늘 질문에 자연스럽게 묻어 나오는 것. 이게 가능해지면 LLM과의 협업은 하루치 대화가 아니라 **compounding 되는 관계**가 된다.

"그걸 파일로 적어두면 되잖아"라는 답은 절반만 맞다. 수동으로 적으면 쌓이는 건 raw 노트이지 정제된 지식이 아니다. 6개월 뒤에 다시 찾을 수 있는 건 grep이 잡는 키워드뿐이고, 그동안 생각이 어떻게 진화했는지는 여전히 머리 속에만 있다. raw와 compiled 지식의 간극을 누가 메우느냐 — 그게 요즘 AI agent memory 논의의 핵심 질문이다.

2026년 들어 이 문제를 비슷한 방식으로 푸는 시도가 동시에 터졌다. 원형은 Andrej Karpathy의 [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) gist. 4월 10일에는 Y Combinator CEO Garry Tan이 자신의 개인 지식 시스템 **gbrain**을 MIT 라이선스로 공개해 첫 24시간 만에 5,400★을 모았다. gbrain의 핵심 구조는 **"현재 compiled 사실 + append-only 타임라인"** 이중 구조 — 각 지식 페이지 상단에 현재 정제된 입장이, 하단에 그 입장이 형성된 경과가 시간순으로 쌓인다. Git working tree와 commit history의 관계와 같다.

이 패턴을 기준으로 두고, 그 위에 얹히는 5개 도구 — basic-memory, graphify, Graphiti, mem0, LightRAG — 가 각각 어느 문제를 풀고 안 푸는지 정리한다.

---

## 가장 흔한 혼동 — wiki 빌더 ≠ 에이전트 메모리

먼저 카테고리를 깔끔하게 나눠둘 필요가 있다.

**Wiki 빌더**는 raw 노트·문서에서 사람이 읽는 compiled 지식 페이지를 생성·유지한다. 출력물은 마크다운. basic-memory, graphify, LightRAG가 여기.

**에이전트 메모리 레이어**는 LLM과 앱 사이에 sit해서 대화 자체를 분석하고 사용자 선호·사실을 기억한다. 출력물은 에이전트가 참조하는 구조화 state. mem0가 여기.

둘은 함께 쓸 수도 따로 쓸 수도 있다. "mem0가 제일 인기 있으니 지식 베이스용으로 쓰자"는 카테고리 혼동이다. 반대로 "basic-memory로 에이전트 세션 메모리를 관리하자"도 마찬가지.

---

## 6개 도구

### 1. Karpathy LLM Wiki — 패턴

도구가 아니라 설계 패턴이다. gist 한 장이 전부. 3-layer 구조 (raw 불변 / wiki LLM-owned / schema CLAUDE.md) + `index.md`(카탈로그) + `log.md`(시맨틱 로그) + 세 연산 (Ingest · Query · Lint).

강점은 인프라 의존이 0이라는 것. 마크다운 + git + LLM이면 끝이다. Obsidian이 곧 IDE, LLM이 programmer, wiki가 codebase가 된다. 2026년 트렌드의 진앙이고, 아래 나오는 대부분의 도구가 이 gist를 직접 인용한다.

**한계**: 패턴 문서지 구현체가 아니다. 스킬·자동화는 본인이 짜야 한다.

---

### 2. basic-memory — 마크다운 위에 얹는 MCP 서버

MCP 서버 형태로 LLM이 로컬 마크다운 vault에 직접 read/write/search 한다. 저장은 마크다운 + SQLite 인덱스, 검색은 FastEmbed 로컬 임베딩 기반 hybrid. `schema_infer` / `validate` / `diff` 같은 자동 점검 도구가 내장돼 있어서 Karpathy가 말한 "lint" 연산을 그대로 구현한다.

매력 포인트는 **기존 마크다운 vault 위에 그대로 얹힌다**는 것. Obsidian·Logseq 충돌 없고 데이터는 외부로 안 나간다. 벤더 락인 0, 오프라인 OK. Claude Desktop과 Claude Code 양쪽에서 같은 MCP 설정으로 접근.

**한계**: 자체 frontmatter 컨벤션이 기존 vault와 충돌할 가능성, 한국어 임베딩 품질은 FastEmbed 모델 선택에 따라 달라 실측 필요, 대규모 운영 사례가 아직 부족. 2.9k★, 성숙하지만 작은 커뮤니티.

---

### 3. graphify — 폴더를 한 번에 지식 그래프로

아무 폴더든 넣으면 AST(코드) + Whisper(영상) + Claude subagent 병렬(문서/이미지)로 분석해 NetworkX 그래프를 만든다. Leiden community detection으로 군집을 찾고 god node, surprising connection, suggested questions를 뽑아준다. 출력은 interactive HTML, JSON, 사람이 읽는 `GRAPH_REPORT.md` 세 개. Claude Code에서 `/graphify .` 한 줄.

강점은 가벼움 (NetworkX + JSON 1개, Neo4j 같은 인프라 없음), 멀티모달, 그리고 **모든 edge에 EXTRACTED/INFERRED/AMBIGUOUS 태그**가 붙어 있다는 것. 무엇이 원문이고 무엇이 추론인지 구분된다. Claude Code 스킬이라 host agent의 subagent를 빌려 쓰므로 별도 LLM 키도 필요 없다.

**한계**: **시간축이 없다**. 1회 스냅샷이 전부. 매일 갱신 시나리오가 아니라 "폴더를 처음 탐색할 때 지도 한 장 받는" 용도에 가장 맞다. 25.2k★.

---

### 4. Graphiti (Zep) — 시간축 있는 knowledge graph

매 "episode"(대화, 문서 추가)마다 LLM이 entity·relationship·deduplication·summarization 여러 번 호출해 Neo4j(또는 FalkorDB)에 incremental로 쌓는다. 핵심은 **모든 fact에 `valid_from`/`valid_to` 시간축**이 붙는다는 것. 과거 시점의 지식 상태를 재구성할 수 있고, 사실이 invalidate된 기록도 보존된다. 검색은 BM25 + vector + graph traversal hybrid, P95 300ms.

시간축이 진짜 강점이다. 에이전트가 "작년 Q2 시점의 내 생각"으로 되돌아가 질의할 수 있다. [학술 논문(arXiv 2501.13956)](https://arxiv.org/html/2501.13956v1)으로 뒷받침. 24.9k★.

**한계는 ingest 비용**. 단일 activity가 여러 LLM 호출을 트리거한다. [Issue #1299](https://github.com/getzep/graphiti/issues/1299)의 한 사용자: "LLM extraction at the end is an overkill (slow and costly)". 실제 보고된 벤치마크: `add_episode_bulk`로 **100 records ingest에 약 1시간** (v0.28.1). 기본 `SEMAPHORE_LIMIT=10`은 rate limit 회피용 — 올릴 수는 있지만 LLM 비용이 그만큼 올라간다. Neo4j 필수.

---

### 5. mem0 — 가장 인기 있는 에이전트 메모리 레이어

앱과 LLM 사이에 sit. 대화가 들어오면 "무엇을 keep할지" LLM이 판단해 저장한다. 저장소는 하이브리드 3-store: key-value(사실·선호), graph(관계), vector(의미). 검색은 세 store를 결합한다. 52.9k★, 카테고리 1위. YC 출신.

대화 기반 에이전트에 드롭인 하기 쉽고 운영 검증 사례 많다. ChatGPT류 "사용자를 기억하는 챗봇"을 만드는 용도에 적합.

**한계**: 사람이 읽는 마크다운 페이지를 만들지 않는다. 출력은 에이전트 전용 memory entry. 마크다운 + git 워크플로우와는 다른 세계다. 지식 베이스 구축 용도 X.

---

### 6. LightRAG — GraphRAG를 가볍게

Microsoft GraphRAG의 경량화. entity·relationship 추출 + dual-level retrieval (low-level entity + high-level theme). GraphRAG 대비 비용·복잡도가 한 단계 낮다. [EMNLP 2025 Findings](https://aclanthology.org/2025.findings-emnlp.568/)에 accepted. 33.1k★.

강점은 multi-hop 질의 ("X와 Y 사이의 관계를 거슬러 올라가면?"). Neural Composer라는 Obsidian 플러그인으로 wrap돼 있어서 마크다운 vault에 바로 얹을 수도 있다.

**한계**: 여전히 LLM 호출 비용 발생. ingest 단계가 공짜 아니다(graphify보단 가볍지만). 수백 페이지 이하 규모면 grep + 단순 검색이 더 효율적이다.

---

## graphify vs Graphiti — 이름 혼동 주의

이름이 한 글자 차이지만 완전히 다른 종이다.

| 항목 | **graphify** | **Graphiti** |
|---|---|---|
| 목적 | 1회 스냅샷 분석 | Continuous memory |
| 시간축 | 없음 | 있음 |
| 저장소 | JSON 파일 | Neo4j / FalkorDB |
| 사용 패턴 | 분기 1회 건강검진 | 매 episode 갱신 |
| Ingest 비용 | 1회 폭발 후 끝 | 매번 폭발 |
| 인프라 | Python 패키지 1개 | DB 컨테이너 + LLM 키 |

"시간축이 멋지다"고 Graphiti를 고르면 ingest 비용에 발목 잡힌다. "매일 추적하고 싶다"고 graphify를 고르면 시간축이 없어서 원하는 걸 못 얻는다. 둘은 보완 관계가 아니라 다른 문제를 푼다.

---

## 인기도 한눈에 (2026-04 기준)

| 도구 | 카테고리 | ★ |
|---|---|---|
| mem0 | 에이전트 메모리 | 52.9k |
| LightRAG | 하이브리드 RAG | 33.1k |
| graphify | 스냅샷 그래프 | 25.2k |
| Graphiti | 시간축 그래프 | 24.9k |
| gbrain (참고) | 마크다운 wiki | 5k+ (공개 1주 미만) |
| basic-memory | 마크다운 wiki MCP | 2.9k |

카테고리가 맞아야 별 수가 의미를 갖는다. mem0가 52k 별이라고 지식 베이스 구축에 맞지는 않는다.

---

## 슬롯별 권장 매트릭스

Karpathy LLM Wiki 위에 시스템을 짤 때 각 슬롯에 무엇을 쓸지. 우선순위는 "가볍고 교체 가능한 것" 순서.

| 슬롯 | 1순위 | 2순위 | 도입 X |
|---|---|---|---|
| **패턴** | Karpathy LLM Wiki ✅ | — | — |
| **저장** | 마크다운 + git | basic-memory(SQLite 인덱스 추가) | Postgres류 (단일 vault엔 과함) |
| **LLM ↔ vault 접근** | Claude Code/Desktop 직접 IO | basic-memory MCP | — |
| **Ingest (raw → wiki)** | basic-memory OR 직접 스킬 | — | mem0 (다른 레이어), Graphiti (비용) |
| **Lint (모순·orphan·stale)** | basic-memory `schema_diff` OR 직접 스킬 | — | — |
| **검색 (수백 페이지 이하)** | grep / Glob | — | LightRAG (시기상조) |
| **검색 (수백+ 페이지)** | basic-memory hybrid | LightRAG / Neural Composer | Microsoft GraphRAG (오버킬) |
| **주기적 스냅샷 분석** | graphify | Microsoft GraphRAG | Graphiti (비용) |
| **시간축 thesis 추적** | `log.md` grep + LLM 해석 | Graphiti (정말 필요해질 때) | — |
| **에이전트 세션 메모리** | mem0 (별 트랙) | — | — |

핵심 원칙 하나: **"필요해지기 전에 도입하지 않는다"**. Postgres, Neo4j, LightRAG 같은 인프라 도구는 vault 규모가 실제로 요구할 때까지 기다린다. 처음엔 거의 모든 슬롯이 "마크다운 + git + grep"만으로 동작한다.

---

## 어떻게 고를까 — 4개 질문

도구를 고르기 전에 스스로에게 물어볼 것.

**Q1. 사람이 읽는 마크다운 페이지가 필요한가, 에이전트가 참조할 구조화 state가 필요한가?**
- 페이지 → wiki 빌더 (basic-memory, graphify, LightRAG)
- state → 에이전트 메모리 레이어 (mem0)

**Q2. 시간축 추적이 진짜 필요한가?**
- Yes → Graphiti. 비용 각오.
- No → graphify 스냅샷 + 단순 `log.md` grep + LLM 해석으로 90% 해결.

**Q3. 매일 갱신인가, 분기 1회 점검인가?**
- 매일 → continuous tool (Graphiti, basic-memory, mem0)
- 분기 → 스냅샷 tool (graphify, Microsoft GraphRAG)

**Q4. 규모가 얼마나 되나?**
- 수백 페이지 이하 → grep + Karpathy 패턴만으로 충분. 어떤 도구도 필수 아님.
- 수백 ~ 수천 페이지 → basic-memory의 hybrid search 또는 LightRAG
- 수만 페이지+ → Microsoft GraphRAG, Graphiti, 전용 인프라

---

## 결론

**Karpathy LLM Wiki 패턴**은 단순한 gist 한 장이지만 2026년 AI agent memory 트렌드의 출발점이다. 도구가 아닌 패턴이라는 게 오히려 강점 — 마크다운 + git + LLM만 있으면 어디서든 쓸 수 있다.

그 위에 얹는 도구는 본인 use case가 요구하는 형태에 따라 다르다. 사람이 읽는 지식 베이스가 필요하면 basic-memory나 graphify. 에이전트가 사용자를 기억하길 원하면 mem0. 시간축이 진짜 필요하면 Graphiti. multi-hop 검색이 일상적이면 LightRAG.

가장 흔한 실수는 인기도 순으로 고르는 것이다. mem0가 52k 별이라도 지식 베이스 용도엔 안 맞고, Graphiti의 시간축이 매력적이라도 매일 ingest 비용이 누적되면 유지 불가능하다. 문제를 먼저 정의하고 카테고리를 맞추는 게 맞다.

개인적 결론: **대부분의 사람에게는 Karpathy LLM Wiki 패턴 + 마크다운 + git + Claude Code가 충분하다**. 필요해지기 전에 Graphiti 같은 인프라를 도입하는 건 과투자다. 수백 페이지 넘어가서 검색이 정말 답답해질 때 basic-memory를 얹는 정도가 현실적인 경로다. 도구가 목적이 아니다. 본인 지식이 compounding 하느냐가 목적이다. 패턴이 맞으면 도구는 나중에 갈아 끼울 수 있다.

---

## 참고

- [Andrej Karpathy, LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- [basicmachines-co/basic-memory](https://github.com/basicmachines-co/basic-memory)
- [safishamsi/graphify](https://github.com/safishamsi/graphify)
- [getzep/graphiti](https://github.com/getzep/graphiti) · [Issue #1299](https://github.com/getzep/graphiti/issues/1299)
- [mem0ai/mem0](https://github.com/mem0ai/mem0)
- [HKUDS/LightRAG](https://github.com/HKUDS/LightRAG) · [EMNLP 2025 paper](https://aclanthology.org/2025.findings-emnlp.568/)
- [Zep: Temporal KG for Agent Memory (arXiv 2501.13956)](https://arxiv.org/html/2501.13956v1)
- [garrytan/gbrain](https://github.com/garrytan/gbrain)
