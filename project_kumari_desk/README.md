# Kumari Catering Desk

An AI front desk for Kumari Catering in Maharagama. It classifies customer messages, answers questions using seven business documents with citations, drafts complaint replies, records API usage, and falls back to Python rules when API requests fail.

Model: openai/gpt-oss-20b via Groq. Why: low token cost and support for strict JSON outputs. Fallback: rules_triage() using pure Python.

## Setup

Developed and run using Python 3.11.16 in a Windows Conda environment.

### 1. Install dependencies

From the `project_kumari_desk` folder, run these commands in an Anaconda Prompt:

```shell
conda create -n kumari_desk python=3.11
conda activate kumari_desk
python -m pip install -r requirements.txt
```

### 2. Configure the API key

Copy `.env.example` to `.env` and enter your Groq API key locally:

```dotenv
GROQ_API_KEY=your_key_here
```

The `.env` file is ignored by Git. Never include an actual API key in notebook cells, outputs, or commits.

### 3. Run the notebook

1. Open `project_01_family_business_desk.ipynb` in VS Code.
2. Install the Microsoft Python and Jupyter extensions if needed.
3. Select the `kumari_desk` Python environment as the notebook kernel.
4. Run the setup cell and subsequent cells in order.
5. Run the demonstration and official SUBMIT cell last.
6. Save the notebook with its execution outputs.

Internet access is required for package installation, the initial MiniLM download, and Groq requests. After downloading the model, embedding generation runs locally on the CPU.

## Project Files

```text
project_kumari_desk/
├── business_docs/
│   ├── complaints.md
│   ├── delivery.md
│   ├── equipment_rental.md
│   ├── faq.md
│   ├── menu_prices.md
│   ├── ordering.md
│   └── payments_cancellations.md
├── project_01_family_business_desk.ipynb
├── README.md
├── requirements.txt
├── .env.example
├── .gitignore
├── project_results.json
├── demo_transcript.md
└── usage_receipts.json
```

Local files and folders excluded from Git:

- `.env`: API credentials.
- `.venv/`: local Python virtual environment.
- `chroma_db/`: generated persistent database.
- Python caches and notebook checkpoints.

The indexing cells regenerate the database from `business_docs/`.

## Knowledge Base

Every business answer must be supported by the supplied documents.

| Document type | Files |
|---|---|
| `catalog` | `menu_prices.md`, `equipment_rental.md` |
| `policy` | `ordering.md`, `delivery.md`, `payments_cancellations.md`, `complaints.md` |
| `general` | `faq.md` |

Each chunk retains its source filename, document type, and word positions.

## Implementation

### Message classification

`rules_triage(message)` uses pure-Python keyword rules without an API.

`classify_message(message)` uses a six-part prompt containing the role, task, context, rules, few-shot examples, and output requirements. Structured JSON output is parsed and validated.

Both classifiers return:

```python
{"category": "...", "urgency": "..."}
```

The notebook includes four customer test messages covering a question, an order request, a normal complaint, and an urgent complaint.

### Document storage

The seven documents are split into 40-word chunks with 15-word overlap. The original documents produce 28 chunks.

Chunks are embedded using `sentence-transformers/all-MiniLM-L6-v2`, producing normalized 384-dimensional vectors.

Chroma stores the text, embeddings, and metadata persistently in `chroma_db/`, using cosine distance.

Stable chunk IDs and upserts prevent duplicate records. Re-indexing also removes outdated chunk IDs.

### Hybrid retrieval

`retrieve(question, k=3, where=None)` combines:

- Semantic search using MiniLM embeddings.
- A custom keyword ranker based on distinct matching words.
- Reciprocal rank fusion with a smoothing constant of 60.

It returns `(score, row)` pairs in descending score order. Rows include `chunk`, `source`, and `doc_type`.

The same metadata filter is applied to both retrieval paths before fusion.

### Grounded answers

`ask_business(question)` returns:

```python
{"answer": "...", "sources": [...]}
```

Answers use retrieved business information and cite supporting filenames. Source names are checked against the retrieved documents. Missing inline citation markers are added from the validated source list.

When the documents cannot answer, the function returns:

```python
{
    "answer": "That is not covered in the handbook.",
    "sources": []
}
```

### Routing and complaints

`handle(message)` returns exactly:

```python
{"route": "...", "output": "..."}
```

Public routes are `question`, `complaint`, and `other`.

- Questions receive grounded business answers.
- Complaints receive warm drafts based on `complaints.md`.
- Order requests receive ordering guidance under the `other` route.
- API and response-validation failures trigger `rules_triage()` and a fallback reply directing the customer to contact Kumari.

Complaint drafting uses supplied delivery timing, keeps optional photos separate from eligibility, and follows the documented reporting window and remedies.

The desk does not process orders, refunds, replacements, or bookings.

## Usage and Cost Estimates

Request receipts record provider-reported input and output token counts. `USAGE_LOG` exposes the required `in` and `out` fields, while the detailed log also records operation, model, status, and simulation labels.

