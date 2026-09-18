# SOCRadar connector for Okta Workflows

**Connector version: 0.7.0**

SOCRadar finds credentials that infostealer malware has stolen and published in botnet logs.
This connector brings that data into Okta Workflows. A flow can match the exposed credentials to
Okta users, act on them, and then close the SOCRadar alarm.

| Card | Description |
|---|---|
| Get Botnet Data | Retrieves credentials exposed through infostealer malware and botnet logs from SOCRadar for a date window. |
| Close Alarm | Changes the status of a SOCRadar alarm and optionally records a comment. |

Both cards call only the SOCRadar platform API. Neither card writes to Okta.

## Authorization

The connector sends a SOCRadar API key in the `API-Key` header. The connection also stores your
SOCRadar Company ID, and every request reads the data of that company.

### Prerequisites

In SOCRadar, open **Settings > API & Integrations > API Options > Company API Key** and copy two
values:

- **API key.** Treat it as a password.
- **Company ID.** It also appears in the platform URL after `/company/`.

### Create a connection

1. Add a SOCRadar card to a flow.
2. Click the connection field and choose **New Connection**.
3. Enter a name, the **API key** and the **Company ID**. Save.
4. Okta does not check the key on save. Run one Get Botnet Data call over a narrow window to test
   it.

### Scopes

None. The connector uses an API key.

## Action cards

Both cards return two output groups: **Response** (`Status Code`, `Body`) and **Result**
(`is_success`, `message`, `response_code`).

**Check `is_success`, not `Status Code`.** SOCRadar returns HTTP 200 even when it rejects a
request. A rejection shows as `is_success` `false`, with the reason in `message`.

### Get Botnet Data

Retrieves credentials exposed through infostealer malware and botnet logs from SOCRadar for a
date window.

**Options**

None.

**Inputs**

Group: **Query**

| Label | Definition | Type | Required |
|---|---|---|---|
| `startDate` | Start of the window, included. Format `YYYY-MM-DD HH:mm:ss` | Text | Yes |
| `endDate` | End of the window, not included. Same format | Text | Yes |
| `page` | Page number, starting at 1 | Text | Yes |
| `limit` | Records per page, from 1 to 5000 | Text | Yes |

**Outputs**

Group: **Response**

| Label | Definition | Type |
|---|---|---|
| `Status Code` | HTTP status of the call | Number |
| `Body` | The parsed response body | Object |

Group: **Result**

| Label | Definition | Type |
|---|---|---|
| `is_success` | `false` when SOCRadar rejected the request | True/False |
| `message` | SOCRadar's message, such as the reason for a rejection | Text |
| `response_code` | The code in the body. It can differ from `Status Code` | Number |

The records are at `data.data` in `Body`. The total for the whole window is at
`data.total_data_count`. To reach the list, chain two **Object Get** cards that each read `data`.

**Limitations and known issues**

- A `limit` above 5000, or a date that cannot be read, is rejected with HTTP 200 and `is_success`
  `false`.
- An empty required input fails the card with HTTP `400` before any request is sent.
- A `page` of 0 returns page 1 again. A window that ends before it starts returns zero records.
- One Object Get on `data` returns an object, not the list. A For Each over it runs once, and the
  run still ends green.
- An empty page does not mean the window is done. It is done when the records you fetched reach
  `total_data_count`.
- Records come newest first. When you page across runs, fix `startDate` and `endDate` once for
  the whole crawl. Otherwise records are skipped or read twice.
- Records can contain passwords in clear text.

### Close Alarm

Changes the status of a SOCRadar alarm and optionally records a comment. A flow usually calls it
after the affected account has been remediated.

**Options**

None.

**Inputs**

Group: **Body**

| Label | Definition | Type | Required |
|---|---|---|---|
| `alarm_id` | The alarm to change | Number | Yes |
| `status` | The target status code, listed below | Number | Yes |
| `comments` | A note stored on the alarm | Text | No |

Status codes: 0 OPEN, 1 INVESTIGATING, 2 RESOLVED, 4 PENDING_INFO, 5 LEGAL_REVIEW,
6 VENDOR_ASSESSMENT, 9 FALSE_POSITIVE, 10 DUPLICATE, 11 PROCESSED_INTERNALLY, 12 MITIGATED,
13 NOT_APPLICABLE.

**Outputs**

The same **Response** and **Result** groups as Get Botnet Data.

**Limitations and known issues**

- An unknown `alarm_id` or an invalid `status` is rejected with HTTP 200 and `is_success`
  `false`. Nothing changes.
- Closing an alarm that already has the target status also returns `is_success` `false`. One
  alarm can cover several people, so a flow that closes it once per person sees this routinely.
- A closed alarm leaves the feed and shifts the pages. Close alarms after a crawl has finished.

## Performance

- Six months of data for an active company can exceed 100,000 records. Start with a narrow
  window.
- Fetch one page per scheduled run and keep the page number and the window in a Workflows table.
  Each run stays short, and a failed run resumes where it stopped.
- Workflows Tables are rate limited. Store only the records you acted on. Stored records can hold
  passwords in clear text.
- Both cards count against your SOCRadar API quota.

## Flow template

SOCRadar also provides a flow template built on these cards. It adds the safety checks: dry run,
a verified domain list, protected accounts and a cap on suspensions per run. It uses Okta's own
connector with the `okta.users.read` and `okta.users.manage` scopes. It skips a user whose
password changed after the leak, or who is already suspended, deprovisioned or deactivated. A
`LOCKED_OUT` user is not skipped, because a lockout is often the attacker trying the leaked
password. A flow
you build yourself from the cards has none of these checks.

Run the template with dry run on first and read the **SOCRadar Audit** table. The cards in the
template show only `Status Code` and `Body`. Read `is_success` there with an **Object Get** card.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| The run is green but no records were processed | SOCRadar rejected the request. Check `is_success` |
| The card fails with `401` | The API key is wrong or was revoked. Edit the connection |
| The card fails with `404` | The connection has no Company ID |
| A card shows "Unable to connect" | The connector has no connection test, so this label does not prove a fault |

## Related

The [SOCRadar API reference](https://github.com/orcunsami/socradar-api-docs) is the OpenAPI file
of the SOCRadar REST API. Its README lists the two paths this connector calls.

## Support

integration@socradar.io
