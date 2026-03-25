## Rezco AI Analyst: Orchestrator Build Requirements

This document maps every capability Rezco needs from their AI Analyst against the current orchestrator architecture (`ORCHESTRATOR_E2E_OVERVIEW.md`), identifies what exists, what's missing, and what tools, features, and integrations must be built.

Sources: Hyde Park Corner meeting transcription, Raw words working-session notes, AI Analyst for Asset Management PDF, rezco.txt analysis, rezco.com, Project Rezco Classification Tool meeting (Madelein Louw showcase).

---

### 1. Knowledge Ingestion Layer

The current orchestrator handles user messages through a chat widget and persists them via the conversations service. It has **no data ingestion pipeline**. Rezco needs the orchestrator to continuously ingest, process, and index information from multiple sources before any PM ever asks a question.

#### 1.1 Tools to Build

| Tool | Purpose | Priority |
|:---|:---|:---|
| **Google Chat Listener** | Background agent that sits in the Rezco Google Chat workspace, captures every message, article, chart, clip, and video link shared between Rob, Simon, and Sean. Files each item to the correct instrument(s) and theme(s) automatically. Must handle multi-topic conversations where speakers jump between shares and geographies. | P0 |
| **Google Meet Transcription Pipeline** | Ingest recorded Google Meet calls and other video/audio (portfolio discussions, management meetings, earnings calls). Transcribe via Whisper or equivalent. Extract structured insights: decisions, action items, sentiment, conviction signals, incomplete decisions. | P0 |
| **Bloomberg/Reuters Data Connector** | Pull charts, articles, PE ratios, earnings data, and financial metrics from Bloomberg and Reuters. Must handle the fact that Rezco cancelled expensive API contracts — explore Bloomberg terminal scraping, B-PIPE, or BLPAPI with minimal licensing. Also ingest stock exchange news service feeds. | P1 |
| **YouTube/Video Ingestor** | Given a YouTube link (e.g., Gundlach macro interview), download, transcribe, extract key macro views and tag them to relevant themes/instruments. | P1 |
| **Document Ingestor** | Bulk-load the Rezco library: books (Buffett annual letters, Patronica, Snowball), sell-side research PDFs, company earnings presentations, quarterly results, investor relations packs, CFA curriculum materials. Parse, chunk, and embed for retrieval. | P0 |
| **Sell-Side Research Ingestor** | Specifically handle broker reports received via email or downloaded from company websites. These must be ingested with a low-medium trust flag and tagged to the correct instruments. | P0 |
| **CSV/Spreadsheet Ingestor** | Accept CSV uploads for total returns, value grids, and other tabular data. Map columns to instruments using FIGI/ISIN codes. | P1 |
| **News/Article Monitor** | Continuous monitoring of financial news sources (Business Insider, CNBC, Bloomberg articles, Reuters). Surface material developments to PMs without being asked. | P2 |
| **Centralized Drive Monitor** | Monitor a central Google Drive (or equivalent) where junior research staff upload PDFs, transcripts, and presentations classified by company. Auto-ingest newly added files. | P1 |

#### 1.2 What Exists Today (Madelein's Research Hub)

The classification tool meeting revealed that a significant research hub already exists as a pilot, built over 2 years ago. This changes the ingestion picture substantially:

- **Research database with document store in Vertex AI** — analysts already add internal research reports, news articles, and sell-side reports. A grounded LLM agent already uses RAG search on this document store to query internal research (e.g., "summarize headwinds and tailwinds of X").
- **Structured data management** — the system already maintains earnings estimates (EPS, EBIT, net income), balance sheets, and cash flow statements per instrument. In-house analysts update forecasts as specific events.
- **Unstructured data management** — news articles, sell-side analyst reports, and research activities are stored and linked to instruments.
- **External data integrations already in place:**
  - **Visible Alpha** — sell-side research estimates, synced into the system.
  - **Morningstar** — classification data and research, synced daily via proxy layer and API.
  - **Data Stream** — financial data, synced daily.
- **Buy/sell recommendation workflow** — analysts produce recommendations with supporting unstructured documents. The system records recommendation type, company data, and backing reports.
- **Classification data with vector suggestions** — a vector database in Spanner suggests values for unpopulated classification fields via similarity search. Currently requires manual acceptance.
- **Two separate consoles** — a data/reporting console (classification, foundational data) and an investment team console (research, reports, earnings estimates). Both are Vue 3 with gRPC web.
- The A2A chat entrypoint handles real-time user messages — this is the **prompted query** path only.
- Conversation persistence stores chat history.
- No connectors to Google Chat, Google Meet, YouTube, or file systems.

**Key implication:** The ingestion layer does not need to be built from scratch. The research hub's document store and data integrations are a foundation to build on. The main gaps are: Google Chat/Meet ingestion, YouTube ingestion, automated curation (currently manual), and connecting the research hub to the new orchestrator's knowledge graph.

**Madelein's recommendation:** Focus on ideating valuable solutions before assessing code refactoring needs. Have Simon walk through what is actually used vs. unused features to avoid unnecessary work.

#### 1.3 Orchestrator Changes Needed

- New `IngestorAgent` sub-agent type within the ADK agent graph (`buildAgent`).
- Each data source gets its own sub-agent (Google Chat agent, Bloomberg agent, etc.) registered in the orchestrator's A2A `AgentCard` skills.
- A new **ingestion queue** alongside the existing A2A event queue `q` — data flows in asynchronously, gets processed, embedded, and indexed without blocking the chat path.
- The `agentExecutor.Execute` flow needs a branch for scheduled/background ingestion tasks, not just user-initiated messages.
- **Integration with the existing research hub:** The orchestrator must connect to the Vertex AI document store and Spanner vector database already in use. The research hub's existing RAG pipeline can be leveraged rather than rebuilt, but must be enhanced with the knowledge graph's non-linear linking (Section 2) and temporal awareness (Section 8).
- **Automation of manual curation:** The research hub currently requires manual upload, save, and categorization. Agents must automate this: monitoring trusted news sources, dropping event data into the research hub, and auto-accepting classification suggestions that meet a confidence threshold.

---

### 2. Knowledge Graph: The "Spiderweb"

This is the single most architecturally significant gap. Rezco explicitly rejects linear filing (folders, tags) in favor of a non-linear knowledge graph where every piece of information links to instruments, themes, sectors, macro factors, and other pieces of information. The Raw words document repeatedly emphasizes that architectural cleanliness at inception is critical — without a "really, really clean" core architecture, the volume of information will become disorganized and inaccessible.

#### 2.1 What Needs to Be Built

**Graph Database or Vector Store with Relational Overlays:**

