---
name: sitejabber-handle-privacy-requests
description: Fulfil a CCPA/GDPR data-subject access or deletion request for a customer's SmartCustomer data.
api: SmartCustomer (Sitejabber) Business API
base: https://api.smartcustomer.com/v1
operations:
- accessCustomerInformation
- removeCustomerInformation
generated: '2026-08-13'
method: generated
source: openapi/sitejabber-business-api-openapi.yml + https://api.sitejabber.com/
---

# Handle a data-subject request

SmartCustomer exposes data-subject access and deletion as API operations, shaped to the CCPA
statutory categories. This is a high-sensitivity flow — read the guardrails first.

## Guardrails

- **Never run either operation autonomously.** Both act on a named real person's personal data.
  Require an identified, verified data-subject request and explicit human authorization per call.
- **`removeCustomerInformation` is irreversible.** There is no undo, no soft delete and no
  confirmation step in the API itself. Your process is the only safeguard.
- Both calls only work if the customer's email was supplied to SmartCustomer by this business. A
  miss is not proof that no data exists elsewhere on the platform.
- Log which operator authorized each call and against which verified request.

## Access

`accessCustomerInformation` — `GET /businesses/{business}/privacy/access?email={email}`
(plus `client_token` in the query and the `user_token` header).

The response is keyed by CCPA category: Identifiers, Customer records information, Protected
classification info, Commercial info (including products or services purchased with website, title,
`orderId` and `orderDate`), Biometric info, Internet or electronic activity, Geolocation data,
sensory data, Professional or employment related, Education info, Inferences. Categories with
nothing on file come back as the string `"N/A"` — treat `"N/A"` as "no data", not as a value.

Return this to the data subject as-is. Do not summarize away categories that came back `N/A`; the
absence is part of the answer.

## Deletion

`removeCustomerInformation` — `POST /businesses/{business}/privacy/remove` with form field `email`.

A success is the bare envelope: `{"status":"OK","success":true}`. There is no report of what was
deleted, so **run the access call first and keep its output** as your record of what existed at the
moment of deletion.

## Rules

- Check the body, not the HTTP status: application errors are HTTP 200 with `success: false`.
- `302` means a required parameter was missing, `307` user not found, `308` insufficient permissions.
- Rate limits apply here as everywhere: 1000/hour per user token, max 10 per 10 seconds.
