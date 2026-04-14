---
title: "Karpathy LLM Wiki 패턴을 내 vault에 적용해봤다 — graphify + DIY skill + lint로 하루 만에 compounding wiki 만들기"
date: 2026-04-14T16:30:00+09:00
draft: false
tags: [llm-wiki, agent-memory, knowledge-graph, graphify, basic-memory, claude-code, obsidian, workflow, ai-agent]
categories: [research]
summary: "이전 글에서 AI Agent 메모리 도구 지형도를 정리했다. 이번엔 그 중 Karpathy LLM Wiki 패턴을 실제 내 vault에 적용한 실전 경험기다. graphify로 부트스트랩, 3-layer 구조 구축, ingest 스킬 3개 확장, 자가 점검 lint 스킬 1개 신설까지 한 세션에 마무리. 하루 된 wiki에서 real error 3개 잡힌 lint 결과까지."
description: "Karpathy LLM Wiki 패턴을 개인 마크다운 vault에 적용한 실전 기록. graphify 부트스트랩 결과, 3-layer 아키텍처 설계, basic-memory 평가 실패, DIY 스킬과 lint 도구 제작까지."
keywords: ["LLM Wiki", "Karpathy", "graphify", "basic-memory", "Claude Code", "Obsidian", "Knowledge Graph", "Agent Memory", "Compounding Knowledge", "Personal Wiki", "2026"]
cover:
  image: ""
  alt: "LLM Wiki pattern applied"
  hidden: true
---

## 이전 글의 후속편 — 이론에서 실천으로

어제 [AI Agent 메모리 도구 6종 비교](/blog/posts/2026-04-14-llm-wiki-agent-memory-tools/)를 정리했다. 지형도는 그려졌으니 다음 질문은 "그래서 어떤 걸 내 vault에 실제로 얹을 것인가". 결론은 이미 그 글에 썼다: **대부분의 사람에게는 Karpathy LLM Wiki 패턴 + 마크다운 + git + Claude Code로 충분하다**.

이 글은 그 결론을 실제 내 vault에 적용한 세션 기록이다. 하루 안에 3-layer 구조 설계 → graphify 부트스트랩 → ingest 스킬 확장 → lint 스킬 제작 → 첫 리포트까지 끝냈다. 중간에 basic-memory 평가가 사이드 이펙트로 끝나는 작은 사고도 있었다. 실전에서만 보이는 것들을 공유한다.

---

## 출발점 — 내 vault가 기존에 안 되고 있던 것

개인 투자 리서치 vault 기준으로 상태를 정리하면:

- 마크다운 + git. Obsidian 호환. 32개 raw 파일 (기업 분석 18 + 스터디 노트 11 + 메타 3). 총 37k words.
- `[[wiki-link]]` 몇 개 이미 달려 있음. 하지만 scattered.
- "내가 지금 이 기업에 대해 어떻게 생각하는지" 답하려면 18개 파일을 다시 읽어야 했음.
- 6개월 전 생각과 오늘 생각이 어떻게 달라졌는지 추적 불가.
- 서로 다른 파일에 흩어진 사실이 **같은 의미**인지 자동으로 알아낼 수 없음.

Karpathy 패턴의 3-layer가 정확히 이 문제를 겨냥한다:

- **Raw** — 본인이 작성한 원본. 불변. 시점 snapshot.
- **Wiki** — LLM이 maintain하는 compiled 지식 페이지. 현재 입장. 자동 갱신.
- **Schema** — 운영 규칙 문서 (CLAUDE.md / AGENTS.md).

추가로 필요한 건 세 연산: **Ingest** (raw → wiki 갱신), **Query** (wiki 우선 검색), **Lint** (모순·stale·orphan 검출). 그리고 두 보조 파일: `_index.md` (콘텐츠 카탈로그), `_log.md` (시맨틱 timeline, append-only).

이 청사진 위에 실제 구현을 얹은 게 이번 세션의 작업이다.

---

## Phase 0 — graphify로 부트스트랩