- Every ingested item becomes a node: articles, chat messages, transcription segments, financial data points, PM views, meeting notes, sell-side reports, earnings call transcripts.
- Each node links to:
  - **Instruments** (via FIGI and ISIN) — e.g., a chat message about Discovery links to its FIGI and `ZAE000022331`.
  - **Themes** — e.g., "South African consumer spending", "online gambling", "banking competition", "AI benefits".
  - **Sectors** — retail, banks, property, gold miners. Must support both standard GICS sectors and custom-defined Rezco categories.
  - **Macro factors** — SA GDP, employment, offshore allocation, interest rates, PMIs.
  - **Geographies** — local vs. global, specific countries (Poland, China), regions (European industrials).
  - **Asset class** — equity, bond, cash.
  - **Other nodes** — an article about online gambling links to Mr. Price (discretionary spend competition) AND betting companies.
  - **Temporal position** — when was this created, when was it superseded, is it still current.
  - **Source type** — proprietary vs. public, internal view vs. sell-side research.
  - **Confidence/conviction** — inferred from language ("I really think" vs. "maybe worth looking at").
  - **Idea vs. Investment status** — whether this represents a low-conviction idea or a high-conviction executed investment (see Section 6).

**Entity Resolution Service:**

- When the AI encounters "Discovery" in a chat message, it must resolve this to the correct instrument (Discovery Holdings, ZAE000022331), not Discovery Channel or any other entity.
- Must handle abbreviations, colloquial names, ticker symbols.
- Leverage Madeleine's existing instrument classification system (3,000 shares, ISIN-based).
- Must parse multi-topic Google Chat/Meet conversations where speakers jump between multiple shares and geographies, mapping each segment to the correct instruments.

**Hierarchical Research Mapping:**

- Research must be mappable at different levels of granularity:
  - Individual instrument (e.g., Discovery Holdings).
  - Sector (e.g., South African retail).
  - Geography (e.g., Poland, global macro).
  - Thematic grouping (e.g., "companies sensitive to interest rate cuts").
- Logic-based groupings must allow broad research (global macro trends) to be filed and retrieved alongside instrument-specific research.

**Temporal Indexing:**

- Every node carries a timestamp.
- Retrieval must support "latest view" vs. "historical view" queries.
- The system must detect when a newer node contradicts an older one on the same instrument/theme.

#### 2.2 Existing Infrastructure to Build On

The Madelein classification tool meeting revealed that foundational components already exist:

- **Vector database in Spanner** — already used for similarity search on instrument classification. Suggests values for unpopulated fields by finding similar instruments. This can be extended to serve as the vector store for the broader knowledge graph.
- **Document store in Vertex AI** — already stores internal research reports and supports RAG search via a grounded LLM agent. This is the existing retrieval backbone for unstructured data.
- **Data source hierarchy** — the instrument database already supports multiple source values per field with a hierarchy determining which value is returned. This is a primitive version of the source weighting layer (Section 9).
- **Research database with filtering** — already filterable by instrument classification (e.g., retail sector). This is the seed for sector/theme-based retrieval.

The gap is that these components operate in isolation and linearly. The knowledge graph must unify them into a non-linear, interconnected system.

#### 2.3 Recommended Technology

| Component | Options | Existing Asset |
|:---|:---|:---|
| Graph store | Neo4j, Apache AGE (PostgreSQL extension), or Amazon Neptune | None — new build |
| Vector store (embeddings) | Extend existing Spanner vector DB, or migrate to Weaviate/Pinecone/Qdrant/pgvector | Spanner vector DB (classification similarity search) |
| Document store | Extend existing Vertex AI document store | Vertex AI document store (research reports + RAG) |
| Entity resolution | Custom NER model fine-tuned on SA financial entities + FIGI/ISIN lookup table from Madeleine's system | Existing FIGI-based instrument registry with Morningstar/Bloomberg classifications |
| Temporal layer | Bi-temporal schema (valid time + transaction time) on all nodes and edges | None — new build |

#### 2.4 Orchestrator Changes Needed

- New **Knowledge Graph Service** — a gRPC service alongside the existing Sessions and Conversations services. Must integrate with the existing Vertex AI document store and Spanner vector DB rather than replacing them.
- The orchestrator's `runner.New` configuration needs a `KnowledgeService` (analogous to the current `MemoryService` and `ArtifactService`).
- Every sub-agent tool call that retrieves context must go through the knowledge graph, not just session history.
- The `SessionEvents` table is insufficient — it stores execution history, not research knowledge. These are fundamentally different persistence concerns.
- The existing RAG search on Vertex AI must be enhanced to traverse graph relationships, not just return document matches. A query about "Discovery" should return not just documents mentioning Discovery, but also related instruments, macro themes, and linked research via graph traversal.

---

### 3. Instrument Data Infrastructure

#### 3.1 Dual Identifier System: FIGI and ISIN

The Hyde Park Corner meeting referenced ISIN as the primary identifier (used by the ops team). The Raw words sessions reference FIGI (Financial Instrument Global Identifier) as the primary key already used in the internal portfolio management system. The system must support both:

- **FIGI** — used in the portfolio management system and for global instrument identification.
- **ISIN** — used by the ops team for classification and maintained clean against delistings, splits, and corporate actions.
- A mapping layer between FIGI and ISIN must be maintained so either can be used as an entry point.

#### 3.2 What Already Exists (Madelein's Research Hub — Classification Tool Meeting)

The classification tool meeting revealed far more existing infrastructure than initially understood:

**Instrument Database (already built):**
- FIGI codes as primary identifier, with currency codes, sector classifications.
- Multiple classification sources per instrument: Morningstar, Bloomberg, user-specified custom values.
- **Data source hierarchy** — when an instrument is queried, the system uses a hierarchy to determine which source's value to return.
- Classification data includes: descriptions, sector, industry, geography, asset class.
- **Custom classification fields** for reporting: Fund House income type, Sanlam reporting categories, floating rate notes, convertible bonds. Required for due diligence and external reporting.
- **Investable groups** — instruments grouped by sector with classification logic.
- **Vector database in Spanner** suggests values for unpopulated fields via similarity search on instruments.
- Daily sync with external providers (Morningstar, Data Stream) through a proxy layer and API integration.
- ISIN-based classification covering ~3,000 shares maintained by the ops team.

**Structured Financial Data (already built):**
- Earnings estimates (EPS, EBIT, net income) maintained per instrument.
- Estimates from two sources: **Visible Alpha** (sell-side consensus) and **internal estimates** from Simon's team.
- Estimates provided yearly, half-yearly, and quarterly.
- Analysts record estimate updates as specific events in the system.
- Balance sheet and cash flow statement data maintained.

**Research and Recommendations (already built):**
- Buy/sell recommendation workflow with supporting unstructured documents.
- Research database filterable by instrument classification.
- Internal research reports stored in Vertex AI document store.
- RAG-enabled LLM agent for querying the research database.

**Two Separate Consoles:**
- **Data and reporting console** — for data/reporting team: classification maintenance, foundational data layers.
- **Investment team console** — for analysts/PMs: research, adding reports, estimating earnings.

**Technical State:**
- Vue 3 frontends with gRPC web.
- Contains legacy authentication patterns and older IAM components.
- Does NOT use newer patterns like MCP servers or REST endpoints from the BFF architecture.
- Some legacy code from 2 years ago that may need modernization.

**What needs caution:** Simon described the system as "a crazy Ferrari" — over-engineered for current needs. Madelein advised focusing on what Simon actually uses before committing to refactoring. Some features may be unused. The right approach is to have Simon screen-share and walk the team through what he finds valuable.

