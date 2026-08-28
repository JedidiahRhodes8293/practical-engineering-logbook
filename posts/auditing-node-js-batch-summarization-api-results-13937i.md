# Auditing Node.js Batch Summarization API Results for Multiple Documents

Short answer: Make the Node.js batch summarization API an asynchronous producer of per-document records, then export results only after a separate evaluator has checked coverage, provenance, and the batch's declared completion policy.

The important trade-off is latency versus evidence. Returning a job ID quickly is useful, but it doesn't prove that every document produced a usable summary. A trustworthy design separates intake, generation, evaluation, and publication, so a caller can tell the difference between “the work was accepted,” “the workers stopped,” and “the result is fit to download.” The Python worker example below shows that contract without tying it to a vendor or pretending a notebook loop is already a production service.

## How should a Node.js batch summarization API export multiple document results?

Use Node.js as the HTTP boundary if that matches the rest of the application, but keep the job contract independent of the web framework. On submission, validate unique document IDs, assign a stable batch ID, record a content digest for each input, and persist the complete item set before acknowledging the request. Workers can then claim individual items under bounded concurrency. A second pass evaluates each summary against its own source. Only the publisher decides whether the batch is downloadable.

That final distinction prevents a subtle but common failure: treating `worker_finished` as `publishable`. A worker can finish with an empty summary, an output that omits the one sentence the reader cares about, or a valid result generated from the wrong revision of a document. None of those cases requires a transport failure. An evaluation record catches them at the boundary where they matter.

The caller needs a compact state model. `accepted` means all input records are durable. `running` means at least one item can still change. `evaluating` means generation is terminal but quality checks are not. `ready` means the export policy passed. `completed_with_errors` means the batch is terminal and its manifest identifies the failed items. Don't collapse those states into a percentage; `99%` cannot say whether one slow document is merely queued or permanently excluded.

The data flow is plain: the Node.js layer accepts documents and returns a batch ID; Python workers summarize items; evaluators compare summaries with sources and store verdicts; the publisher writes an immutable JSONL artifact plus a manifest. Polling reads recorded state rather than inspecting worker processes. A webhook can be added later, but it should announce the same terminal record that polling exposes — two notification paths must not invent two meanings of done.

## A runnable Python worker and evaluation gate

This example deliberately keeps model generation deterministic. That makes the orchestration testable without network access or token spend, while leaving one narrow function to replace with a model adapter. It writes one JSONL line per input and refuses publication when an item lacks a passing evaluation. The short summary rule is simplistic; the job and evidence boundaries are the point.

```python
import asyncio
import hashlib
import json
from dataclasses import asdict, dataclass
from pathlib import Path


@dataclass(frozen=True)
class Document:
    document_id: str
    text: str


@dataclass(frozen=True)
class Result:
    document_id: str
    source_digest: str
    summary: str
    evaluation: str
    error_code: str | None


def digest(text: str) -> str:
    return hashlib.sha256(text.encode("utf-8")).hexdigest()


def summarize(text: str) -> str:
    sentences = [part.strip() for part in text.split(".") if part.strip()]
    return ". ".join(sentences[:2]) + ("." if sentences else "")


def evaluate(document: Document, summary: str) -> tuple[str, str | None]:
    if not summary.strip():
        return "failed", "empty_summary"
    source_terms = set(document.text.lower().split())
    summary_terms = set(summary.lower().split())
    if not source_terms.intersection(summary_terms):
        return "failed", "source_overlap_missing"
    return "passed", None


async def process(
    document: Document, semaphore: asyncio.Semaphore
) -> Result:
    async with semaphore:
        summary = await asyncio.to_thread(summarize, document.text)
        verdict, error_code = evaluate(document, summary)
        return Result(
            document_id=document.document_id,
            source_digest=digest(document.text),
            summary=summary,
            evaluation=verdict,
            error_code=error_code,
        )


async def run_batch(documents: list[Document], concurrency: int) -> list[Result]:
    document_ids = [document.document_id for document in documents]
    if len(document_ids) != len(set(document_ids)):
        raise ValueError("document IDs must be unique within a batch")
    semaphore = asyncio.Semaphore(concurrency)
    return await asyncio.gather(
        *(process(document, semaphore) for document in documents)
    )


def publish(results: list[Result], destination: Path) -> dict[str, object]:
    failed = [result for result in results if result.evaluation != "passed"]
    if failed:
        codes = sorted({result.error_code for result in failed})
        raise RuntimeError(f"publication blocked by evaluation: {codes}")

    payload = "".join(json.dumps(asdict(result)) + "\n" for result in results)
    destination.write_text(payload, encoding="utf-8")
    return {
        "item_count": len(results),
        "artifact_digest": digest(payload),
        "format": "jsonl",
    }


async def main() -> None:
    documents = [
        Document("policy-17", "Retention is thirty days. Deletion is automatic."),
        Document("memo-42", "The launch moved to Tuesday. Review stays on Friday."),
    ]
    results = await run_batch(documents, concurrency=4)
    manifest = publish(results, Path("summary-results.jsonl"))
    print(json.dumps(manifest, indent=2))


if __name__ == "__main__":
    asyncio.run(main())
```

Short code. Strict boundary.

In production, the in-memory gather becomes durable item claims, and the deterministic function becomes a versioned model adapter. Preserve the interface. Every result should carry the source digest, model configuration, prompt version, evaluation version, attempt identity, and terminal error code. Those fields answer the painful rerun question: did the source change, did generation change, or did the judge change?

