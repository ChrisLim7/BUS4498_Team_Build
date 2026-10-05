# Prepare Treasurer Review Packet Task Specification

## Basic Information

- **Task ID:** T5
- **Task name:** Prepare Treasurer Review Packet
- **Task type:** Act
- **Task owner:** PayBack workflow controller

## 1. Task Description

This task assembles one review packet for each request that is ready for approval, using a fixed layout: a one-line summary, the request details, the receipt file, the rule-by-rule results with evidence, and the member's fix history. It saves the packet to the treasurer's review queue and sends the treasurer one notification. The workflow needs this task so the treasurer can decide from one verified page instead of rechecking the receipt and rules by hand. It does not recommend approval or rejection beyond reporting T3's or T7's result.

## 2. Inputs

### Input 1

- **Input name:** Review result
- **Contents and format:** T3's outbound deliverable with status "Ready," or a T7 decision marked "Ready for approval," including the evidence summary.
- **Source:** T3: Review Request Against Funding Rules or T7: Resolve Review Exception

### Input 2

- **Input name:** Request record
- **Contents and format:** All form fields, receipt file links, and the fix round number.
- **Source:** T1: Retrieve Reimbursement Request

### Input 3

- **Input name:** Receipt details
- **Contents and format:** The extracted vendor, date, line items, and total.
- **Source:** T2: Extract Receipt Details

- **If a required input is missing or invalid:** If the review result is not marked ready or any input is missing, do not create a packet. Send the request to T7: Resolve Review Exception.

## 3. Outputs

### Output 1

- **Output name:** Treasurer review packet
- **Contents and format:** One page in the treasurer's review queue with the summary, request details, receipt file, rule-by-rule results, and fix history.
- **Next task or recipient:** T6: Approve Reimbursement Request
- **Complete when:** The packet is saved, read back, visible in the review queue, and the treasurer notification is sent.

## 4. Planned Tools

### Tool 1

- **Tool name:** `build_review_packet`
- **Input:** Review result; Request record; Receipt details
- **Output:** Treasurer review packet
- **Implementation Route:** Functions/scripts (fixed packet layout), plus a database write keyed on request ID and fix round
- **Integration approach:** Direct integration
- **Role in this task:** Fills the packet layout, saves it to the review queue, and reads it back.
- **Task timeout:** 30 seconds for the whole T5 run, including all tools and retries.
- **Maximum retries:** 1
- **Retry only when:** The save fails with a temporary error and a read-back on the same key shows no saved packet. Because the save is keyed on request ID and round, a retry cannot create a second packet.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record "packet not saved" and send the request to T7: Resolve Review Exception.

### Tool 2

- **Tool name:** `notify_treasurer`
- **Input:** Treasurer review packet
- **Output:** Treasurer review packet
- **Implementation Route:** Web API calls (club email service send API)
- **Integration approach:** Direct integration
- **Role in this task:** Sends the treasurer one email with a link to the packet.
- **Task timeout:** Within the 30-second T5 limit.
- **Maximum retries:** 1
- **Retry only when:** The email service explicitly rejects the message before accepting it and the send log shows no accepted notification for this request ID and round. Wait 5 seconds. If the outcome is uncertain, do not resend.
- **On timeout, exhausted retries, or an error that cannot be retried:** Keep the packet in the review queue, record the notification status as failed or uncertain, and send the case to T7 so the treasurer still learns about it.