**What to salvage vs. rebuild:**
- **Salvage:** Instrument registry, FIGI mapping, classification data, external data integrations (Morningstar, Data Stream, Visible Alpha), Vertex AI document store, Spanner vector DB, earnings estimate infrastructure, buy/sell recommendation schema, custom classification fields, investable groups.
- **Rebuild/Enhance:** Connect to the orchestrator's knowledge graph, add temporal awareness, add conviction tracking, automate manual curation processes, modernize authentication/IAM, add the non-linear linking that the current linear filing system lacks.

#### 3.3 What Needs to Be Built (on top of existing infrastructure)

| Component | Description | Existing Asset |
|:---|:---|:---|
| **Instrument Master Service** | Central registry with FIGI primary key and ISIN secondary key. Stores: name, ticker, sector, GICS, custom Rezco categories, asset class, geography, listing status, corporate actions. | Instrument DB with FIGI, Morningstar/Bloomberg classifications, custom fields — **exists, needs orchestrator integration** |
| **Holdings Sync** | Daily sync from portfolio management system: per-fund weights, benchmark weights, relative positions. "What's our exposure to X?" | Partial — ops team maintains some data; needs automated daily sync pipeline |
| **Financial Data Store** | PE ratios, earnings, free cash flow, share prices, earnings revisions. Updated weekly minimum. | Earnings estimates exist (Visible Alpha + internal). PE/price data needs additional sourcing or Bloomberg connector |
| **Value Grid Service** | Stores Rezco's proprietary "value grids" — small tables of key metrics per share. | Does not exist — new build |
| **Model Store** | Latest quarterly financial model per company. Supports "model rolling." | Does not exist — new build |
| **Benchmark Comparison Engine** | Compare fund allocations against benchmark. Flag discrepancies (0% allocation in a 5% benchmark stock). | Does not exist — new build |
| **Classification Automation Agent** | Automate the manual acceptance workflow for vector-suggested classification values. Track unspecified values daily, surface suggestions, auto-accept above confidence threshold. | Vector suggestion exists in Spanner — needs agent automation layer |
| **Custom Classification Maintenance Agent** | Auto-suggest and update custom fields (Fund House income type, etc.) for reporting compliance. | Custom fields exist — manual maintenance needs automation |

#### 3.4 Orchestrator Changes Needed

- New `InstrumentTool`, `HoldingsTool`, `FinancialDataTool`, `ValueGridTool`, `ModelTool`, and `BenchmarkTool` registered in the ADK agent graph. The InstrumentTool must connect to the existing instrument database rather than building a parallel one.
- New `ClassificationMaintenanceAgent` and `CustomFieldAgent` — background agents that automate the currently manual data maintenance workflows in the research hub.
- These are **grounding tools** — they ensure the AI's responses reference real, current financial data rather than hallucinating numbers.
- The orchestrator's tool execution layer needs to handle structured data responses (tables, time series) not just text.
- Integration with the existing Visible Alpha and Morningstar data pipelines — the orchestrator should consume data from these sources through the research hub's existing proxy layer, not build parallel integrations.

---

### 4. Analyst Personas

Rezco wants competing analytical frameworks applied to the same data. This is not just prompt engineering — it requires distinct agent configurations with different knowledge bases, reasoning styles, and source preferences.

The Raw words document adds significant depth: personas must be trained on curated, high-quality datasets including public domain newsletters, trade decisions, books, and professional standards like the CFA curriculum. The goal is "cognitive diversity" — simulating debates between different philosophies to avoid groupthink.

#### 4.1 Personas to Build

| Persona | Knowledge Base | Reasoning Style |
|:---|:---|:---|
| **Rezco Analyst** | Rezco library, internal process docs, historical PM views, proprietary models, CFA curriculum | Bottom-up fundamental, growth-at-reasonable-price (GARP), always relative cross-sector comparison, conviction tracking, headwind/tailwind assessment |
| **Warren Buffett** | Every annual letter, Snowball biography, Buffett's public investment decisions, trade history, learnings | Long-term value, margin of safety, quality of management, circle of competence |
| **Ray Dalio** | Dalio's published work, macro positioning, Bridgewater letters | Macro-systematic, risk parity, economic machine framework |
| **Stanley Druckenmiller** | Druckenmiller interviews, macro positioning history, trade decisions | Top-down macro, liquidity focus, willingness to concentrate, momentum-aware |
| **Jeffrey Gundlach** | YouTube interviews, macro commentary, DoubleLine research | Fixed income/macro lens, contrarian, data-heavy |
| **Robin Hood Analyst** | Reddit, Twitter/X, retail investor forums | Sentiment-driven, meme-aware, crowd-positioning, FOMO signals |
| **Technical/Momentum Analyst** | Price action data, technical analysis frameworks | Chart-based, momentum recognition, price stabilization detection, relative price performance |

#### 4.2 Configurable Engagement Controls

The Raw words document emphasizes a critical concern: AI personas must not become a distraction. The system needs a "dial" to control activity levels:

- **Frequency control** — configurable per persona: how often it contributes unprompted.
- **Relevance threshold** — minimum confidence/relevance score before surfacing an insight.
- **Delivery mode** — asynchronous (clustered into morning meetings) vs. continuous background vs. real-time synchronous (future state).

#### 4.3 Phased Persona Maturity

1. **Phase 1:** Asynchronous prompts and clustered outputs (morning meetings, on-demand queries).
2. **Phase 2:** Continuous 24/7 background monitoring with configurable alerts.
3. **Phase 3 (Future):** Synchronous, real-time participation in live investment debates.

#### 4.4 Orchestrator Changes Needed

- The `buildAgent` function in `agent.go` must support **parameterized agent construction** — same orchestrator, different system prompts, different retrieval scopes, different source weighting.
- A `PersonaConfig` passed via the A2A request metadata or as a tool parameter, including engagement level settings.
- The session service needs to track which persona was used for each interaction so PMs can compare views.
- The front-end needs a persona selector or the ability to say "ask Buffett" in natural language.

---

### 5. Headwinds/Tailwinds Analysis Framework

This is a core Rezco investment philosophy that the AI must deeply understand and apply. It was extensively detailed in the Raw words document and must be a first-class capability, not just a prompt template.

#### 5.1 Concept

Rezco evaluates companies through the lens of whether they operate in a **tailwind** or **headwind** environment:

- **Tailwind environment:** Positive factors outweigh negatives. Companies are likely to underpromise and overdeliver. Management provides positive news incrementally while consistently beating expectations. The environment is "good and getting better."
- **Headwind environment:** Binary and high-risk. Management may be reluctant to disclose the full extent of bad news, instead "drip-feeding" negatives while hoping conditions improve. Phrases like "it's tough out there" or "consumers are struggling" are key indicators.

#### 5.2 What Needs to Be Built

**Sentiment and Language Analysis Engine:**

| Capability | Description |
|:---|:---|
| **Longitudinal Earnings Call Analysis** | Analyze earnings calls and reports over time for the same company. Detect shifts in management tone — from "bullish" to "defensive" or vice versa. |
| **Linguistic Cue Detection** | Track specific negative indicators: "it's tough out there," "consumers are struggling," management "hedging their bets," overly optimistic forecasts. Also track positive signals: "we continue to see improvement," underpromise language. |
| **Headwind/Tailwind Classifier** | For each covered company, maintain a running classification: net headwind, net tailwind, or transitioning. This feeds into the investment case reports and screening. |
| **Inflection Point Detection** | Scan years of historical transcripts to find inflection points — e.g., long-term projects finally "bearing fruit" — that are not yet reflected in financial models. |

