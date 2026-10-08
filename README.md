# Intelligent Voucher Classification Using Open-Source LLMs

An AI-powered system that reads structured financial transaction data and predicts the correct accounting voucher type for each row.

It combines deterministic accounting rules, Retrieval-Augmented Generation (RAG) and a locally hosted open-source LLM, so both clear-cut and ambiguous transactions are handled.

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [How It Works](#how-it-works)
- [Objectives](#objectives)
- [Target Users](#target-users)
- [Open-Source AI](#open-source-ai)
- [Why This Technology](#why-this-technology)
- [System Architecture](#system-architecture)
- [Workflow](#workflow)
- [Data Flow and Messy Data](#data-flow-and-messy-data)
- [Technology Stack](#technology-stack)
- [Features](#features)
- [Input and Output](#input-and-output)
- [Evaluation](#evaluation)
- [Challenges and Mitigation](#challenges-and-mitigation)
- [Future Scope](#future-scope)

---

## Problem Statement

Accounting systems hold many kinds of transactions, and each one belongs to a different voucher category. The provided dataset has structured fields such as:

- Supplier and customer details
- Invoice number and date
- Item descriptions and quantities
- Taxable value, GST, discounts and freight
- Payment, debit and credit information
- Return information
- Purchase and sales order references
- Inventory movement information
- Payroll information
- Import/export information

The **voucher-type field is intentionally missing**. The goal is to work out the most suitable voucher category for every transaction from the remaining fields.

### Target voucher categories (27)

| | | |
|---|---|---|
| Purchase | Sales | Purchase Return / Debit Note |
| Sales Return / Credit Note | Payment | Receipt |
| Contra | Journal | Salary / Payroll |
| Attendance | Purchase Order | Sales Order |
| Receipt Note | Delivery Note | Rejection In |
| Rejection Out | Stock Journal | Physical Stock |
| Material In | Material Out | Job Work In Order |
| Job Work Out Order | Import | Export |
| Expense | Advance / Prepayment | Other / Miscellaneous |

### Core challenge

Simple keyword matching isn't reliable here, because several categories have overlapping characteristics:

- Purchase vs Sales
- Purchase Return vs Sales Return
- Payment vs Receipt
- Contra vs Payment/Receipt
- Purchase vs Stock Journal

---

## How It Works

The project takes a **rule-first hybrid approach**. Instead of sending every transaction to an LLM, it first handles the clear cases with accounting rules and passes only the ambiguous ones to retrieval and the model.

```
Excel Transaction Dataset
          ↓
   Data Preprocessing
          ↓
      Rule Engine
       /       \
 Clear Case   Ambiguous Case
      ↓            ↓
   Direct      RAG Retrieval
Classification      ↓
      |       Chroma Vector Store
      |             ↓
      |     Qwen3.8 (via LM Studio)
      |             |
      +-------------+
             ↓
      Confidence Check
             ↓
   Structured JSON Output
```

1. **Structured input.** An Excel file of transactions without voucher-type labels.
2. **Preprocessing.** Fields are cleaned and normalized.
3. **Rule-based classification.** Predefined accounting rules settle clear cases, such as payroll fields pointing to Salary / Payroll.
4. **Retrieval.** For ambiguous cases, similar labeled examples are fetched from ChromaDB.
5. **LLM reasoning.** The transaction and examples go to Qwen3.8, running locally through LM Studio.
6. **Confidence validation.** The prediction is checked before it is accepted.
7. **Structured output.** A JSON result with the identifier, voucher type, confidence and a short reason.

### Why a hybrid design?

A purely rule-based system struggles when information is incomplete, several categories share fields, signals conflict, or new patterns appear. A purely LLM-based system can be inconsistent, spends expensive reasoning on easy rows, and is harder to evaluate. Combining the two gets the strengths of each.

---

## Objectives

- **Automated classification** of every structured transaction into a voucher category.
- **Multi-field reasoning** across several fields instead of single keywords.
- **27-class support** for all categories in the challenge.
- **Better handling of similar categories** (the pairs listed above).
- **Open-source AI** with local inference rather than a proprietary API as the main engine.
- **Hybrid intelligence** that mixes explicit rules with retrieval-assisted reasoning.
- **Structured output** as machine-readable JSON.
- **Evaluation** on unseen records using accuracy, precision, recall and F1.

---

## Target Users

| User | Benefit |
|---|---|
| Accountants | Less repetitive manual classification and fewer misclassifications |
| CA firms | More consistent processing of large client datasets |
| Small traders and businesses | Classify monthly records without heavy accounting automation |
| ERP / accounting software vendors | Possible integration into existing workflows |

**Example use case:** The Boys Traders exports last month's transactions to Excel with a blank voucher-type column. They upload the file and get a type back for every row, with a confidence score and a short reason. The few rows the system is unsure about are marked for review, so the accountant checks only those instead of the whole month.

---

## Open-Source AI

The project uses **Qwen3.8**, an open-source language model, run locally through **LM Studio**. The system doesn't depend on a paid cloud AI API.

The model is only used for transactions the rules can't settle with confidence. For each one it receives:

- The transaction's fields in structured form
- Accounting signals found during preprocessing
- Definitions of the relevant voucher types
- A few similar, already-labeled transactions retrieved from ChromaDB
- Instructions to answer only with one of the defined voucher categories

The prototype does **not** use fine-tuning. LoRA / QLoRA fine-tuning is future scope.

---

## Why This Technology

1. **Handles ambiguous transactions.** The model looks at several fields together, which single keywords can't do.
2. **Runs locally.** No dependency on a cloud AI service, and the prototype is easy to demo in a controlled setup.
3. **Works with RAG.** Retrieved examples show the model how similar transactions were classified before.
4. **Controlled output.** The answer must be one of the 27 categories, and both the category and the required fields are validated.
5. **Structured output.** Predictions are machine-readable, so they're easy to validate, evaluate and integrate.
6. **Fewer unnecessary LLM calls.** Rules handle clear cases, so the system is cheaper and more predictable.
7. **No fine-tuning needed yet.** There isn't a large labeled dataset, so prompting, rules and a small set of examples are the right fit for now.

---

## System Architecture

The system has five main stages:

1. Input and preprocessing
2. Rule-based classification
3. Retrieval and LLM reasoning
4. Confidence validation
5. Structured output and evaluation

| Component | Responsibility |
|---|---|
| Excel Dataset | Provides structured transaction records |
| Data Cleaning & Normalization | Prepares consistent input |
| Accounting Rule Engine | Handles clear cases with deterministic rules |
| RAG Retriever | Retrieves relevant examples for ambiguous cases |
| Chroma Vector Store | Stores labeled examples |
| Qwen3.8 | Contextual classification and reasoning |
| LM Studio | Local model execution |
| Confidence Check | Validates the prediction |
| Fallback Handling | Handles uncertain or invalid predictions |
| JSON Generator | Produces standardized output |

The design follows separation of concerns: rules make deterministic decisions, retrieval supplies examples, the model does contextual reasoning, and validation checks the result. That makes the system easier to test, debug, evaluate and extend.

---

## Workflow

This is a **staged workflow**, not a free-roaming autonomous agent. Every transaction goes through the same controlled sequence.

```
Transaction
     ↓
Preprocessing
     ↓
Rule-Based Classification
     ↓
Is the case clear?
   /          \
 Yes           No
  ↓             ↓
Rule         Retrieve Similar
Prediction   Labeled Examples
               ↓
           Chroma RAG
               ↓
            Qwen3.8
               ↓
        Structured JSON
               ↓
       Confidence Check
          /          \
      Confident     Uncertain
         ↓             ↓
      Accept       Fallback /
                   Review Flag
```

| Stage | Responsibility |
|---|---|
| Rules | Handle clear accounting cases |
| Chroma RAG | Retrieve relevant labeled examples |
| Qwen3.8 | Classify using context |
| Confidence check | Validate the prediction |
| Fallback | Deal with uncertain results |
| JSON output | Produce a standard result |

---

## Data Flow and Messy Data

What happens to every row:

1. Read the Excel file and clean each row so dates, amounts and company names look the same everywhere.
2. Note which important fields are blank. Rows with important blanks get flagged at the end.
3. Work out which side The Boys Traders is on (buyer, seller, both or neither) and give that to the model as a plain fact.
4. If a strong rule can settle the row, use it and skip the model. Employee pay details, for example, always mean salary.
5. Otherwise gather weaker hints and look up similar labeled rows.
6. Give the model the row, the hints and the examples; it picks one voucher type.
7. Check the answer. If it isn't one of the 27 types, or the model isn't confident, the row becomes Other / Miscellaneous.
8. Rows with important blanks or low confidence get a review flag. Every row goes out as JSON, in a table and on the upload page.

### Example: one row, start to finish

*This is how the system is expected to behave, not a recorded run.*

| Step | What happens |
|---|---|
| Input | Debit note DN-0007. Seller is Sharma Traders, buyer is The Boys Traders. 20 plastic storage containers, 4,000 before tax and 720 GST. It refers to invoice INV-S-1042 and says 20 units were damaged in transit and sent back. |
| Cleaning | Dates, amounts and names are put into one standard format. |
| Blank fields | Nothing important is missing, so no flag. |
| Which side are we on | We're the buyer, so the goods are going back to our supplier. |
| Rules | No strong rule settles it, but a weak rule notices the reference to an older invoice and the return wording, and hints at a return. |
| Similar examples | A few already-labeled return rows are shown to the model. |
| Model | It sees the row, the buyer fact, the hint and the examples, and replies with a voucher type, confidence and short reason. Expected: Purchase Return / Debit Note. |
| Check | The answer is one of the 27 types and the confidence is high enough, so it's accepted. |
| Output | A JSON record and a row in the results table on the upload page. |

### When the data is messy

- **Blank fields.** Blanks are allowed. If an important field is empty, the row still gets an answer but is flagged for review, and the output lists the missing fields. Only fields that matter for that kind of row are flagged (a salary row has no seller, and that's normal).
- **Narration that disagrees with the fields.** Structured fields win over free text. If the narration says "sale of goods" but The Boys Traders is the buyer, it isn't treated as a sale.
- **Unusable output.** If the model returns broken JSON or a type outside the 27, the system retries once. If that fails too, the row goes to Other / Miscellaneous with a review flag.

A flagged row looks like this (an example of the format, not a real result):

```json
{
  "invoice_number": "INV-S-1050",
  "voucher_type": "Purchase",
  "confidence": 0.6,
  "reason": "The Boys Traders is the buyer, but items and amounts are missing.",
  "needs_review": true,
  "missing_fields": ["item_description", "taxable_value", "date"]
}
```

---

## Technology Stack

| Technology | Purpose |
|---|---|
| Python | Core development and integration |
| Pandas + openpyxl | Transaction processing and Excel handling |
| Qwen3.8 | Primary LLM for contextual classification |
| LM Studio | Local LLM inference and model serving |
| ChromaDB + Sentence Transformers | RAG: retrieve similar reference transactions |
| scikit-learn | Model evaluation |
| Streamlit | Interactive demo interface |
| Git + GitHub | Version control and collaboration |

All core components are open-source or openly available, so the classification pipeline doesn't rely on a proprietary cloud AI API.

---

## Features

**Core**

- Reads an Excel file and processes every row.
- Picks one voucher type per row, from the 27 allowed types.
- Works out whether The Boys Traders is the buyer or seller in each row and passes that to the model as a fact.
- Uses simple rules for clear cases: employee pay details, foreign-currency rows with customs details, and transfers between the company's own cash and bank accounts.
- Uses a local open-source model for rows the rules can't settle.
- Shows the model a few similar, already-solved examples for tricky cases.
- Returns JSON for every row with the voucher type, a confidence score and a short reason.
- Flags a row for review when important fields are blank or the model isn't confident.
- Checks that every answer is one of the 27 types; otherwise the row becomes Other / Miscellaneous.
- Keeps rules and company-name spellings in a settings file, so they can change without touching the main code.
- Provides an upload page where an evaluator can upload a file and view results.
- Includes an evaluation script reporting accuracy, plus precision, recall and F1 per category, on rows with known answers.

**Planned**

- Download results as an Excel file.
- Highlight rows that need review on the upload page.
- Filter results by voucher type or confidence.
- A report showing which similar voucher types get confused.

---

## Input and Output

**Input:** an Excel (`.xlsx`) file with fields such as transaction/invoice number, date, seller/buyer, description, amount and GST, payment information, inventory or return information, and other available details. The voucher type is *not* provided; the system generates it.

**Output:** each transaction gets one of the 27 voucher categories.

```json
{
  "invoice_number": "INV-2026-1042",
  "voucher_type": "Purchase",
  "confidence": 0.91,
  "reason": "The transaction contains supplier-side purchase information, item details, taxable value and GST consistent with a purchase transaction."
}
```

**Interface (Streamlit):** Upload Excel → Validate → Classify transactions → View results → Export results.

### Validation and guardrails

Every prediction, whether from the rules or from the model, is checked before it becomes a final result:

- Valid structured output
- Required fields present
- Valid voucher category
- Consistency checks
- Invalid-response handling

---

## Evaluation

The system is evaluated on previously unseen labeled transactions using:

- Accuracy
- Precision, recall and F1-score
- Macro F1
- Confusion matrix
- Per-category performance

Different setups are compared to see which components actually help:

```
Rules  →  Qwen3.8 via LM Studio  →  Rules + Qwen3.8  →  Rules + Qwen3.8 + RAG
```

---

## Challenges and Mitigation

| Challenge | Why it's hard | What we'll do |
|---|---|---|
| Knowing who "we" are | The company name can be spelled differently across rows, or be missing | Keep a list of spellings for The Boys Traders; if none match, assume the most frequent party is us |
| Types that look alike | Purchase/Sales, the two returns, and Contra vs Payment/Receipt have almost the same fields | Work out direction first in code, and include paired examples in the prompt that show both sides |
| Missing or contradictory fields | Narration can disagree with seller and buyer, and some cells will be empty | Structured fields win over narration; if we truly can't tell, the row goes to Other / Miscellaneous with low confidence |
| No labeled data to train on | The real dataset hides the voucher column | Use prompts, rules and a few labeled examples; fine-tuning comes later |
| Rare types | Job work, rejections and material in/out are easy to mix up | Give them their own rules and extra examples |
| Inconsistent model answers | The same row can come back different on different runs | Temperature 0, fixed JSON format, a validator and one retry |
| Slow on laptops | Bigger models crawl without a GPU | A small quantized model, short outputs, and rules that skip the model when possible |
| Testing without the hidden answers | The real labels aren't available during development | Test on our own labeled sample set and hold back rows that are never tuned on |

---

## Future Scope

- **Model improvement:** LoRA / QLoRA fine-tuning, better handling of difficult categories, organization-specific accounting terminology.
- **Human-in-the-loop:** accountants review low-confidence rows, and verified results become future reference data.
- **Accounting integration:** Tally and other accounting platforms.
- **Multilingual support:** Indian regional languages, mixed-language descriptions and transliteration.
- **Customization:** organizations configure their own rules, voucher definitions, reference examples and validation requirements.
- **Production deployment:** Frontend → Backend API → Model serving → Database → Monitoring.
- **Beyond classification:** voucher creation, transaction validation, exception detection, workflow automation and human-review prioritization.
