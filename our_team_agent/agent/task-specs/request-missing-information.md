# Request Missing Information Task Specification

## Basic Information

- **Task ID:** T4
- **Task name:** Request Missing Information
- **Task type:** Act
- **Task owner:** PayBack workflow controller; the club treasurer approves the message template

## 1. Task Description

This task sends the member one email listing exactly what to fix, using a fixed message template the treasurer has approved, and gives the member 5 days to resubmit. The fix list comes from T3, from the missing-items list in T1's intake status, or from the treasurer's reason in T6 or T7. The task adds 1 to the request's fix round. A member can receive at most 2 fix requests per request; if fixes are still needed after that, the case goes to the treasurer. A resubmission within 5 days starts a new run at T1. If the deadline passes with no resubmission, the request is closed as incomplete and recorded in T8. The workflow needs this task so members fix problems before the treasurer ever sees the request, which is how PayBack lowers the return rate.

## 2. Inputs

### Input 1

- **Input name:** Fix list
- **Contents and format:** Short, specific instructions for the member, each tied to the rule or missing item it addresses.
- **Source:** T3: Review Request Against Funding Rules, the T1 intake status (through the D1 check), T6: Approve Reimbursement Request, or T7: Resolve Review Exception

### Input 2

- **Input name:** Request record
- **Contents and format:** Request ID, current fix round, member name, and Cal Poly email.
- **Source:** T1: Retrieve Reimbursement Request

### Input 3

- **Input name:** Fix request template
- **Contents and format:** Approved subject line and message body with placeholders for the fix list, resubmission link, and deadline.
- **Source:** Club treasurer

- **If a required input is missing or invalid:** If the fix list is empty, the template is missing, or the request has already had 2 fix rounds (unless the treasurer approved another round in T7), do not send. Send the request to T7: Resolve Review Exception.

## 3. Outputs

### Output 1

- **Output name:** Fix request message
- **Contents and format:** One email to the member with the fix list, the resubmission link, and the 5-day deadline.
- **Next task or recipient:** The club member
- **Complete when:** The email service confirms it accepted the message.

### Output 2

- **Output name:** Fix request record
- **Contents and format:** Request ID, fix round, send time, send status, fix list, and resubmission deadline.
- **Next task or recipient:** D3 check: resubmission within 5 days → T1: Retrieve Reimbursement Request; no resubmission → T8: Record Request Outcome as closed incomplete
- **Complete when:** The record is saved, read back, and shows the deadline.

## 4. Planned Tools

### Tool 1

- **Tool name:** `send_fix_request`
- **Input:** Fix list; Request record; Fix request template
- **Output:** Fix request message
- **Implementation Route:** Web API calls (club email service send API)
- **Integration approach:** Direct integration
- **Role in this task:** Fills the template and sends one email per request ID and fix round.
- **Task timeout:** 30 seconds for sending and recording.
- **Maximum retries:** 1
- **Retry only when:** The email service explicitly rejects the message before accepting it (for example, a temporary rate limit) and the send log shows no accepted message for this request ID and round. Wait 5 seconds. Each request ID and round can have only one accepted message, so a retry cannot send twice. If the send times out or the response is unclear, do not retry.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record the send status as failed or uncertain and send the request to T7: Resolve Review Exception. Do not start the 5-day deadline for a message that may not have been delivered.

### Tool 2

- **Tool name:** `record_fix_request`
- **Input:** Request record
- **Output:** Fix request record
- **Implementation Route:** Database queries (write, as an upsert keyed on request ID and fix round)
- **Integration approach:** Direct integration
- **Role in this task:** Saves the fix request with its send status and deadline, then reads it back.
- **Task timeout:** Within the 30-second T4 limit.
- **Maximum retries:** 1
- **Retry only when:** The write fails with a temporary error and a read-back on the same key shows nothing saved. Because the write is an upsert on that key, it cannot create a second record.
- **On timeout, exhausted retries, or an error that cannot be retried:** Send the request to T7 with the status "fix request not recorded" so the treasurer can track the deadline manually.

### Tool 3

- **Tool name:** `check_resubmission_deadline`
- **Input:** Fix request record
- **Output:** Fix request record
- **Implementation Route:** Database queries (scheduled read when the 5-day deadline passes)
- **Integration approach:** Direct integration
- **Role in this task:** Checks whether a resubmission arrived before the deadline. If not, it marks the request "closed incomplete" for T8.
- **Task timeout:** 10 seconds per check.
- **Maximum retries:** 1
- **Retry only when:** A temporary database error occurs. Wait 1 minute. The read cannot create duplicates, and the "closed incomplete" status is written only once per request ID and round.
- **On timeout, exhausted retries, or an error that cannot be retried:** Send the request to T7 with the status "deadline check failed." Never close a request without confirming that no resubmission arrived.