#### 5.3 Orchestrator Changes Needed

- New `SentimentAnalysisTool` and `HeadwindTailwindClassifier` registered in the ADK agent graph.
- These tools must be available to the Research Agent, Comparison Agent, Report Generation Agent, and Screening Agent.
- Classification results must be persisted in the knowledge graph as first-class nodes linked to instruments and timestamped.

---

### 6. Conviction Workflow: Ideas to Investments Pipeline

The Raw words document introduces a critical workflow that was only hinted at in the transcription. This is the system for managing how investment ideas progress from low-conviction debates to high-conviction executed trades.

#### 6.1 Concept

- **Ideas** are low-conviction. They represent debated but unexecuted concepts. They sit in an "ideas archive" and risk stagnating if not actively monitored.
- **Investments** are high-conviction. They have been researched, debated, and executed (bought or sold).
- The analyst's primary role is to transition ideas from low to high conviction through continuous research and data monitoring.

#### 6.2 What Needs to Be Built

**Idea Registry:**

- Every debated investment idea gets a record: instrument(s), original thesis, date discussed, conviction level (low/medium/high), key assumptions, what would change conviction, who raised it.
- Ideas link to the original conversation/meeting where they were discussed.

**Post-Debate Monitoring:**

- After an idea is debated, the system must **continuously link the original investment logic to the ongoing stream of information**:
  - Share price movements (price stabilization, momentum shifts, relative performance).
  - Sell-side initiation reports and updated analyst research.
  - Macro data (PMIs, interest rate cuts, economic indicators).
  - News flow (thematic trends, company-specific developments).
- When external data aligns with the original thesis, the system triggers a re-evaluation prompt.

**Conviction Progression Tracker:**

- Track conviction level changes over time per idea/instrument.
- Record what caused each change (new data, price movement, management meeting, macro shift).
- Provide a clear audit trail from "idea" to "investment" (or to "archived/rejected").

**Execution Gap Detection (Naspers Case Study):**

- The Raw words document details a specific failure mode: the team identified Naspers at a discount, got distracted by geopolitical noise (Iranian war), and only achieved a 1.8% position vs. a 5% target before the price rose sharply.
- The system must detect when:
  - An idea has internal consensus ("the stock looks cheap") but no execution has occurred.
  - The share price continues to drop, reinforcing the thesis — flag as "incomplete work."
  - Time is passing without action and the opportunity window may be closing.
- Proactive alerts: "You discussed Naspers on [date], agreed it was undervalued at [price], the price is now [lower price] — this is incomplete work."

**Negative Momentum Handling:**

- A falling share price improves valuation but carries momentum risk.
- The system must distinguish between "cheap and getting cheaper" (dangerous) and "price stabilization" (potential entry signal).
- Technical analysis markers: momentum shifts, sideways movement, relative price performance vs. market.

#### 6.3 Orchestrator Changes Needed

- New `IdeaRegistry` service — a gRPC service for creating, updating, and querying ideas.
- New `ConvictionProgressionTool` — tracks and updates conviction levels, linked to the knowledge graph.
- New `ExecutionGapDetector` — background agent that scans for incomplete work (debated ideas with no subsequent execution).
- The Morning Briefing Agent must include an "incomplete work" section highlighting stale ideas where market data supports the original thesis.
- New `MomentumAnalysisTool` — assess price momentum, stabilization signals, and relative performance to inform timing of re-evaluation prompts.

---

### 7. Signal Amplification and Noise Filtering

The Raw words document introduces a sophisticated framework for distinguishing actionable signals from market noise. This is not just about filtering — it's about amplifying faint signals that traditional systems would discard.

#### 7.1 Concept

- **Noise/Beta:** The vast majority of investment reports and narratives are widely distributed, already priced into the market, and offer no competitive advantage.
- **Signal:** Faint, easily overlooked insights that are critical to investment success or failure. A PM's primary value is discerning these signals.
- **The problem:** Traditional systems use rigid filters that may strip out the very signals that matter. Human analysts themselves can be "bad filters" who remove signals they perceive as noise based on limited experience.

#### 7.2 What Needs to Be Built

| Capability | Description |
|:---|:---|
| **Signal Scoring Engine** | Score every piece of incoming information on a noise-to-signal spectrum. Factors: novelty (is this already widely known?), specificity (does it relate to a held or debated instrument?), source quality, alignment with internal theses. |
| **Manual Signal Amplification** | Allow PMs to mark specific data points as "high signal" even if the automated scoring is low. The system must learn from these manual overrides. |
| **Anchoring Bias Detector** | Specifically flag when PM behavior shows anchoring bias — using current holdings as "true north" rather than objective valuation. e.g., "You've held X for 3 years at this weight; the fundamentals have deteriorated but no position change has been made." |
| **Cognitive Bias Alerts** | Beyond anchoring, flag other behavioral biases: confirmation bias (only seeking supporting evidence), loss aversion (holding losers too long), recency bias (overweighting recent events). |
| **Contextual Opportunity Triggers** | Detect when internal consensus ("the stock looks cheap") is followed by market movements that reinforce the thesis (price continues to drop). Surface these as high-priority signals, not routine alerts. |

#### 7.3 Orchestrator Changes Needed

- New `SignalScoringTool` registered in the ADK agent graph, called by all retrieval agents before presenting results.
- New `BiasDetectionAgent` — background agent that periodically scans PM behavior patterns and portfolio positioning for cognitive biases.
- Signal scores stored as metadata on knowledge graph nodes, influencing retrieval ranking.
- The front-end must allow PMs to promote/demote signal scores with a click.

---

### 8. Contradiction and Temporal Intelligence

This is a cross-cutting capability that multiple agents depend on. It is not a single tool but an architectural layer.

#### 8.1 What Needs to Be Built

**Conviction Tracking System:**
- Every time a PM expresses a view on an instrument, record it with: timestamp, instrument, direction (bullish/bearish/neutral), confidence level (inferred from language), context (what prompted the view), source (chat, meeting, prompted query).
- Over time, build a conviction curve per instrument per PM.
- Surface when conviction has changed without explicit acknowledgment.

**Contradiction Detection Engine:**
- Compare current views against historical views on the same instrument.
- Compare PM views against portfolio positioning ("you're worried about SA macro but overweight retailers").
- Compare management statements against financial results ("management said growth was strong but earnings declined").
- Compare internal views against sell-side consensus (with appropriate skepticism weighting).
- Compare stated intentions against actual execution (see Execution Gap Detection in Section 6).

**Temporal Query Resolution:**
- When a PM asks "What do we think about Discovery?", the system must:
  1. Retrieve the most recent view first.
  2. Note if the view has changed over time and why.
  3. Flag if the view is stale (no update in X months despite material developments).
- All retrieval defaults to "most recent" unless explicitly asked for historical.

#### 8.2 Orchestrator Changes Needed

- The `SessionEvents` table needs augmentation with semantic metadata: instrument references, sentiment, confidence scores.
- A new `ConvictionService` that maintains the conviction state machine per instrument per PM.
- Contradiction detection runs as a background process, not just on-demand — results feed into the morning briefing and unprompted alerts.

