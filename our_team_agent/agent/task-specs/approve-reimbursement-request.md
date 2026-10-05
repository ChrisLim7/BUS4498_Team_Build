# Approve Reimbursement Request Task Specification

## Basic Information

- **Task ID:** T6
- **Task name:** Approve Reimbursement Request
- **Task type:** Decide
- **Task owner:** Club treasurer (the club president decides when the treasurer submitted the request)

## 1. Task Description

The treasurer reviews the packet and makes one of three decisions:

- **Approve:** the request goes to payment through the club's existing payment process, outside PayBack.
- **Return for changes:** the treasurer gives a specific reason the member can act on.
- **Reject:** the treasurer gives a reason.

This is a human decision because it commits the club's money. PayBack never approves or pays a request. A return counts toward the charter's return-rate measure, so the treasurer's reason is recorded exactly.

**Human response deadline:** 3 business days after the packet appears in the review queue. A missed deadline is not approval. The request stays pending, and the case goes to the club president.

## 2. Inputs

### Input 1

- **Input name:** Treasurer review packet
- **Contents and format:** Summary, request details, receipt file, rule-by-rule results with evidence, and fix history.
- **Source:** T5: Prepare Treasurer Review Packet

- **If a required input is missing or invalid:** If the packet does not load or is missing the receipt or rule results, the treasurer does not decide. The case goes to T7: Resolve Review Exception.

## 3. Outputs

### Output 1

- **Output name:** Treasurer decision
- **Contents and format:** Request ID, decision (approve, return, or reject), reason for a return or rejection, decision maker's name, and timestamp.
- **Next task or recipient:** Approved or rejected → T8: Record Request Outcome. Returned → T4: Request Missing Information, with the treasurer's reason as the fix list.
- **Complete when:** The decision is saved and read back with the decision maker's name and timestamp.

## 4. Planned Tools

### Tool 1

- **Tool name:** `record_treasurer_decision`
- **Input:** Treasurer review packet; the treasurer's selected decision
- **Output:** Treasurer decision
- **Implementation Route:** Database queries (the review page saves the decision keyed on request ID and fix round, then reads it back)
- **Integration approach:** Direct integration
- **Role in this task:** Records the treasurer's decision only. It cannot approve, return, or reject a request on its own.
- **Task timeout:** Human response deadline: 3 business days after the packet appears in the review queue.
- **Maximum retries:** Not applicable — manual task.
- **Retry only when:** Not applicable.
- **On timeout, exhausted retries, or an error that cannot be retried:** If the deadline passes, record "decision overdue" and notify the club president; the request stays pending, never approved. If saving fails, the review page shows the error, and the decision does not count until it is saved and read back.