The evaluator shown above is intentionally weak because lexical overlap alone cannot establish summary quality. A practical harness combines deterministic assertions with corpus-specific examples: required entities, forbidden claims, date and number preservation, empty-input behavior, citation coverage, and human-labelled usefulness. I'm not sure one generic semantic score can resolve all of those; a legal memo and a support transcript punish different omissions. The evidence that would change the policy is performance on a versioned, representative eval set, reviewed by the people who consume the summaries.

This is the notebook-to-prod bridge. Start with a handful of labelled documents beside an exploratory notebook, freeze them as fixtures, and run them whenever the prompt, model, chunker, or reduction step changes. A batch system makes bad configuration move faster too. Evals decide whether faster is acceptable.

## Completion policy is a product decision

There are two defensible publication policies. An all-or-nothing export blocks until every item passes. It fits regulated or reconciliation-heavy workflows where omission is worse than delay. A partial export publishes terminal successes and includes explicit failed records in its manifest. It fits review queues where useful work should arrive even if one malformed input needs attention.

| Policy | Publication condition | Caller receives | Better fit |
|---|---|---|---|
| All or nothing | Every item passes evaluation | One complete artifact or no artifact | Reconciliation-heavy workflows |
| Partial results | Every item is terminal | Passed summaries plus explicit failure records | Human review queues |

Pick one in the API contract. A client shouldn't infer it from whether a file happens to exist.

For either policy, the manifest needs enough information to reconcile the artifact without reading every summary: batch ID, submitted count, terminal count, passed count, failed count, export format, artifact digest, and the versions of generation and evaluation policy. The invariant is simple: passed plus failed equals submitted before publication. If partial output is allowed, failed document IDs and machine-readable error codes belong in the artifact or a companion manifest, never only in logs.

Retries require similar precision. Retry transient model-call failures with bounded backoff, but don't blindly retry deterministic validation failures such as empty source text. A lease lets another worker reclaim abandoned work; an attempt ID prevents an older worker from overwriting a newer result after its lease expires. Duplicate execution can happen in distributed systems, so publication should select one committed terminal record for each document ID. This is where a content digest earns its keep — the publisher can prove that each summary corresponds to the accepted source revision.

Consider a concrete failure drill with 1,000 accepted documents. Worker A claims document 640, generates a summary, and pauses before committing it. Its lease expires, so worker B claims the same item and commits a passing result under a newer attempt ID. Worker A resumes and tries to write its older result. Meanwhile, 998 other items have passed and document 912 has failed deterministic validation because its accepted body is empty. A weak implementation lets worker A overwrite worker B, reports 100% because all tasks stopped, and exports 999 lines without explaining the omission. The stricter contract rejects A's stale commit, records 912 as `failed` with `empty_source`, derives terminal counts from the item ledger, and applies the declared publication policy. Under all-or-nothing rules, no artifact becomes visible. Under partial-result rules, the artifact contains 999 passing rows and one explicit failure row, while its manifest still reconciles to all 1,000 accepted IDs. No model benchmark is needed to test this drill; controlled worker timing and ledger assertions are enough. This is the sort of unglamorous eval case that keeps a promising notebook from becoming an untraceable production bill.

Cancellation also needs a declared meaning. It can stop unclaimed work, request interruption of active work, or merely prevent publication. Those are different promises. Record the chosen behavior and keep completed item records long enough for reconciliation under the application's retention policy.

## Token budgets and cross-document evaluation

Count tokens before dispatch and store actual usage after generation. The official `tiktoken` project provides a BPE tokenizer library for supported encodings, but the selected encoding must match the model configuration being estimated. An estimate is a routing input, not a quality score. It can decide whether to reject, chunk, or route a document before workers spend the batch budget.

Chunking creates its own quality risk. A map step may summarize each section accurately while the reduction step erases contradictions between documents. A single oversized synthesis can preserve cross-document context but makes selective retry and source attribution harder. The better choice depends on the corpus, so test both on examples containing repeated names, conflicting dates, minority findings, and one deliberately irrelevant long document. Measure source coverage and contradiction handling alongside readability.

Prompt cost belongs in the job ledger. Record input and output token counts per attempt and enforce a per-batch ceiling before admitting more work. If the ceiling is reached, remaining items need an explicit terminal policy; silently dropping them creates a polished but incomplete export. Cost controls should also be exercised by tests, because an accidental prompt expansion can change the operational shape of a batch long before anyone notices a monthly total.

There is a real limitation here: a durable queue, evaluator, and immutable export add storage, migrations, and operational ownership. They are not suitable for a disposable notebook run where losing ten summaries is harmless. An in-process loop is clearer for that case. A batch API is also the wrong interaction for token streaming, human approval between documents, or inputs governed by incompatible retention rules; keep those flows separate rather than forcing them into one job.

## Ship the evidence, not merely the text

Before deployment, walk one batch through acceptance, worker loss, duplicate delivery, evaluation failure, partial completion, cancellation, and artifact verification. Confirm that polling is safe at any frequency, terminal states never move backward, and the downloaded bytes match the manifest digest. Then run the eval corpus under the exact prompt, model, tokenizer, chunker, and reduction versions that will enter production.

Observability should follow the same boundaries. Track queue age, claim duration, generation attempts, evaluation verdicts, token use, terminal item counts, publication delay, and reconciliation failures. Logs need the batch ID, document ID, and attempt ID so one document can be traced without searching raw text. Avoid putting source content or summaries in logs by default; application retention and access rules still apply to generated text.

Finally, make the Node.js client boring: submit once with an idempotency key, retain the returned batch ID, poll with backoff or consume a signed completion notification, verify the artifact digest, and reconcile document IDs before handing summaries downstream. The exciting work belongs in eval design. The API's job is to make missing evidence impossible to mistake for success.

## Sources

- https://github.com/openai/tiktoken
