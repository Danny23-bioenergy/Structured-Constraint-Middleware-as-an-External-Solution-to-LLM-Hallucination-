Structured Constraint Middleware (SCM)
A Lightweight External Solution to LLM Hallucination

Author: Danny Nawa Muyunda
Affiliation: Copperbelt University, Zambia
DOI (v2): 10.5281/zenodo.24422271 | DOI (v1): 10.5281/zenodo.24301140
License: Open Source

What is SCM?

Large Language Models (LLMs) generate responses through probabilistic next-token prediction — not verified fact retrieval. This makes them prone to hallucination: producing confident, fluent, but factually incorrect outputs.

Structured Constraint Middleware (SCM) is a lightweight, model-agnostic external wrapper that sits between an LLM and the user. It audits every response before it is delivered — evaluating confidence and citation quality — and either passes or blocks the answer. No model retraining required. No internal model access needed.

How It Works
User Question → Domain Detection → Constrained LLM Call → Confidence + Citation Check → Answer or Abstain
The question is classified by domain (medicine, law, finance, bioenergy, agriculture, general)
The model is prompted to answer only from established literature and return a structured response with a confidence score and source
The middleware evaluates whether confidence meets the threshold (≥75) AND a valid citation exists
If both conditions are met — the answer is delivered. Otherwise — the tool abstains and logs the reason
Versions
SCM v0 — Initial Prototype

The first prototype established the basic pipeline but had a critical flaw: the abstained column defaulted to False for every response. Abstention was attempted by scraping "i don't know" from the raw answer text which never triggered reliably. The model answered everything regardless of confidence or citation quality — no real gating existed.

SCM v1 — Regex Gate

v1 introduced a proper two-gate decision function — confidence threshold + citation check — making abstention an actual block on the response rather than a log annotation.

How it works:

Epistemic boundary prompting — model must append SOURCE: and CONFIDENCE: XX/100 to every response
Regex extraction of confidence score and citation
Abstains if confidence < 75 OR no valid citation found

v1 Benchmark Results (Full 100-Question Run — TruthfulQA + HaluEval):

Metric	Result
Total questions	100
Answered (passed gate)	65 / 100
Abstained — low confidence	17
Abstained — low confidence and no citation	17
Abstained — confident but no citation	1
Flagged (confidence < 70)	30 / 100
Avg confidence (answered only)	90.8 / 100
API errors	0

v1 Weaknesses Discovered:

Regex fragility — bold markdown formatting (e.g. **CONFIDENCE:** 85/100) caused the parser to miss scores, defaulting to 0 and blocking accurate answers
Overconfident citation fabrication — the model could bypass the gate by inventing plausible-sounding citations with high self-reported confidence
SCM v2 — JSON Schema + Conditional Web Search

v2 was built to close both weaknesses from v1 while preserving the tool's lightweight, model-agnostic design.

Key upgrades:

Strict JSON Schema enforcement — replaces regex with API-level json_schema enforcement ({"answer", "source", "confidence"}). Markdown formatting can no longer corrupt parsing
Conditional web search — if the model is confident (≥75) but has no valid citation, the middleware triggers a DuckDuckGo search (max_results=2) and re-asks with real web context
Preserved diagnostic logging — evaluates raw model knowledge before retrieval to expose the boundary between internal training and external search

v2 Benchmark Results:

Metric	Result
Questions completed before token quota	23 / 100
Real answers passed	9 / 23
Precision on passed answers	100% (9/9)
Hallucination pass-through	0%
Token quota errors logged (HTTP 429)	77 / 100
v1 markdown false abstentions	Eliminated
v1 citation bluffs	Caught and abstained

Token Quota Note:
v2 uses up to 3 API calls per question (domain detection, raw answer, conditional web search) versus v1's 2 calls. Under Groq's free tier (200,000 tokens/day), the pipeline exhausted the daily quota at question 24. This is a documented tradeoff — v2 is more accurate but more token-intensive. v1 remains the practical choice for constrained environments.

Performance Summary
Feature / Metric	Baseline	SCM v1	SCM v2
Output Parsing	Unstructured	Regex	JSON Schema
External Retrieval	None	None	Conditional (DuckDuckGo)
Silent Blank Responses	36 / 100	0%	0%
Markdown Parsing Bugs	N/A	Yes	Eliminated
Overconfident Bluffs	Frequent	Rare	0% (Rows 1-23)
Total Abstained	0%	35%	14% real + 77% token errors
Precision on Passed Outputs	~75%	~90%	100% (9/9)
Token Calls per Question	1	2	Up to 3
Full 100-Q Completion	Yes	Yes	No (quota at Q24)
Files
File	Description
middleware.py	Main middleware engine — run directly or import into your project
baseline_test.py	Runs 100 benchmark questions through raw AI with no middleware
official_nightmare.py	Runs 100 benchmark questions through the full middleware pipeline
Quick Start

1. Install dependencies:

bash
pip install groq datasets duckduckgo-search

2. Set your API key and model in middleware.py and baseline_test.py:

python
API_KEY = "paste_your_groq_key_here"
MODEL = "paste_your_model_here"  # e.g. llama-3.1-8b-instant

3. Run interactively:

bash
python middleware.py

4. Run benchmarks:

bash
python baseline_test.py
python official_nightmare.py
Switching API Providers

The tool works with any OpenAI-compatible API. Just swap the client import:

python
# Groq (default)
from groq import Groq
client = Groq(api_key=API_KEY)

# OpenAI
from openai import OpenAI
client = OpenAI(api_key=API_KEY)

# Together AI
from openai import OpenAI
client = OpenAI(api_key=API_KEY, base_url="https://api.together.xyz/v1")

Everything else stays the same.

Research Paper

Full methodology, benchmark analysis and version comparison:
📄 Zenodo v2 — DOI: 10.5281/zenodo.24422271
📄 Zenodo v1 — DOI: 10.5281/zenodo.24301140

Why This Exists

This tool was built independently by a final-year Bioenergy Engineering student at Copperbelt University, Zambia — using only Groq's free inference tier and a total compute budget under K50. The goal is simple: make AI outputs safer and more honest, especially in high-stakes domains like medicine and engineering, and keep the solution accessible to everyone regardless of resources.

Built in Zambia. Open to the world.

