# Review 1 — Action Items

Based on reviewer feedback after first submission.

---

## Fix 1 — Add confidence bounds and defaults to `AnswerResponse`

**File:** `starter/src/schemas.py` lines 18–24

**Problem:** `AnswerResponse.confidence` has no bounds or default, so the model can return any float (including values > 1 or < 0) without validation failing. The reviewer wants more responsibility in the schema itself.

**Fix:**
Status: Done
```python
class AnswerResponse(BaseModel):
    """Structured response for Q&A tasks"""
    question: str
    answer: str
    sources: List[str] = Field(default_factory=list, description="List of source document IDs")
    confidence: float = Field(default=0.0, ge=0.0, le=1.0, description="Confidence score between 0 and 1")
    timestamp: datetime = Field(default_factory=datetime.now)
```

Changes:
- `confidence` gets `ge=0.0, le=1.0` bounds + a `default=0.0` so missing values don't crash
- `sources` gets `default_factory=list` so an empty list is valid
- `timestamp` gets `default_factory=datetime.now` so the model doesn't have to produce it

---

## Fix 2 — Calculator tool must return a string, not a dict
Status: Done
**File:** `starter/src/tools.py` lines 71–104

**Problem:** The `calculator_tool` currently returns a `dict`. LangChain tools are expected to return a `str`. Returning a non-string breaks the tool contract and causes issues when the result is passed back into the LLM as a tool message.

**Fix:** Keep the rich metadata in the log (already done via `logger.log_tool_use`), but return a plain string from the tool itself:

```python
@tool
def calculator_tool(mathematical_expression: str) -> str:
    """
    Performs mathematical calculation.

    Args:
        mathematical_expression: a mathematical expression as input.

    Examples:
    - "2+2" -> "Result: 4"
    - "2*2*4" -> "Result: 16"

    Returns:
        Result of the computation as a string
    """
    if not re.match(r'^[\d\s\+\-\*\/\.\(\)\%\^]+$', mathematical_expression):
        result_str = f"Error: Unsafe expression rejected: `{mathematical_expression}`."
        logger.log_tool_use("calculator_tool", {"expression": mathematical_expression}, {"error": result_str})
        return result_str
    try:
        value = eval(mathematical_expression)
        result_str = f"Result: {value} (expression: {mathematical_expression})"
        logger.log_tool_use(
            "calculator_tool",
            {"expression": mathematical_expression},
            {"expression": mathematical_expression, "result": value, "explanation": f"Evaluated `{mathematical_expression}` to get {value}."}
        )
        return result_str
    except Exception as e:
        result_str = f"Error: Could not evaluate `{mathematical_expression}`. {e}"
        logger.log_tool_use("calculator_tool", {"expression": mathematical_expression}, {"error": result_str})
        return result_str
```

---

## Fix 3 — Add `total` to INV-001 metadata so invoice totals are consistent
Status: 
**File:** `starter/src/retrieval.py` line 49

**Problem:** INV-002 and INV-003 have a `total` key in their metadata, but INV-001 does not. When the calculation agent searches for invoice totals, `_get_document_amount` finds nothing for INV-001 and it gets silently skipped. This makes the sum of all invoices wrong and non-deterministic.

The document content says the total is $22,000 (subtotal $20,000 + tax $2,000).

**Fix:** Add `"total": 22000` to INV-001's metadata:

```python
Document(
    doc_id="INV-001",
    title="Invoice #12345",
    content="""...""",  # unchanged
    doc_type="invoice",
    metadata={"total": 22000, "client": "Acme Corporation", "date": "2024-01-15"}
),
```

After this fix, re-run the calculation query `What is the total value of all invoices combined?` and **save the updated session/log files** so the submitted artifacts reflect the correct $305,800 total.

---

## Fix 4 — Refresh saved session and log artifacts

**Problem:** The reviewer noted the saved artifacts had inconsistent invoice totals. After Fix 3, delete the old sessions and logs and re-run all three example flows to regenerate clean artifacts.

```bash
cd starter
rm -rf sessions/ logs/
python main.py
```

Run these three queries in order:
1. `What are the payment terms for invoice INV-002?` (Q&A)
2. `Summarize all contracts` (Summarization)
3. `What is the total value of all invoices combined?` (Calculation — should now return $305,800)

This gives the reviewer fresh, consistent end-to-end evidence.

---

## Fix 5 — Align `final-readme.md` with actual workspace

**Problem:** The reviewer noted alignment issues between the README and the actual workspace. Review `final-readme.md` against the code and fix any discrepancies:

- [ ] Example 1 (Q&A): INV-001 payment terms — verify the assistant actually returns "Net 30 days" and update if different
- [ ] Example 3 (Calculation): Confirm the total shown ($305,800) matches what the fixed app now produces
- [ ] Project structure table: confirm all listed files exist in the submitted folder
- [ ] Tool list in "Built With": confirm versions match `requirements.txt`

---

## Summary Table

| # | File | Change | Priority |
|---|------|--------|----------|
| 1 | `schemas.py:18` | Add `ge/le` bounds + defaults to `AnswerResponse` | High |
| 2 | `tools.py:72` | Change calculator tool return type from `dict` to `str` | High |
| 3 | `retrieval.py:49` | Add `"total": 22000` to INV-001 metadata | High |
| 4 | `sessions/`, `logs/` | Delete and regenerate artifacts after Fix 3 | High |
| 5 | `final-readme.md` | Verify examples and structure match actual code | Medium |