---

### 9. Source Weighting and Trust Layer

The orchestrator currently treats all input equally. Rezco needs a hierarchy.

#### 9.1 Weighting Rules

| Source | Trust Level | Notes |
|:---|:---|:---|
| PM stated view (Rob, Simon, Sean) | Highest | Conviction language matters: "I really think" > "maybe" |
| Internal research (Rezco models, value grids) | High | Proprietary, maintained by the team |
| Management meetings (company executives) | Medium-High | Direct but potentially biased toward their own company; may drip-feed bad news in headwind environments |
| Bloomberg/Reuters data | Medium | Factual but requires interpretation |
| Independent research | Medium | Quality varies by source |
| Sell-side research (Morgan Stanley, etc.) | Low-Medium | Conflicts of interest; "punch shops" feed misleading info; may be deliberately misleading |
| Reddit/Twitter (Robin Hood persona) | Low | Sentiment signal only, not factual |

#### 9.2 Implementation

- Every node in the knowledge graph carries a `source_type` and `trust_weight`.
- The retrieval layer applies trust-weighted ranking when synthesizing responses.
- The system must **always distinguish** between what was provided to it (proprietary) vs. what came from the internet. This is both a compliance and an accuracy requirement.
- Sell-side research specifically requires skepticism: the system must note when sell-side views may be influenced by banking relationships or deal flow.

---

### 10. Core AI Analyst Capabilities (Sub-Agents)

These are the functional sub-agents that sit behind the orchestrator and handle specific analytical tasks. Each one maps to a skill in the A2A `AgentCard`.

#### 10.1 Research & Retrieval Agent

**What it does:** Handles prompted queries. When a PM asks "What's our latest view on Discovery?", this agent:
1. Queries the knowledge graph for all nodes linked to Discovery (FIGI + ISIN).
2. Filters by recency, prioritizes internal views over external.
3. Synthesizes across sources: latest chat discussions, last management meeting, most recent model, current financial data, any flagged contradictions.
4. Includes headwind/tailwind classification and current conviction level.
5. Returns a structured response with sources cited.

**Tools needed:**
- `KnowledgeGraphQuery` — semantic + graph traversal search.
- `InstrumentLookup` — resolve company name to FIGI/ISIN.
- `FinancialDataFetch` — current PE, earnings, price.
- `HoldingsLookup` — current portfolio weight and benchmark weight.
- `TemporalFilter` — distinguish current vs. historical views.
- `HeadwindTailwindClassifier` — current environment assessment.
- `ConvictionTracker` — current conviction level and trajectory.

#### 10.2 Comparison Agent

**What it does:** Handles relative analysis queries. "How does FirstRand compare to Standard Bank?" Rezco's process is fundamentally relative — a bank isn't just compared to other banks, but to retailers, property companies, and the opportunity cost of holding cash. Must support the "Comparability Across Companies" framework from Raw words — integrating quantitative metrics with qualitative narrative analysis.

**Tools needed:**
- All Research Agent tools, plus:
- `CrossSectorComparison` — pull data for multiple instruments across sectors. Must support both GICS and custom Rezco categories.
- `RelativeValuation` — PE relative to growth (GARP methodology), relative to sector, relative to market.
- `PortfolioContext` — "we own 3% of FirstRand and 0% of Standard Bank" changes the analysis.
- `HeadwindTailwindComparison` — compare environmental assessments side-by-side.

#### 10.3 Report Generation Agent

**What it does:** Generates structured investment case documents. Includes:
- Investment thesis (bull case / bear case).
- Headwind/tailwind environment assessment with sentiment trajectory.
- Latest quarterly results summary.
- Relevant charts and data tables.
- Management meeting notes and outstanding questions.
- Rezco's historical views and conviction trajectory.
- Idea-to-investment status and any execution gaps.

**Tools needed:**
- All Research Agent tools, plus:
- `ChartGenerator` — produce financial charts (PE over time, price vs. earnings, relative performance, conviction curve).
- `TemplateRenderer` — format output as a structured memo (PDF or markdown).
- `SentimentTrajectory` — plot management tone changes over time from earnings calls.

#### 10.4 Management Meeting Prep Agent

**What it does:** Before a PM meets company management (e.g., "I'm seeing Discovery for lunch"), this agent:
1. Pulls the full investment case.
2. Identifies gaps — what information is missing or outdated.
3. Flags contradictions — "management said X last quarter, but results show Y."
4. Assesses current headwind/tailwind environment.
5. Generates targeted questions aligned with Rezco's investment process.
6. Notes any linguistic cue shifts detected in recent earnings calls.

**Tools needed:**
- All Research Agent tools, plus:
- `GapAnalysis` — identify missing data points in the investment case.
- `ContradictionDetector` — cross-reference management statements vs. financial reality.
- `QuestionGenerator` — produce questions grounded in Rezco's philosophy.
- `LinguisticCueHistory` — surface detected tone shifts for discussion.

#### 10.5 Morning Briefing Agent (Unprompted)

**What it does:** Runs on a schedule (daily, pre-market). Produces a briefing covering:
- Overnight market moves relevant to the portfolio.
- New articles/news on held positions or watchlist names.
- Items from yesterday's Google Chat that were discussed but not resolved.
- Flags: earnings revisions (positive or negative), price movements hitting thresholds, upcoming results dates.
- Proactive nudges: "You discussed X two months ago and said it was too expensive. It's down 15% since then."
- **Incomplete work alerts:** Ideas that have stagnated, execution gaps, conviction mismatches.
- **Benchmark gap flags:** Stocks with significant benchmark weight but 0% portfolio allocation that fit Rezco's style.
- **Signal highlights:** High-scoring signals from overnight that passed the noise filter.

**Tools needed:**
- `ScheduledTaskRunner` — cron-style execution within the orchestrator.
- `PortfolioMonitor` — compare current holdings/watchlist against overnight data.
- `UnresolvedThreads` — scan recent Google Chat for open discussion threads.
- `ThresholdAlerts` — configurable price/valuation triggers.
- `NotificationDelivery` — push the briefing to Google Chat or the front-end.
- `ExecutionGapDetector` — flag incomplete work.
- `BenchmarkGapScanner` — identify underweight positions that fit the fund's profile.

#### 10.6 Quantitative Screening Agent

**What it does:** Identifies shares that fit Rezco's style criteria:
- Low PE relative to growth (GARP methodology).
- Positive earnings revisions.
- Quality metrics (ROE, free cash flow generation).
- Not already owned (or underweight relative to conviction).
- Operating in a tailwind environment.
- Benchmark comparison: flag shares with high benchmark weight that are not held and fit the Rezco style.

**Tools needed:**
- `FinancialScreener` — filter the instrument universe by quantitative criteria.
- `StyleFit` — score instruments against Rezco's defined style parameters.
- `EarningsRevisionTracker` — flag positive/negative revision trends.
- `HeadwindTailwindClassifier` — filter for tailwind environments.
- `BenchmarkComparison` — overlay benchmark weight data.

#### 10.7 Model Rolling Agent

**What it does:** Given an existing financial model for a company and the latest quarterly results, rolls the model forward. Updates revenue, earnings, margins, and valuation metrics.

