# Retrieve Reimbursement Request Task Specification

## Basic Information

- **Task ID:** T1
- **Task name:** Retrieve Reimbursement Request
- **Task type:** Retrieve
- **Task owner:** PayBack workflow controller

## 1. Task Description

When a member submits or resubmits a request, this task collects everything the review needs: the form and receipt files, the current club funding rules, and the club's past requests. It then checks with fixed rules that every required form field is filled in and at least one receipt file in PDF, JPG, or PNG format is attached. The workflow needs this task so that later tasks work from one complete, current record, and so that missing basics go back to the member before any review starts. It does not judge whether the request follows the rules.

## 2. Inputs

### Input 1

- **Input name:** Reimbursement request form
- **Contents and format:** Structured form submission with request ID, fix round number, member name, Cal Poly email, club name, amount requested, purchase date, purchase description, event name, selected expense category, optional pre-approval reference, and receipt files (PDF, JPG, or PNG). The form has no fields for bank or payment details.
- **Source:** Club member, through the PayBack request form

### Input 2

- **Input name:** Club funding rules
- **Contents and format:** The club's current funding rules document (Markdown or PDF) with its version date.
- **Source:** Club treasurer, who maintains the rules document

### Input 3

- **Input name:** Request history
- **Contents and format:** Table of the club's past requests with request ID, member, vendor, purchase date, total, receipt file fingerprint, and final outcome.
- **Source:** T8: Record Request Outcome (outcome log)

- **If a required input is missing or invalid:** A missing form field or receipt is not an error; it is recorded in the intake status and sent through the D1 check to T4: Request Missing Information. If the rules document or request history cannot be read, the request goes to T7: Resolve Review Exception, and the run does not continue as if the data were complete.

## 3. Outputs

### Output 1

- **Output name:** Request record
- **Contents and format:** One structured record with all form fields, receipt file links, and the fix round number.
- **Next task or recipient:** T2: Extract Receipt Details and T3: Review Request Against Funding Rules; also used by T4 and T5
- **Complete when:** Every form field and receipt link from the submission appears in the record.

### Output 2

- **Output name:** Club funding rules and request history
- **Contents and format:** The rules document with its version date, and the club's request history table.
- **Next task or recipient:** T3: Review Request Against Funding Rules
- **Complete when:** Both sources have been read in full and the rules version is recorded.

### Output 3

- **Output name:** Intake status
- **Contents and format:** Complete or incomplete, with a list of missing fields or files.
- **Next task or recipient:** D1 check: complete → T2: Extract Receipt Details; incomplete → T4: Request Missing Information
- **Complete when:** The status is recorded for this request and round.

## 4. Planned Tools

### Tool 1

- **Tool name:** `retrieve_request_submission`
- **Input:** Reimbursement request form
- **Output:** Request record
- **Implementation Route:** Database queries (read-only on form submissions and receipt file storage)
- **Integration approach:** Direct integration
- **Role in this task:** Reads one submission and its receipt files by request ID and builds the request record.
- **Task timeout:** 60 seconds for the whole T1 run, including all tools and retries.
- **Maximum retries:** 1
- **Retry only when:** A temporary connection error or timeout occurs. Wait 5 seconds before retrying. The tool only reads, so a retry cannot change or duplicate anything.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record "intake failed" with the error and send the request to T7: Resolve Review Exception.

### Tool 2

- **Tool name:** `load_funding_rules`
- **Input:** Club funding rules
- **Output:** Club funding rules and request history
- **Implementation Route:** File operations (read-only)
- **Integration approach:** Direct integration
- **Role in this task:** Loads the current rules document and records its version date.
- **Task timeout:** Within the 60-second T1 limit.
- **Maximum retries:** 1
- **Retry only when:** The file cannot be read because of a temporary error. Wait 5 seconds. Read-only, so no duplicates.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record "rules unavailable" and send the request to T7. Never review a request against outdated or guessed rules.

### Tool 3

- **Tool name:** `retrieve_request_history`
- **Input:** Request history
- **Output:** Club funding rules and request history
- **Implementation Route:** Database queries (read-only on the outcome log)
- **Integration approach:** Direct integration
- **Role in this task:** Reads the club's past requests so T3 can check for duplicates.
- **Task timeout:** Within the 60-second T1 limit.
- **Maximum retries:** 1
- **Retry only when:** A temporary connection error or timeout occurs. Wait 5 seconds. Read-only, so no duplicates.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record "history unavailable" and send the request to T7. An unreadable history is never treated as "no past requests."

### Tool 4

- **Tool name:** `check_required_fields`
- **Input:** Request record
- **Output:** Intake status
- **Implementation Route:** Functions/scripts (deterministic field and file-type checks)
- **Integration approach:** Direct integration
- **Role in this task:** Confirms that every required field is filled in and a receipt file in an accepted format is attached, and lists anything missing.
- **Task timeout:** Within the 60-second T1 limit.
- **Maximum retries:** 0
- **Retry only when:** Not applicable.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record "intake check failed" and send the request to T7. Do not mark the request complete without the check.
