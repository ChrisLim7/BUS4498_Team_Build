# Resolve Review Exception Task Specification

## Basic Information

- **Task ID:** T7
- **Task name:** Resolve Review Exception
- **Task type:** Decide
- **Task owner:** Club treasurer (the club president decides when the treasurer submitted the request)

## 1. Task Description

The treasurer handles every case PayBack cannot finish safely on its own:

- possible duplicate submissions
- receipts that contradict the request
- rules that are unclear or don't cover the case
- prohibited items
- requests still needing fixes after 2 rounds
- tool failures or uncertain message sends in T1, T2, T4, T5, or T8

The treasurer reads the exception note and evidence, then makes one of three decisions:

- **Ready for approval:** the request continues to T5.
- **Member must fix:** the request goes back to the member through T4. The treasurer may allow one extra round, recorded with a reason.
- **Reject or close:** the outcome goes to T8.

When a tool failed, the treasurer can also fix the cause, such as a missing rules file. This is a human task because these cases need judgment, and some, like a possible duplicate, could affect a member's standing.

**Human response deadline:** 3 business days after the exception is assigned. A missed deadline is not a decision; the case stays open, and the club president is notified.

## 2. Inputs

### Input 1

- **Input name:** Exception case
- **Contents and format:** Request ID, the task that escalated it, the reason, the evidence summary and unresolved issues, and the request record.
- **Source:** T1, T2, T3, T4, T5, or T8

- **If a required input is missing or invalid:** If the exception note is incomplete, the treasurer can still open the request record and receipt directly. The case stays open until the treasurer records a decision.

## 3. Outputs

### Output 1

- **Output name:** Exception decision
- **Contents and format:** Request ID, decision (ready for approval, member must fix, or reject or close), reason, fix list when applicable, whether an extra fix round is approved, decision maker's name, and timestamp.
- **Next task or recipient:** Ready for approval → T5: Prepare Treasurer Review Packet. Member must fix → T4: Request Missing Information. Reject or close → T8: Record Request Outcome.
- **Complete when:** The decision is saved and read back with the decision maker's name and timestamp.

## 4. Planned Tools

### Tool 1

- **Tool name:** `record_exception_decision`
- **Input:** Exception case; the treasurer's selected decision
- **Output:** Exception decision
- **Implementation Route:** Database queries (the review page saves the decision keyed on request ID and exception number, then reads it back)
- **Integration approach:** Direct integration
- **Role in this task:** Shows the exception with its evidence and records the treasurer's decision. It cannot decide anything on its own.
- **Task timeout:** Human response deadline: 3 business days after the exception is assigned.
- **Maximum retries:** Not applicable — manual task.
- **Retry only when:** Not applicable.
- **On timeout, exhausted retries, or an error that cannot be retried:** If the deadline passes, mark the case "overdue" and notify the club president; the request stays open and is never treated as approved. If saving fails, the decision does not count until it is saved and read back.
