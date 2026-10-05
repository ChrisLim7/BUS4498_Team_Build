# Extract Receipt Details Task Specification

## Basic Information

- **Task ID:** T2
- **Task name:** Extract Receipt Details
- **Task type:** Sense
- **Task owner:** PayBack workflow controller

## 1. Task Description

This task reads each receipt image or PDF with a vision-capable language model and turns it into structured details: vendor, purchase date, line items, subtotal, tax, total, and whether the receipt is itemized. Each field gets a high or low confidence level, and fields the model cannot read are listed instead of guessed. Any payment card digits on the receipt are ignored and not stored. The workflow needs this task because T3 cannot check the rules until the receipt's contents are available as data. A blurry or unreadable receipt is not a failure; it is reported so that T3 can ask for a clearer copy.

## 2. Inputs

### Input 1

- **Input name:** Request record
- **Contents and format:** Request ID, fix round, and links to the receipt files (PDF, JPG, or PNG).
- **Source:** T1: Retrieve Reimbursement Request

- **If a required input is missing or invalid:** If a receipt file link is broken or the file cannot be opened, record "receipt file unavailable" and send the request to T7: Resolve Review Exception.

## 3. Outputs

### Output 1

- **Output name:** Receipt details
- **Contents and format:** Structured record per receipt with vendor, purchase date, line items (description, quantity, and price), subtotal, tax, total, itemized yes or no, a confidence level for each field, and a list of unreadable fields.
- **Next task or recipient:** T3: Review Request Against Funding Rules
- **Complete when:** Every receipt file in the request has a record, and every field is either filled in with a confidence level or listed as unreadable.

## 4. Planned Tools

### Tool 1

- **Tool name:** `extract_receipt_details`
- **Input:** Request record
- **Output:** Receipt details
- **Implementation Route:** Web API calls (vision-capable language model through the Anthropic API)
- **Integration approach:** Direct integration
- **Role in this task:** Sends each receipt image to the model with a fixed extraction prompt and returns the structured fields with confidence levels.
- **Task timeout:** 60 seconds for the whole T2 run, including all receipts and retries.
- **Maximum retries:** 1
- **Retry only when:** The API returns a temporary error, such as a rate limit or server error. Wait 5 seconds before retrying. The call only reads the receipt and changes no records, so a retry cannot create duplicates.
- **On timeout, exhausted retries, or an error that cannot be retried:** Record "extraction failed" with the error and send the request to T7: Resolve Review Exception. Do not pass partial or guessed receipt details to T3.