[이전 글에서 graphify](/blog/posts/2026-04-14-llm-wiki-agent-memory-tools/#3-graphify--폴더를-한-번에-지식-그래프로)는 "폴더를 한 번에 지식 그래프로"로 소개했다. 실전에서는 wiki 시드 생성기로 쓰는 게 정답이었다.

`/graphify .` 한 줄. Claude Code 세션의 subagent 2개가 병렬로 32 파일을 읽고 JSON 청크를 생성, 병합, NetworkX 그래프로 build, Leiden community detection, 분석 리포트 생성까지 한 호출에 끝냄.

### 구체 산출물

```
204 nodes · 260 edges · 15 communities
Extraction quality: 88% EXTRACTED · 12% INFERRED · 0% AMBIGUOUS
INFERRED edges: 31개 (평균 confidence 0.77)
Hyperedges: 6개 (3+ 노드 그룹 관계)
```

**88% EXTRACTED는 이 분석의 신뢰도 측정이다.** 260개 edge 중 229개가 본인 raw에 명시적으로 적힌 관계, 31개만 LLM 추론. 즉 graphify가 하는 일의 대부분은 "환각"이 아니라 **이미 적혀 있는 걸 구조화**하는 것. 12% INFERRED가 진짜 "AI가 발견해준 것"이고, 이 중 일부가 아래 설명할 surprise connections다.

### God nodes — 내 사고의 허브가 뭔지 정량 확인

| 순위 | 노드 | edge 수 |
|---|---|---|
| 1 | NVIDIA Full-Stack AI Platform | 16 |
| 2 | NVIDIA Company Index | 15 |
| 3 | **Networking & Optics** | **14** |
| 4 | NVDA Value Chain | 12 |
| 5 | NVDA Company Overview | 11 |

**3위가 의외였다.** 본인은 의식적으로 기업 본업·경쟁·valuation을 중심으로 분석한다고 생각했는데, 실제 raw에서는 **networking/optics가 valuation·moat보다 더 많이 등장**한다. graphify가 이 거울을 보여주지 않았으면 networking-optics를 wiki의 독립 theme로 만들 생각을 못 했을 것이다.

### Communities = 바로 쓸 수 있는 theme 후보

Leiden 알고리즘이 자동으로 15개 군집을 만들어줬다. 이 중 내가 본 흥미로운 것들:

- C0 Valuation & Street Consensus (36 nodes, cohesion 0.07)
- C3 Networking/Optics Ecosystem (17 nodes)
- C5 Physical AI / Robotics (15 nodes)
- C7 Fabless Supply Chain TSMC/HBM (13 nodes)
- C8 Groq/ASIC Inference Defense (13 nodes)
- C13 Taiwan Geopolitical Risk (4 nodes, cohesion 0.5)
- C14 Materials Chokepoints (3 nodes, cohesion 0.67)

cohesion이 높은 C13/C14는 작지만 단단한 주제 — 이미 wiki theme로 굳힐 준비가 된 것. 큰 C0(36 nodes)는 cohesion 0.07로 헐거워서 "더 잘 묶어야 하는 신호".

**결론적으로 wiki/themes/ 폴더의 첫 6개 페이지가 이 community 결과에서 거의 1:1 로 나왔다.** 백지에서 짜는 부담이 절반 이상 사라진다는 게 graphify의 가장 실용적 가치.

### Surprising connections — 본인이 명시 안 한 cross-community bridge

graphify가 INFERRED로 잡은 edge 31개 중 의미 있는 것 몇 개:

```
CSP Custom ASIC trend  ──semantically_similar──>  Groq LPX (NVDA $20B 인수)
Rubin: 10x lower cost per token  ──semantically_similar──>  Groq LPX
Non-GPU revenue bypasses CoWoS bottleneck
   ──semantically_similar──>  chip→system trade-off rationale
```

처음 두 edge는 **별도 community에 있던 개념들**이다. CSP ASIC(C1 Moats)과 Groq LPX(C8 Defense)가 같은 위협의 서로 다른 표현이라는 추론. Rubin의 10x cost 주장과 Groq 인수가 같은 방어 전략의 두 축이라는 추론.

내가 raw에 각각 따로 적어뒀지만 한 번도 의식적으로 연결하지 않았던 패턴. 이 두 INFERRED가 합쳐져서 wiki 분석 페이지 한 편이 생겼다 — Karpathy가 말한 "good answers can be filed back into the wiki as new pages"의 구현체.

---

## Phase 1 — wiki 골격과 시드 페이지

graphify 결과를 시드로 써서 3-layer 구조를 세팅:

```
personal/investing/
├── CLAUDE.md ←─────────── SCHEMA (운영 규칙 + 3개 page template)
│
├── journal/    ┐
├── study/      ├── RAW (불변)
├── scrapbook/  ┘
│
└── wiki/                ← LLM-owned COMPILED layer
    ├── _index.md        (카탈로그)
    ├── _log.md          (시맨틱 timeline, append-only)
    ├── companies/
    ├── concepts/
    ├── themes/
    └── analyses/
```

7개 시드 페이지를 작성했다: company 1 (NVDA) + concept 2 (HBM, CoWoS) + theme 3 (networking-optics, physical-ai-robotics, supply-crisis-taiwan) + analysis 1 (ASIC inference defense — graphify surprise를 그대로 글로 굳힘).

각 페이지 frontmatter에 `last_ingested`, `source_count` 를 넣어뒀다 (나중에 lint의 stale check가 여기를 본다).

**이 골격의 숨은 강점 하나**: wiki 페이지 구조가 graphify community와 거의 1:1이기 때문에, 매 분기 graphify를 재실행하면 "새 community가 생겼다 = 새 theme 페이지 후보" 식으로 diff만 보면 자동 운영 신호가 나온다.

---

## Phase 1.5 — basic-memory 평가의 사이드 이펙트

[basic-memory](https://github.com/basicmachines-co/basic-memory)가 혹시 Phase 2·3을 라이브러리 한 개로 대체할 수 있을까 평가했다. 결과는 미채택. 이유가 실전에서만 드러나는 종류라 짧게 공유한다.

### 기대했던 것

"마크다운 vault 그대로 위에 얹는 MCP 서버 → ingest 자동화, 검색 업그레이드, lint를 한 라이브러리에 흡수"

### 실제로 만난 것

설치 후 wiki를 project로 등록하고 `basic-memory reindex` 한 줄 실행. 이게 첫 사고였다. **`reindex`는 등록된 모든 project를 한 번에 처리한다.** `--project` 플래그가 없어 per-command 격리 불가. 그리고 config 기본값 `ensure_frontmatter_on_sync: true` + `disable_permalinks: false` 조합 때문에, 방금 만든 시드 페이지 8개 전부의 frontmatter가 자동 재작성됐다:

```diff
- tags: [investing, company, semi]
+ tags:
+ - investing
+ - company
+ - semi
- related: ["[[concepts/HBM]]"]
+ related:
+ - '[[concepts/HBM]]'
+ permalink: investing-wiki/companies/nvda
```

의미론적으로는 동등하다 (YAML parser가 읽으면 같음, Obsidian/Hugo 모두 두 형식 OK). 본문은 건드리지 않았다. 하지만 **본인이 통제해야 할 파일을 동의 없이 mutation한 것**은 사용자 신뢰의 근본 단절점이다.

거기에:
- default 임베딩 모델이 `bge-small-en-v1.5` — **영어 전용**. 한국어 semantic search 품질 X.
- basic-memory의 핵심 가치인 "semantic graph"를 쓰려면 `- [category] content #tag` / `- relation_type [[link]]` 같은 자체 bullet 포맷으로 wiki를 재작성해야 함. prose 기반 wiki와 근본 충돌.

즉시 investing-wiki project를 제거해서 추가 mutation 차단. 4 deal-breaker가 겹쳐서 **DIY 스킬이 통제권·격리·한국어·prose 자유도에서 더 나은 선택**으로 판명됐다.

### 교훈 — 자동 mutation 디폴트는 위험 신호

README에 "Markdown is source of truth, works with existing editors like Obsidian"이라고 쓰여 있지만, **기본 설정이 사용자 파일을 건드리는 도구는 사실 source of truth가 아니다**. 격리 없이 전역 처리하는 명령어도 마찬가지. 도구 평가 시 "자동 수정 옵션이 옵트-인인가 옵트-아웃인가"가 중요한 체크 항목이다.

---

## Phase 2 — ingest 스킬에 wiki 갱신 단계 추가

Claude Code의 슬래시 커맨드 3개를 확장했다:

- `/investment-study new` — 스터디 노트 작성 후 **관련 wiki 페이지 갱신 제안**
- `/investment-scrap <url>` — 스크랩 정리 후 **관련 wiki 페이지 갱신 제안**
- `/trade-journal <ticker> <action>` — 매매 기록 후 **wiki Thesis와의 정합성 체크 + 불일치 시 변경 이력 섹션 갱신 제안**

각 커맨드의 기존 단계에 "Wiki update" 섹션이 마지막에 추가되는 구조. 실행 흐름:

1. raw 노트 작성 (기존 동작)
2. 언급된 entity/concept/theme grep → wiki 페이지 후보 식별
3. 기존 페이지면 thesis/facts/risks/Recent Updates 섹션 갱신 제안 (diff로 사용자 확인)
4. 없는 페이지면 새로 만들지 물어봄
5. `_index.md` source count / 한 줄 요약 갱신
6. `_log.md`에 한 줄 append: `## [YYYY-MM-DD] ingest | {제목} → updated: ...`

**원칙 두 개**:
- **raw는 절대 수정 안 함** (immutable — 나중의 thesis 변경이 과거 시점을 왜곡하면 안 됨)
- **모든 wiki 갱신은 사용자 확인 후 적용** (자동 hook 아님 — 통제 유지)

가장 흥미로운 건 `/trade-journal`의 `action` 모드. 매매 실행은 **thesis 변경 신호인 경우가 많다**. 그래서 매매 근거가 기존 wiki Thesis와 일관되는지 확인하고, 불일치면 wiki 페이지의 `## Thesis 변경 이력` 섹션에 `- YYYY-MM-DD: {old} → {new}, 근거: ...` 를 기록한다. raw 매매 파일은 그대로 두고.

이게 Karpathy의 "compiled + append" 패턴의 핵심: 현재 compiled 입장은 갱신되지만, 과거 timeline은 보존. git working tree와 commit history의 관계와 같다.

---

## Phase 3 — /wiki-lint 스킬 + 첫 실행

[이전 글](/blog/posts/2026-04-14-llm-wiki-agent-memory-tools/)에서 "lint는 작은 wiki에서도 즉시 가치"라고 썼는데, 이번에 그걸 검증했다. 7페이지짜리 1일 된 wiki에서 실제로 가치 있는 결과가 나왔다.

### 7개 check

1. **Broken placeholder links** — `[[link]]`가 가리키는 파일 존재 검증
2. **Orphan pages** — inbound link 0인 페이지
3. **Missing concept pages** — raw에 자주 등장하지만 wiki 없는 것
4. **Stale pages** — last_ingested 60일+ AND 새 raw 유입
5. **Thesis vs trade mismatch** — wiki 입장과 매매 방향 정합성
6. **Frontmatter integrity** — 필수 필드 누락
7. **Low-activity pages** — source_count ≤ 1

자동 수정 X, 보고서만. 월 1회 또는 분기 1회 수동 호출 가정.

### 첫 실행 결과 — 기대보다 유용했다

| Check | 건수 | 실용성 |
|---|---|---|
| 1A. Broken links (real errors) | **3** ⭐ | 즉시 수정 대상. 수동으론 못 잡음 |
| 1B. Missing target pages | 4 (17 참조) | 알고 있던 pending |
| 6. Frontmatter 결손 | 1 ⭐ | template 버그까지 드러남 |
| 2. Orphans | 0 | — |
| 4. Stale / 5. Trade mismatch / 7. Low-activity | 0 / N/A / 0 | 작은 wiki엔 noise |

3개 real error 중 하나는 이런 것:

```diff
- [[robotics-study|2026-04-11-robotics-study]]
+ [[2026-04-11-robotics-study|robotics 스터디]]
```

Obsidian `[[target|display]]` 문법을 역순으로 썼다. 내가 시드 페이지 작성하면서 무의식적으로 저지른 실수. 작성 시점엔 아무 이상이 없어 보였고, Obsidian에서 링크 클릭하기 전에는 안 드러났을 오류. **lint가 사전 경고 역할을 정확히 수행**했다.

또 다른 발견 하나: analysis 페이지 template에 `last_ingested` 를 뺐던 것 (나는 "analysis는 시점 고정 문서"라는 가정이었음). 이게 lint 결과 frontmatter 결손으로 잡혀서 **template 레벨 버그까지 드러남**. 단순 페이지 검사가 아니라 내 운영 매뉴얼 자체의 취약점을 찾아준 셈.

**결론: lint는 7페이지, 1일 된 wiki에서도 ROI 양수**. 작을 때 일찍 도입하는 게 맞다.

---

## 한 세션에서 배운 것 5개

1. **패턴 > 도구.** Karpathy LLM Wiki 패턴은 gist 한 장이지만, 그 위에 어떤 OSS를 얹느냐는 부차. basic-memory 미채택 후에도 Phase 2·3 plan은 그대로 진행됐다. 도구는 갈아끼울 수 있지만 패턴은 load-bearing.

2. **자동 mutation 디폴트는 신뢰의 단절점.** 도구 평가 시 "사용자 파일 수정이 옵트-인인가 옵트-아웃인가"가 결정적. 옵트-아웃은 편리함을 제공하지만 통제 상실의 대가다.

3. **graphify 같은 1회성 부트스트랩 도구가 첫 시드 ROI 최대.** 백지에서 wiki 7개 수동 작성하는 부담을 god node + community가 절반 줄인다. 매일 쓰는 도구가 아니어도 1회 가치가 충분히 크면 도입 정당.

4. **lint는 스케일 전에 도입해도 가치 있다.** 7페이지에서 real error 3개 + template bug 1개 잡힘. "커지면 도입"이라는 이연 결정은 오히려 초기 품질 부채를 쌓는다.

5. **graphify community → wiki theme 매핑이 자동 diff 신호를 준다.** 분기 graphify 재실행 결과와 현재 wiki 구조를 diff하면 "새 community = 새 theme 후보"라는 저비용 신호가 나온다. 수동 점검 시간이 크게 줄 가능성이 있다.

---

## 앞으로의 모니터링 기준

이 세션 이후 무엇을 어떤 트리거로 재평가할지 미리 적어둔다.

| 신호 | 의미 | 액션 |
|---|---|---|
| wiki 페이지 30+ 도달 | grep 한계 근접 | qmd 또는 basic-memory 재평가 (옵트-아웃 mutation 해결됐는지 확인) |
| wiki 페이지 100+ 도달 | grep 완전 한계 | LightRAG via Neural Composer 도입 |
| `/wiki-lint` real error 5+ 누적 | 수동 수정 비효율 | lint에 제한적 auto-fix 추가 |
| stale check가 의미 있는 결과 내기 시작 | wiki가 정상 누적 사이클 진입 | 분기 lint 정례화 |
| "지난 분기 thesis 어떻게 바뀌었나?" 질문 자주 함 | 시간축 추적 needs 발생 | Graphiti 재평가 |
| 매매 기록 5건+ 누적 | thesis-trade mismatch check 의미 발현 | lint Check 5 활성 |
| graphify 재실행에서 새 god node 등장 | 내 사고에 새 허브 형성 | wiki 페이지 신설 |
| graphify community 분포가 크게 바뀜 | 사고 구조 자체가 진화 | wiki theme 재구조화 |

기준이 **"도구의 인기도"가 아니라 "내 vault에서 발생하는 구체 마찰 신호"**라는 게 중요하다. 2026년 트렌드가 어떻게 움직이든, 내 페이지 수 30개 넘기 전까지 새 검색 도구 도입은 과투자다.

---

## 결론 — 하루면 충분하다

이론에서 실천까지 거리가 생각보다 짧았다. Karpathy gist 한 장 읽고, 지형도 한 번 그려보고, 패턴을 내 vault에 얹고, 스킬 확장하고, lint 만들어 돌려본 것까지 **한 세션에 끝났다**. 도구 인프라는 0 — 마크다운 + git + Claude Code. 외부 의존도 0.

가장 의외의 가치는 **graphify가 내 사고 구조의 거울이 되어준 것**이다. 의식하지 못했던 두 번째 thesis 영역을 god node 분석에서 발견했고, 별도 파일에 흩어져 있던 두 개념이 같은 logic이라는 걸 INFERRED edge가 연결해줬다. 백지에서 wiki 구조를 짜는 게 아니라 **이미 존재하는 내 사고의 구조를 추출**한 것.

두 번째 가치는 **basic-memory 평가 실패가 오히려 DIY 우월성의 검증이 된 것**. 외부 도구에 맡기면 단순해질 거라는 기대가 틀렸다. 통제권·격리·언어 지원·prose 자유도 네 가지가 모두 DIY에서 이겼다.

세 번째 가치는 **lint가 작은 wiki에서도 즉시 작동**한 것. 수동으로 절대 못 잡을 alias 역순 오류 2개 + template 버그 1개가 1일 된 7페이지 wiki에서 드러났다. 자동 검사가 이연 결정이 아닌 초기 투자라는 걸 실감했다.

도구가 목적이 아니다. 본인 지식이 compounding 하느냐가 목적이다. 지금 이 글을 쓰는 시점에서 내 vault의 wiki 레이어는 7페이지뿐이다. 하지만 다음 스터디 노트부터 자동 갱신이 시작되고, 매매 기록은 thesis 변경을 자동 감지하고, lint가 주기적으로 self-check한다. **하루 만의 일회성 작업으로 vault가 compounding 모드로 전환**됐다는 게 이번 실험의 요지다.

---

## 참고

- [이전 글: AI Agent 메모리 도구 6종 비교](/blog/posts/2026-04-14-llm-wiki-agent-memory-tools/)
- [Andrej Karpathy, LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- [safishamsi/graphify](https://github.com/safishamsi/graphify)
- [basicmachines-co/basic-memory](https://github.com/basicmachines-co/basic-memory)
- [getzep/graphiti](https://github.com/getzep/graphiti)
