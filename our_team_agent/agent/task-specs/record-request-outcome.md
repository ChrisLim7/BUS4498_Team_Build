# Record Request Outcome Task Specification

## Basic Information

- **Task ID:** T8
- **Task name:** Record Request Outcome
- **Task type:** Remember
- **Task owner:** PayBack workflow controller; the club treasurer reviews the outcome log

## 1. Task Description

This task writes one final record per request when it is approved, rejected, or closed as incomplete. Each record shows whether the treasurer returned the request at least once, how many fix rounds it took, and the receipt fingerprint. The workflow needs this task for two reasons:

- The log is the request history T1 loads for future duplicate checks.
- It lets the club calculate the share of requests the treasurer returns at least once, the charter's measure for the 35% → 10% goal.

The task uses fixed rules only.

## 2. Inputs

### Input 1

- **Input name:** Final outcome
- **Contents and format:** Request ID and final status (approved, rejected, or closed incomplete), with the deciding person or rule and the timestamp.
- **Source:** T6: Approve Reimbursement Request, T7: Resolve Review Exception, or T4: Request Missing Information (closed incomplete)

### Input 2

- **Input name:** Request record
- **Contents and format:** Request ID, member, club, vendor, purchase date, total, receipt fingerprint, fix rounds, and treasurer returns.
- **Source:** T1: Retrieve Reimbursement Request

- **If a required input is missing or invalid:** If the final status or request ID is missing, do not write a record. Send the case to T7: Resolve Review Exception.

## 3. Outputs

### Output 1

- **Output name:** Outcome log entry
- **Contents and format:** One row per request with request ID, member, club, vendor, purchase date, total, receipt fingerprint, final status, treasurer returned (yes or no), number of fix rounds, and closing date.
- **Next task or recipient:** Request history used by T1 in future runs, and the club treasurer's return-rate report
- **Complete when:** The row is saved and read back. The run is then complete.

## 4. Planned Tools

### Tool 1

- **Tool name:** `record_request_outcome`
- **Input:** Final outcome; Request record
- **Output:** Outcome log entry
- **Implementation Route:** Database queries (write, as an upsert keyed on request ID)
- **Integration approach:** Direct integration
- **Role in this task:** Saves the final outcome row and reads it back.
- **Task timeout:** 15 seconds per run.
- **Maximum retries:** 1
- **Retry only when:** The write fails with a temporary error and a read-back on the same request ID shows no saved row. Wait 2 seconds. Because the write is an upsert on request ID, a retry cannot create a duplicate row.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record "outcome not saved" and send the case to T7: Resolve Review Exception. The run is not complete until the outcome is saved.
