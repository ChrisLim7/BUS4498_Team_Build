# Review Request Against Funding Rules Task Specification

```yaml
# BASIC INFORMATION
task_id: "T3"
task_name: "Review Request Against Funding Rules"
task_owner: "PayBack agent; the club treasurer keeps all approval authority"

# Agent Inference Configuration
Provider: Claude
Model: "claude-sonnet-5-5"
Role: Interpret the request, receipt details, and funding rules; select the next permitted subtask; classify the expense; and write a member-facing fix list.
Maximum inference requests per task run: 5
On inference failure or exhausted limits: Record the unresolved status and hand the case to the club treasurer.
```

## 1. Task Goal

- **Objective:** Decide whether one reimbursement request is ready for the treasurer, needs specific fixes the member can make, or needs the treasurer's judgment. The decision must be supported by recorded evidence for every applicable funding rule, so that requests reaching the treasurer are complete the first time. The task never approves, rejects, or pays a request.

## 2. Inbound Inputs

### Input 1

- **Input name:** Request record
- **What it contains:** Request ID, fix round number (0, 1, or 2), member name and Cal Poly email, club name, amount requested, purchase date, purchase description, event name, the expense category the member selected, optional pre-approval reference, and links to the receipt files. No bank or payment details.
- **Source:** T1: Retrieve Reimbursement Request

### Input 2

- **Input name:** Receipt details
- **What it contains:** Vendor, purchase date, line items (description, quantity, and price), subtotal, tax, total, whether the receipt is itemized, a high or low confidence level for each field, and any fields that could not be read.
- **Source:** T2: Extract Receipt Details

### Input 3

- **Input name:** Club funding rules
- **What it contains:** The current version of the club's funding rules document, including allowed expense categories, itemized-receipt requirements, pre-approval requirements and amount thresholds, prohibited items, and the submission deadline after purchase.
- **Source:** T1: Retrieve Reimbursement Request (rules document maintained by the club treasurer)

### Input 4

- **Input name:** Request history
- **What it contains:** The club's past requests from the outcome log: request ID, member, vendor, purchase date, total, receipt file fingerprint, and final outcome.
- **Source:** T1: Retrieve Reimbursement Request (from the T8 outcome log)

## 3. Tool Permissions and Boundaries

### Task-Wide Limits

- **Total task timeout:** 2 minutes for one task run, including all tool calls, retries, and waiting.
- **Maximum tool calls:** 12 calls across all tools during one task run; retries count toward this total.

### Tool 1

- **Tool name:** `lookup_funding_rule`
- **Tool type:** File operation (read-only search of the club funding rules document)
- **Supports these permitted subtasks:** Classify Expense Category; Check Rule Compliance
- **Allowed use:** Read the sections of the current funding rules document that apply to the request's expense category, amount, and purchase date.
- **Prohibited use:** Editing the rules, reading rules from any other club or source, or treating an outside website or policy as a rule.
- **Approval required:** None within the allowed use.
- **Timeout per call:** 5 seconds.
- **Maximum retries per call:** 1
- **Retry conditions and failure response:** Retry once after 2 seconds if the file cannot be read because of a temporary error. If it still fails, stop and escalate to the club treasurer with the status "rules unavailable." The tool is read-only, so a retry cannot change anything.

### Tool 2

- **Tool name:** `compare_receipt_to_request`
- **Tool type:** Python script (deterministic comparison)
- **Supports these permitted subtasks:** Match Receipt to Request
- **Allowed use:** Compare the receipt's total, purchase date, and vendor with the amount, date, and description on the request form, and return each difference.
- **Prohibited use:** Changing the member's requested amount or the receipt details.
- **Approval required:** None within the allowed use.
- **Timeout per call:** 5 seconds.
- **Maximum retries per call:** 0
- **Retry conditions and failure response:** No retries; the result is deterministic. On failure, record "comparison failed" and escalate to the club treasurer. A second call after another receipt file is examined is a new call, not a retry.

### Tool 3

- **Tool name:** `check_duplicate_submission`
- **Tool type:** Database query (read-only on the request history)
- **Supports these permitted subtasks:** Check for Duplicate Submission
- **Allowed use:** Search the club's past requests for the same receipt fingerprint, or the same vendor, date, and total.
- **Prohibited use:** Reading other clubs' records, contacting members about a possible duplicate, or changing any record.
- **Approval required:** None within the allowed use.
- **Timeout per call:** 10 seconds.
- **Maximum retries per call:** 1
- **Retry conditions and failure response:** Retry once after 2 seconds on a temporary database error. If it still fails, do not assume there is no duplicate; escalate to the club treasurer with the status "duplicate check unavailable." Read-only, so no duplicates are created.

### Tool 4

