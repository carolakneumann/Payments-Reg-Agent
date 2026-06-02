# ✦ Payments Regulations Navigator - A RAG Agent Built in n8n
> The README is longer than most because the decisions are the portfolio, not just the output.
## Why I Built This
As a PM at Wells Fargo working on debit card and digital wallet products, I regularly need to answer regulatory questions before a PRD is written — what error resolution timeline applies here, does Reg II routing affect this tokenized transaction, does the CFPB Prepaid Rule change our disclosure requirements.
These are scoping questions. They shape the PRD, the edge cases you hand to engineering, and the UX decisions you make in week one. Getting them wrong means rework.
Regulation E, Electronic Fund Transfer Act (EFTA), the Nacha Operating Rules, the CFPB Prepaid Rule — none of these reference each other, none are easily searchable, and a single product question can touch four sources before you have a usable answer.
The documents exist. The knowledge exists. The problem is retrieval.
So I built a RAG agent in n8n that lets you ask natural language questions against a corpus of payments regulation documents and get cited, traceable answers pointing to the exact clause and source. The goal isn't to replace a regulatory review. It's to let a PM scope a feature without losing half a day to research.
This README is not a polished product announcement. It's a record of what I built, what broke, what I changed, and what I still don't trust the system to do.
### The Problem Was Harder Than I Expected
I went in thinking the hard part would be the AI. It wasn't.
The hard part was the documents.
Payment regulation PDFs are some of the worst-formatted documents on the planet. Regulatory bodies publish specs with three-column layouts, nested tables, footnotes that override the main body text, and version histories embedded in headers. OCR on older Federal Register annexes produces strings like "§ l005.1l(a)(l)" instead of "§ 1005.11(a)(1)." A naïve chunking strategy — 512 tokens with overlap, the standard tutorial answer — returned confident-sounding nonsense.

This matched exactly what practitioners building enterprise RAG systems describe: the document processing challenge is where most real-world systems fail, not the model selection. The lesson that hit hardest: document quality detection should be the first thing you build, not an afterthought. I wasted two weeks debugging retrieval quality before I realized the problem was upstream. Once I scored documents by extraction quality and routed them to different processing pipelines — clean PDFs got hierarchical chunking, messier docs got simpler fixed-chunk treatment with manual review flags — retrieval quality jumped more than any embedding model swap ever did.

#### Architecture Decisions (and the Ones I Reversed)
**What I originally built**
A linear n8n workflow: trigger → PDF ingestion → fixed-size chunks → embed → store in vector DB → query → respond. Clean. Fast to build. Embarrassingly bad at anything nuanced.

**What I changed and why**

Chunking strategy. Regulation documents have structure. A Regulation E error resolution section is different from its definitions section. Nacha's core operating rules are different from its appendices and risk management guidelines.  I rebuilt chunking to respect document hierarchy — section level, paragraph level, clause level — and added keyword triggers that shift retrieval granularity based on query type, If the question contains words like "business days," "provisional credit," or "§ 1005," the system drops to clause-level precision. Broad questions like "what are the Reg E error resolution requirements?" stay at section level.

Metadata over embeddings. This was the counterintuitive lesson. I spent time chasing better embedding models when the actual ROI was in metadata architecture. Payment regulations have highly specific attributes: Regulation name and part, CFR citation, effective date, amendment version, transaction type scope. Once I built domain-specific metadata schemas and used keyword matching for filter-first retrieval, precision improved significantly — more than switching from one embedding model to another ever did.

Hybrid retrieval. Pure semantic search fails more often than anyone admits, especially in specialized regulatory language. "ODFI" and "RDFI" are semantically similar in embedding space but represent opposite sides of an ACH transaction with different obligations under the Nacha rules. "Network" means something different in a Reg II routing context than in a Nacha context than in a card scheme context. I added a keyword layer and a simple document relationship graph to catch cross-references — Reg E and the Nacha rules both govern ACH error resolution and cite each other, and the complete answer to most disputes questions requires pulling from both.