`cost_cents()` estimates the cost of known, real usage in US cents. Simulated requests are excluded, and unavailable token usage is reported separately.

The recorded pricing table for `openai/gpt-oss-20b` is:

| Token type | USD per million tokens |
|---|---:|
| Input | 0.075 |
| Output | 0.30 |

Pricing recorded on 9 November 2026 from [Groq's model documentation](https://console.groq.com/docs/models).

These are estimates at the recorded rates, rather than account invoices.

Run:

```python
usage_report = save_usage_report()
```

after additional model requests to refresh `usage_receipts.json`.

## Verification

All eight official acceptance checks passed in `project_01_family_business_desk.ipynb`.

The eight official grading cells and official SUBMIT cell were retained unchanged. The setup was adapted to use Groq.

| Official check | Score |
|---|---:|
| Knowledge base and document labels | 15/15 |
| Pure-Python rules triage | 10/10 |
| AI triage and test messages | 15/15 |
| Persistent labelled Chroma store | 15/15 |
| Hybrid retrieval and filtering | 15/15 |
| Grounded answers and refusal | 10/10 |
| Router and fallback | 10/10 |
| Usage receipts and cost estimate | 10/10 |
| **Total** | **100/100** |

The official SUBMIT cell generated `project_results.json`.

Additional verification includes:

- Chunk coverage, unique IDs, and metadata checks.
- Embedding dimensions, finite values, and normalization.
- Persistence verified by a separate Python process.
- Seven filter scenarios across semantic, keyword, and hybrid retrieval.
- Unsupported questions and empty-result refusal.
- Four simulated outages covering classification, business answering, complaint drafting, and ordering guidance.
- Cost arithmetic and unknown-usage handling.

## Failure Log

### 1. Test expectation disagreed with the urgency rules

- **Observed:** The salty-food complaint returned normal urgency, while my test expected high.
- **Cause:** I had treated a complaint as automatically urgent, although the prompt distinguishes category from urgency.
- **Fix:** Corrected the expected urgency to normal and added a separate explicitly urgent complaint.
- **Verification:** Reran all four classification tests. All four matched their expected category and urgency.

### 2. Missing inline citation caused answer validation to fail

- **Observed:** The model returned the correct price and listed `menu_prices.md` in sources, but omitted the inline citation.
- **Cause:** The implementation required the source filename in both the sources list and the answer text.
- **Fix:** Added Python formatting that attaches missing citations from validated sources. Conflicting citations are still rejected.
- **Verification:** The chicken lunch pack answer returned Rs. 650 with `[menu_prices.md]` successfully.

### 3. Complaint draft ignored timing already supplied

- **Observed:** The customer said delivery occurred 30 minutes ago, but the draft asked for the delivery time again and delayed offering a remedy.
- **Cause:** The prompt did not explicitly require using supplied relative timing or distinguish optional photos from eligibility.
- **Fix:** Clarified how to handle known, unknown, and late reporting times, and that photos are optional.
- **Verification:** The revised draft recognized reporting within two hours and offered the customer's choice of same-day replacement or full refund of affected packs.

### 4. Template setup expected a different API provider

- **Observed:** The original setup stopped because `OPENROUTER_API_KEY` was missing, while this project used Groq.
- **Cause:** The provided setup was configured for OpenRouter.
- **Fix:** Adapted the setup and request logging to use the existing `GROQ_API_KEY`, while preserving the official grading cells.
- **Verification:** Groq setup succeeded, and all eight official acceptance checks subsequently passed.

## Demo Transcript

### Question 1

**Customer:** What does a chicken lunch pack cost?

**Desk:** A chicken lunch pack costs Rs. 650. [menu_prices.md]

### Question 2

**Customer:** Which payment methods are accepted?

**Desk:** We accept payment by cash or bank transfer only – card payments are not accepted. [payments_cancellations.md]

### Question 3

**Customer:** How much is delivery within 5 km of Maharagama junction, and beyond 5 km within greater Colombo?

**Desk:** Delivery is free within 5 km of Maharagama junction. Beyond 5 km, delivery is a flat Rs. 1,000 covering Colombo and the suburbs. [delivery.md]

### Question 4

**Customer:** When is a deposit required, and what percentage must I pay?

**Desk:** A 50% deposit is required at confirmation for orders above Rs. 30,000. The balance is due on delivery. [payments_cancellations.md]

### Question 5

**Customer:** How are poya-day orders delivered?

**Desk:** Poya-day orders are delivered the evening before the poya day. They are packed for overnight refrigeration and come with reheating instructions. [delivery.md] [ordering.md]

### Live Refusal

**Customer:** What is your bank account number?

**Desk:** That is not covered in the handbook.

## Limitations

- Keyword triage has limited language and intent understanding.
- Retrieval ranks available chunks; high ranking alone does not establish that a question is answerable.
- Valid citation filenames do not independently prove that every claim is supported. Answers also require comparison with the cited documents.
- The documents contain no live booking calendar or bank account details.
- The desk provides information and drafts rather than carrying out business transactions.