**Important caveat from the meeting:** "It can roll a model... it can't quite get the logic" for building models from scratch. This agent updates existing models, it does not create them.

**Tools needed:**
- `ModelStore` — retrieve the latest model.
- `FinancialResultsParser` — extract key figures from quarterly results.
- `ModelUpdater` — apply new figures to the existing model structure.
- `ValidationCheck` — sanity-check outputs against historical patterns.

#### 10.8 Incomplete Work Monitor Agent (Background)

**What it does:** Continuously scans for stagnant ideas and execution gaps. This is the agent that prevents the Naspers scenario from recurring.

- Scans the idea registry for ideas older than a configurable threshold with no conviction update.
- Cross-references stagnant ideas against current market data to detect if the thesis is being validated.
- Detects "distraction drift" — when the team was on track to execute but external noise (geopolitical events, unrelated macro) caused loss of focus.
- Generates "incomplete work" reports with specific action recommendations.

**Tools needed:**
- `IdeaRegistryQuery` — pull stagnant ideas.
- `MarketDataCheck` — current price/valuation vs. idea inception.
- `ConversationHistory` — when was this last discussed?
- `DistractionAnalysis` — what else was the team focused on during the gap?

---

### 11. Portfolio Construction Awareness

The Raw words document introduces a requirement that goes beyond individual stock analysis: the system must support deliberate portfolio construction, ensuring the portfolio is "benchmark-agnostic but not benchmark-ignorant."

#### 11.1 What Needs to Be Built

| Capability | Description |
|:---|:---|
| **Benchmark Deviation Tracker** | Continuously track how the portfolio deviates from market indices. Ensure deviations are deliberate choices, not accidental drift. |
| **Sector/Theme Exposure Analysis** | Map portfolio holdings to sectors, themes, and macro factors. Identify concentration risks and unintended bets. |
| **Risk Factor Analysis Module** | (Future — "extra module" per Raw words) Evaluate how various assets combine and impact overall portfolio risk. Correlation analysis, factor exposure. |
| **Position Sizing Monitor** | Track whether executed positions reach their target sizes. Flag when partial fills (e.g., 1.8% vs. 5% target) leave the portfolio underweight to conviction. |
| **Deliberate vs. Accidental Deviation** | For each benchmark deviation, the system must be able to trace it back to a deliberate decision or flag it as accidental drift requiring attention. |

#### 11.2 Orchestrator Changes Needed

- New `PortfolioConstructionTool` and `ExposureAnalysisTool` registered in the ADK agent graph.
- Background process that runs daily to update deviation tracking.
- Results feed into the morning briefing and are available on-demand.

---

### 12. Front-End and Interaction Layer

The current front-end is a chat widget (`ChatWidget.vue`). Rezco needs significantly more.

#### 12.1 Required Interfaces

| Interface | Description |
|:---|:---|
| **Chat (exists)** | Prompted queries. Enhance with: persona selector, instrument auto-complete, source citations in responses, conviction indicators, signal/noise scoring on results. |
| **Morning Briefing Dashboard** | Daily view: overnight moves, alerts, flags, unresolved threads, proactive nudges, incomplete work alerts, benchmark gap flags. Not a chat — a structured dashboard. |
| **Instrument Deep-Dive** | Per-instrument page: latest view, conviction curve, headwind/tailwind assessment, financial data, value grid, model, linked research, upcoming events, idea-to-investment status, sentiment trajectory from earnings calls. |
| **Portfolio Overview** | Current holdings, relative weights vs. benchmark, contribution analysis, sector/theme exposure, deliberate vs. accidental deviations. Links to instrument deep-dives. |
| **Ideas Pipeline** | Kanban-style view of all ideas: low conviction → medium → high → executed. Filter by sector, theme, PM. Click to see full context and linked market data. |
| **Report Viewer** | Rendered investment case reports. Exportable to PDF for client meetings. |
| **Meeting Prep View** | Before a management meeting: generated questions, gaps, contradictions, current investment case, linguistic cue history. |
| **Knowledge Graph Explorer** | Visual spiderweb view: see how instruments, themes, and data points connect. Click to drill down. |
| **Persona Comparison View** | Side-by-side: "Rezco view vs. Buffett view vs. Druckenmiller view" on the same instrument. |
| **Signal Feed** | Real-time feed of high-scoring signals, filterable by instrument, sector, theme. PMs can promote/demote signal scores. |

#### 12.2 Orchestrator Changes Needed

- The A2A streaming protocol currently returns text parts. Needs to support **structured response types**: tables, charts, dashboards, instrument cards.
- The `StreamResponse` message types (Msg, Task, StatusUpdate, ArtifactUpdate) need extension for rich content: embedded charts, data tables, links to instrument entities.
- New API endpoints beyond the A2A chat path for dashboard data, instrument lookups, portfolio views, idea pipeline, and signal feed.

---

### 13. Scheduled and Background Execution

The current orchestrator is purely reactive — it responds to user messages. Rezco needs proactive, scheduled capabilities.

#### 13.1 What Needs to Be Built

| Capability | Schedule | Description |
|:---|:---|:---|
| **Morning Briefing** | Daily, pre-market (06:00 SAST) | Compile overnight developments, portfolio alerts, unresolved threads, incomplete work, benchmark gaps, high signals |
| **Holdings Sync** | Daily, post-market close | Pull latest holdings and weights from portfolio management system |
| **Financial Data Refresh** | Weekly (minimum) | Update PE ratios, earnings, prices across the instrument universe |
| **Google Chat Processing** | Continuous / near-real-time | Ingest and file new messages as they arrive |
| **Google Drive Monitoring** | Continuous / near-real-time | Detect and ingest newly uploaded PDFs, transcripts, presentations |
| **Earnings Revision Monitor** | Daily | Scan for positive/negative earnings revisions across covered universe |
| **Contradiction Scan** | Weekly | Cross-reference current portfolio positioning against stated views |
| **Stale View Detection** | Weekly | Flag instruments where the last PM view is older than X months but material events have occurred |
| **Incomplete Work Scan** | Daily | Scan idea registry for stagnant ideas where market data supports the original thesis |
| **Headwind/Tailwind Refresh** | Weekly | Update environment classifications based on latest earnings calls and data |
| **Benchmark Deviation Check** | Daily | Compare portfolio to benchmark, flag unintentional drift |
| **24/7 Market Screening** | Continuous | AI personas screen global markets against their specific philosophies, flag opportunities per Rezco style |

#### 13.2 Orchestrator Changes Needed