What I Learned About n8n Specifically
n8n made the workflow layer genuinely fast to iterate. The visual graph is useful for spotting where context is getting lost between nodes, and the ability to mock individual nodes while keeping the rest of the chain live saved hours of debugging. That said, for complex branching logic around retrieval quality and fallback strategies, the visual representation becomes harder to reason about than code would be. I'd use n8n again for this type of agent, but I'd be honest with a team that it has a complexity ceiling.
The observability story also required work. n8n doesn't give you retrieval confidence scores or token traces out of the box. I added logging nodes at the retrieval and generation steps to capture what chunks were actually used in each answer — not because the outputs needed it, but because I needed to build trust in the system before I could trust the system.
 Implementation Guidance
1.	n8n Setup: Use the AI Agent node rather than the basic "Chains" node. It allows for "Tools" (like a calculator for $10$-day provisional credit windows).
2.	OpenAI: Use gpt-4o-mini for the Query Classifier (it’s fast and cheap) and gpt-4o for the final Synthesis to ensure the legal nuance is captured.
3.	Supabase: Enable the pgvector extension and use the Match Documents stored procedure to handle the hybrid search logic.

Limitations I'm Not Hiding

The system is only as current as its last ingestion run. Regulations update. The agent doesn't know what it doesn't know. If you ask it about a guideline that was revised after the last document load, it will answer confidently from the outdated version. This is not a solvable problem with prompting — it's a data pipeline problem, and it's the first thing I'd address before putting this in front of a compliance team.
Cross-jurisdictional queries degrade. Asking "how do Reg E dispute requirements differ from NACHA ACH rules for unauthorized transactions?" requires synthesizing across multiple regulatory frameworks simultaneously. The agent handles this inconsistently. Sometimes the retrieval lands well. Sometimes it over-indexes on one jurisdiction and misses the other. I haven't solved this and I say so clearly in the interface.
Tables are still partially broken. Regulatory documents embed critical information in tables — exemption thresholds, velocity limits, transaction type matrices. My table extraction pipeline converts simple tables to structured metadata well enough, but complex nested tables with merged cells still produce degraded chunks. The most consequential regulatory thresholds often live in exactly those tables.
I don't trust it for legal decisions. The agent is a research accelerator, not a compliance officer. Every answer surfaces its source documents and chunk references for a reason — so a human can verify. I built that into the UX intentionally, not as a disclaimer, but because the product wouldn't be honest without it.

What I'd Do Differently
If I were starting over, I'd spend the first sprint on nothing but document quality assessment before writing a single line of retrieval logic. I'd also build the evaluation harness — a set of known questions with verified answers — before building the agent itself. Testing against real questions revealed failure modes that no amount of vibes-checking the demo ever would have.
I'd also be more skeptical of my own retrieval quality metrics early on. Watching an agent answer test questions correctly is not the same as the agent being reliable. The questions that really test a RAG system are the ones you didn't think to ask.

Stack
•	Orchestration: n8n Cloud
•	Vector DB: Supabase pgvector
•	Embeddings: Open AI text-embedding-3-small
•	LLM: OpenAI GPT-4o for query analysis and answer generation
•	Document processing: PyMuPDF for clean PDFs, Tesseract for OCR fallback
•	Chunking: Custom hierarchical parser — no off-the-shelf chunking library handled regulatory document structure well enough
•	Document source: Google Drive /regulations 
•	Output: Google Drive · cited answers saved per query

The Honest Version of the Outcome

The agent works. It reduces the time to find a specific regulatory clause from 20-40 minutes of document hunting to under two minutes for well-formatted source documents. It surfaces cross-references a human researcher would likely miss. It produces answers that are cited and verifiable.
It also fails on complex cross-jurisdictional queries, gets confused by outdated documents in the corpus, and handles poorly-formatted annexes inconsistently.
That gap — between what it does well and what it still can't do reliably — is where the interesting product work lives.

<img width="946" height="759" alt="image" src="https://github.com/user-attachments/assets/b7f3f3e3-fbff-49b6-a682-75c46536b6c7" />