- **Tool name:** `save_review_result`
- **Tool type:** Database query (write, as an upsert keyed on request ID and fix round)
- **Supports these permitted subtasks:** Draft Fix List; final recording of the result
- **Allowed use:** Save the review result, rule-by-rule evidence, and fix list for this request and round, then read it back.
- **Prohibited use:** Changing a request's approval status, editing past rounds, or writing to the outcome log (T8 does that).
- **Approval required:** None within the allowed use.
- **Timeout per call:** 5 seconds.
- **Maximum retries per call:** 1
- **Retry conditions and failure response:** Retry once after 2 seconds only if a read-back on the same request ID and round shows no saved result. Because the write is an upsert on that key, it cannot create a second result. If the read-back fails or shows a different result, the outcome is uncertain: stop and escalate to the club treasurer.

## 4. How the Agent Should Reason

### Permitted Subtask 1

- **Subtask name:** Match Receipt to Request
- **Subtask description:** Compares the receipt's total, date, and vendor with the request form and produces a list of matches and differences.
- **Subtask boundary:** May only compare. A difference the member can fix, such as a typo in the amount, becomes a fix. A receipt from a different vendor or date than the form describes goes to the treasurer.
- **Retry limits:** 1 additional attempt, only when the request has more than one receipt file.

### Permitted Subtask 2

- **Subtask name:** Classify Expense Category
- **Subtask description:** Uses the line items and purchase description to choose the funding category that fits, and notes whether it matches the category the member selected.
- **Subtask boundary:** Only categories listed in the funding rules may be used. If items fit no category, or fit more than one, the case goes to the treasurer.
- **Retry limits:** 1 additional attempt after new rule text is retrieved.

### Permitted Subtask 3

- **Subtask name:** Check Rule Compliance
- **Subtask description:** Checks each rule that applies to the category, such as itemized receipt, pre-approval above the rules' threshold, prohibited items, and submission deadline, and records pass, fail, or cannot tell, with evidence.
- **Subtask boundary:** Apply the rules exactly as written. Do not invent, waive, or reinterpret a rule. A rule that is unclear or does not cover the case goes to the treasurer.
- **Retry limits:** Each rule may be checked at most twice.

### Permitted Subtask 4

- **Subtask name:** Check for Duplicate Submission
- **Subtask description:** Searches past requests for the same receipt or the same vendor, date, and total, and records whether a possible duplicate exists.
- **Subtask boundary:** May only flag. A possible duplicate always goes to the treasurer; never tell the member they submitted a duplicate.
- **Retry limits:** 0

### Permitted Subtask 5

- **Subtask name:** Draft Fix List
- **Subtask description:** Turns every failed item the member can correct into a short, specific instruction, such as "Upload the itemized receipt from the vendor, not the card slip."
- **Subtask boundary:** Only include fixes the member can make. Do not promise approval, and do not send anything; T4 sends the message.
- **Retry limits:** 1

- **Decision guidance:** After each subtask, use its findings to select the permitted subtask most likely to resolve the most important remaining uncertainty. Do not follow a fixed sequence. If no permitted subtask can make useful progress, stop and hand the case to a person.

## 5. When to Stop or Hand Off to a Human

- **Stop successfully when:** The review result is saved and read back, the receipt has been compared with the form, a possible duplicate has been ruled out, and either every applicable rule shows a pass with evidence (Ready) or every failed item has a specific fix the member can make (Needs fixes). Confidence alone is not enough.
- **Hand off early when:** A possible duplicate is found; the receipt contradicts the request in a way the member cannot fix; a rule is unclear or does not cover the case; a prohibited item appears; fixes are still needed after 2 rounds; any tool fails after its retries; or a time, tool-call, or inference limit is reached.
- **Hand off to:** The club treasurer, through T7: Resolve Review Exception.

Stop at the first applicable budget limit or handoff condition. While awaiting review, take no further autonomous action.

## 6. Outbound Deliverable

- **Status:** Completed (Ready or Needs fixes) or escalated to human.
- **Result or recommendation:** "Ready for treasurer review" or "Needs fixes" with the fix list. If escalated before reaching a supported result, write undetermined.
- **Evidence summary:** The receipt-to-form comparison, the expense category with its rule reference, the pass or fail result and evidence for each applicable rule, and the duplicate check result.
- **Subtasks performed:** Permitted subtasks completed, including repeated attempts.
- **Unresolved issues:** Remaining uncertainties or questions; use none only if no unresolved issue remains.
- **Handoff note:** Reason for stopping, unresolved questions, and what the treasurer needs to decide; write "Not applicable" for a completed task.
- **Next task or recipient:** Ready → T5: Prepare Treasurer Review Packet. Needs fixes → T4: Request Missing Information. Escalated → T7: Resolve Review Exception.