- A **task scheduler** within the orchestrator — the current architecture only processes tasks triggered by A2A messages.
- Scheduled tasks must be able to write results to the conversations service (so they appear in the chat/dashboard) and to the knowledge graph (so they're retrievable).
- The `agentExecutor.Execute` function needs to support system-initiated runs, not just user-initiated ones. This means sessions created by the system (e.g., `UserID = "system"`) with appropriate metadata.
- A cron-like configuration surface (which tasks run when, with what parameters).

---

### 14. Scope Boundaries

Rezco was explicit about what is in scope now vs. later.

#### 14.1 Phase 1 (Now): South African Equities

- Full instrument-level research coverage for SA equities.
- Individual company models, value grids, management meetings.
- The ~3,000 instruments in Madeleine's classification system.
- Google Chat integration, meeting transcription, prompted queries, morning briefing.
- Conviction workflow (ideas to investments pipeline).
- Headwind/tailwind analysis on SA companies.
- Signal amplification and noise filtering.

#### 14.2 Phase 2 (Later): Global Allocation

- Regional/thematic level (not individual stock): "European industrials ETF", "China allocation".
- Macro research feeding allocation decisions: how much offshore vs. local? Which regions?
- Gundlach/Druckenmiller/Dalio macro personas become more relevant here.
- Macro indicator monitoring (PMIs, interest rates, economic data by country).

#### 14.3 Phase 3 (Way Down the Line): Performance Attribution and Risk

- Relative contribution analysis (which positions drove fund vs. benchmark performance).
- Attribution reporting for client communications.
- Risk factor analysis module (how assets combine to impact portfolio risk).
- Described as "three years time" territory.

---

### 15. Integration Architecture Summary

```
                                    ┌──────────────────────────────┐
                                    │        Front-End Layer        │
                                    │  Chat │ Dashboard │ Deep Dive │
                                    │  Ideas Pipeline │ Reports    │
                                    │  Signal Feed │ Graph Explorer │
                                    └──────────────┬───────────────┘
                                                   │
                                    ┌──────────────▼───────────────┐
                                    │       Console Gateway         │
                                    │   (A2A + Auth + Persist)      │
                                    └──────────────┬───────────────┘
                                                   │
                    ┌──────────────────────────────▼────────────────────────────────┐
                    │                        ORCHESTRATOR                            │
                    │                                                                │
                    │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌────────────┐  │
                    │  │ Research  │  │ Compare   │  │ Report    │  │ Meeting    │  │
                    │  │ Agent     │  │ Agent     │  │ Agent     │  │ Prep Agent │  │
                    │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬──────┘  │
                    │        │              │              │              │           │
                    │  ┌─────┴─────┐  ┌─────┴─────┐  ┌────┴──────┐  ┌───┴────────┐ │
                    │  │ Morning   │  │ Screening │  │ Model     │  │ Incomplete │ │
                    │  │ Briefing  │  │ Agent     │  │ Rolling   │  │ Work       │ │
                    │  │ Agent     │  │           │  │ Agent     │  │ Monitor    │ │
                    │  └─────┬─────┘  └─────┬─────┘  └────┬──────┘  └───┬────────┘ │
                    │        │              │              │              │           │
                    │  ┌─────┴─────┐  ┌─────┴─────┐  ┌────┴──────┐  ┌───┴────────┐ │
                    │  │ Persona   │  │ Signal    │  │ Bias      │  │ Ingestor   │ │
                    │  │ Engine    │  │ Scoring   │  │ Detection │  │ Agents     │ │
                    │  │           │  │ Engine    │  │ Agent     │  │            │ │
                    │  └───────────┘  └───────────┘  └───────────┘  └────────────┘ │
                    │                                                                │
                    │  ┌─────────────────────────────────────────────────────────┐   │
                    │  │              Headwind/Tailwind + Sentiment              │   │
                    │  │              Conviction + Contradiction                 │   │
                    │  │                  (Cross-Cutting Layer)                  │   │
                    │  └─────────────────────────────────────────────────────────┘   │
                    │                                                                │
                    └────────┬───────────────┬───────────────┬──────────────────────┘
                             │               │               │
               ┌─────────────▼──┐  ┌─────────▼────────┐  ┌──▼──────────────┐
               │   Knowledge    │  │   Instrument     │  │    Sessions     │
               │    Graph       │  │   Master +       │  │    + Events     │
               │  (Spiderweb)   │  │   Financial Data │  │    (Spanner)    │
               │                │  │   + Holdings     │  │                 │
               └────────┬───────┘  └─────────┬────────┘  └─────────────────┘
                        │                    │
                ┌───────┴────────┐   ┌───────┴────────┐
                │ Idea Registry  │   │ Conviction     │
                │ + Execution    │   │ + Bias         │
                │ Gap Tracking   │   │ Tracking       │
                └────────┬───────┘   └───────┬────────┘
                         │                   │
           ┌─────────────┴───────────────────┴──────────────┐
           │              External Integrations              │
           │                                                 │
           │  Google Chat │ Google Meet │ Google Drive       │
           │  Bloomberg │ Reuters │ YouTube │ News Feeds     │
           │  Portfolio Mgmt System │ CSV Uploads            │
           │  Sell-Side Email │ Company Websites              │
           └─────────────────────────────────────────────────┘
```

---

### 16. Key Architectural Principles (From Rezco)

These were stated directly by the team across all sessions and must guide all build decisions:

1. **Architecture at inception matters.** The biggest cost is PM time training the AI. Getting the foundation wrong means redoing that investment. The core architecture must be "really, really clean."
2. **Flexible foundation.** Must be able to add capabilities (global allocation, performance attribution, risk factor analysis) without changing the underlying architecture.
3. **Not a black box.** PMs must understand why the AI is saying what it's saying. Sources must be cited. Reasoning must be transparent.
4. **The PM is the bottleneck — reduce friction, don't add it.** Every feature must save PM time, not create new workflows to manage.
5. **Continuous, not episodic.** Portfolio management is ongoing analysis. The system must maintain state across months and years, not reset per conversation.
6. **Proprietary vs. public must always be distinguishable.** Compliance requirement. The AI must always know and disclose the source type.
7. **Analyst, not portfolio manager.** The AI provides information and analysis. It does not make buy/sell recommendations or generate alpha claims at this stage.
8. **Buy-side is different from sell-side.** Analysis is always relative and cross-sector, never siloed. A bank competes with a retailer for capital allocation.
9. **Benchmark-agnostic but not benchmark-ignorant.** The portfolio doesn't track an index, but must always know how it deviates from one and why.
10. **Signal over noise.** The system's purpose is not to process more data, but to amplify faint signals that drive investment success while filtering the noise that sell-side and media generate.
11. **Ideas must not die in the archive.** Every debated idea must be actively monitored and resurfaced when market conditions align with the original thesis.
12. **Cognitive diversity by design.** Multiple AI personas with genuinely different philosophies prevent groupthink and ensure the team sees every angle.

---

### 17. Gap Summary: Current Orchestrator vs. Rezco Requirements

Updated to reflect existing infrastructure discovered in the Madelein classification tool meeting.

| Capability | Current State | Required State | Gap Size |
|:---|:---|:---|:---|
| Chat-based Q&A | Exists (A2A + streaming) | Enhance with instrument resolution, source citations, persona selection, signal scoring | Medium |
| Data ingestion pipeline | **Partial** — research hub has document store (Vertex AI), manual upload/categorize workflow, Morningstar/Data Stream/Visible Alpha daily syncs | Automate curation, add Google Chat/Meet/YouTube ingestion, connect to orchestrator knowledge graph | **Medium** (was Large) |
| Knowledge graph (Spiderweb) | **Partial** — Spanner vector DB (similarity search), Vertex AI doc store (RAG), research DB filterable by classification | Full graph DB with non-linear linking, temporal indexing, entity resolution, trust weighting, hierarchical mapping | **Medium-Large** (was Large) |
| Instrument master data | **Exists** — FIGI-based instrument DB with Morningstar/Bloomberg classifications, custom fields, data source hierarchy, investable groups, ~3,000 shares | Add ISIN secondary key, connect to orchestrator, automate classification maintenance, add value grids, model store, benchmark comparison engine | **Small-Medium** (was Medium) |
| Structured financial data | **Exists** — earnings estimates (EPS, EBIT, net income) from Visible Alpha + internal, balance sheets, cash flow statements | Add PE ratios, share prices, earnings revisions tracking, free cash flow. Connect to orchestrator grounding tools | **Small** (was part of Medium) |
| Research & recommendations | **Exists** — buy/sell recommendation workflow, supporting documents, research database, RAG-enabled LLM agent | Enhance with conviction tracking, headwind/tailwind assessment, temporal awareness, non-linear linking | **Medium** (was Large) |
| Classification automation | **Partial** — vector DB suggests values, but requires manual acceptance | Agent to auto-accept above threshold, daily scan for gaps, custom field maintenance agent | **Small** |
| Analyst personas | Does not exist | 7 parameterized agent configs with distinct knowledge bases, reasoning styles, configurable engagement | Medium |
| Headwind/tailwind analysis | Does not exist | Sentiment engine, linguistic cue detection, longitudinal earnings call analysis, environment classifier | Large |
| Conviction workflow (ideas → investments) | Does not exist | Idea registry, post-debate monitoring, conviction progression, execution gap detection | Large |
| Signal amplification / noise filtering | Does not exist | Signal scoring engine, manual amplification, bias detection, contextual opportunity triggers | Large |
| Report generation | **Partial** — analysts produce recommendation reports with supporting docs | Automate with charts, data, conviction history, H/T assessment, sentiment trajectory | **Medium** (was Medium) |
| Morning briefing (unprompted) | Does not exist | Scheduled daily execution with portfolio monitoring, incomplete work, benchmark gaps, signal highlights | Large |
| Management meeting prep | Does not exist | Gap analysis, contradiction detection, question generation, linguistic cue history | Medium |
| Contradiction/temporal intelligence | Does not exist | Cross-temporal view tracking, conviction curves, stale view detection | Large |
| Source weighting/trust layer | **Partial** — data source hierarchy exists for instrument fields | Extend to all knowledge graph nodes, add trust scoring, PM conviction weighting | **Small-Medium** (was Medium) |
| Portfolio construction awareness | Does not exist | Benchmark deviation tracking, exposure analysis, position sizing monitor, deliberate vs. accidental | Medium |
| Scheduled/background execution | Does not exist | Cron-style task scheduler with 12+ scheduled tasks within orchestrator | Medium |
| Front-end (beyond chat) | **Partial** — two consoles exist (data/reporting + investment team), Vue 3, gRPC web | Add 10 new interfaces: dashboard, deep-dive, ideas pipeline, portfolio view, signal feed, graph explorer. Modernize existing consoles (IAM, MCP, BFF) | **Medium-Large** (was Large) |
| Model rolling | Does not exist | Update existing financial models with latest quarterly results | Small |
| Quantitative screening | Does not exist | Filter instrument universe by style criteria, earnings revisions, H/T classification, benchmark gaps | Small |

---

### 18. Recommended Build Order

Updated to reflect the existing research hub infrastructure. Sprints are reordered to build on what exists rather than starting from scratch.

**Pre-Sprint: Discovery and Access (1 week)**
- Simon screen-shares and walks Techbridge through the research hub to identify what is actually used vs. unused (per Madelein's recommendation)
- Techbridge gets access to the research hub (both consoles)
- Inventory existing APIs, data pipelines, and Vertex AI/Spanner infrastructure
- Identify which legacy code (auth, IAM) must be modernized vs. can be left as-is
- Map existing data source hierarchy to the planned trust/weighting layer

**Sprint 1-2: Foundation (Integrate with Existing)**
- Connect orchestrator to existing instrument database (FIGI registry, classifications, Morningstar/Data Stream/Visible Alpha pipelines)
- Connect orchestrator to existing Vertex AI document store and Spanner vector DB
- Knowledge Graph schema and service — overlay on existing research DB, adding non-linear linking and temporal indexing
- Extend existing RAG pipeline with graph traversal capabilities
- Idea Registry schema and service

**Sprint 3-4: Core Analyst + Automation**
- Research & Retrieval Agent (prompted queries via enhanced knowledge graph + existing document store)
- Google Chat Listener (begin continuous ingestion — fills the biggest gap)
- Meeting Transcription Pipeline (process existing recordings)
- Entity resolution (company name → FIGI/ISIN mapping, leveraging existing instrument DB)
- **Classification Automation Agent** — automate the manual acceptance of vector-suggested values, daily gap scanning
- **Custom Field Maintenance Agent** — automate custom classification updates for reporting
- Basic signal scoring on ingested content

**Sprint 5-6: Intelligence Layer**
- Temporal indexing and conviction tracking
- Contradiction detection engine
- Source weighting and trust layer (extend existing data source hierarchy to all knowledge graph nodes)
- Comparison Agent (with GARP methodology)
- Headwind/tailwind classification engine

**Sprint 7-8: Proactive Capabilities**
- Morning Briefing Agent (scheduled execution)
- Holdings Sync (daily from portfolio management system)
- Financial Data Store — add PE ratios, share prices, earnings revisions to existing earnings estimate infrastructure
- Threshold alerts and monitoring
- Execution Gap Detection (incomplete work monitor)
- Benchmark deviation tracking

**Sprint 9-10: Conviction and Signal Layer**
- Conviction progression workflow (ideas → investments pipeline)
- Signal amplification engine with manual override
- Cognitive bias detection (anchoring, confirmation, loss aversion)
- Longitudinal sentiment analysis on earnings calls
- Linguistic cue detection
- **Research curation agent** — monitor trusted news sources, auto-add to research hub (replacing manual analyst effort)

**Sprint 11-12: Advanced Features**
- Analyst Personas (Buffett, Dalio, Druckenmiller, Gundlach, Robin Hood, Technical)
- Report Generation Agent (enhance existing recommendation workflow with H/T assessment and sentiment trajectory)
- Management Meeting Prep Agent
- Model Rolling Agent
- Quantitative Screening Agent (with H/T filter and benchmark gaps)
- **Buy/sell recommendation automation** — agent generates recommendation reports with hypotheses, leveraging existing recommendation schema

**Sprint 13-14: Front-End Expansion**
- Morning briefing dashboard
- Instrument deep-dive pages (with conviction curves and H/T classification)
- Ideas pipeline (kanban view)
- Portfolio overview (with deliberate vs. accidental deviation)
- Signal feed
- Knowledge graph explorer
- Persona comparison view
- Modernize existing consoles where needed (IAM, MCP servers, BFF REST endpoints)

**Ongoing: Phase 2+**
- Global allocation support (macro personas active)
- 24/7 autonomous market screening
- Real-time synchronous persona participation in live debates
- Performance attribution (Phase 3)
- Risk factor analysis module (Phase 3)
- Reduce analyst headcount as agents take over curation, summarization, and research tasks
