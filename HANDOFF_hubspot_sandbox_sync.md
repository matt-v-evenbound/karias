# Handoff: finish the HubSpot sandbox schema sync

## Goal
Make the sandbox's object schemas match production. Only add what's missing or fix what differs. Don't touch records, emails or data.

- Production: portal **243261036**. **Read-only. Never send it a POST, PATCH, PUT or DELETE, not even a test.**
- Sandbox: portal **245802241**. This is where all writes go. Don't delete anything there: the client has custom apps built in it.

## Credentials (environment variables)
- `HUBSPOT_SANDBOX_SERVICE_KEY`: use for all sandbox writes. Its scopes were just updated to allow contact and company property writes. Send it as a plain Bearer token. Before writing, confirm it's on portal 245802241 with a GET, for example `GET /account-info/v3/details` (returns `portalId`).
- `HUBSPOT_PRODUCTION_PAT`: use only to read production if needed. It's a personal access key, so exchange it for a token first: `POST https://api.hubapi.com/localdevauth/v1/auth/refresh` with body `{"encodedOAuthRefreshToken": "<PAT>"}`. Use `oauthAccessToken` from the response, and check that `hubId` is 243261036. The exchange call changes nothing in the portal.
- `HUBSPOT_SANDBOX_PAT`: can read the sandbox. It can't write properties (it's missing `contacts-write` and `companies-write`), but it can create property groups.

## Done already
- Full comparison of production and sandbox (see `hubspot_prod_vs_sandbox_diff.md`).
- In the sandbox, created the 3 contact property groups: `communication_preference`, `fedlogic_member_outreach` and `_merge_review_row_`.
- Custom objects need nothing created. All 4 production objects (Audit Log Event, Plan Enrollment, Heymarket Message and Merge Review Row) exist in the sandbox, with every production property.

## Still to do (all sandbox, all in `hubspot_sandbox_payloads.json`)
1. **Create 37 contact properties**: `POST /crm/v3/properties/contacts`, one per entry in `create.contacts`.
2. **Create 2 company properties**: `POST /crm/v3/properties/companies` (`create.companies`: `cobra_administrator` and `group_code`).
3. **Update 7 contact properties**: `PATCH /crm/v3/properties/contacts/{name}` with each entry's `patch` (`update.contacts`). Changes: move 5 properties into the new groups, fix 1 label, and fix the option labels on 2 dropdowns. The option values already match. Only the labels differ.

The payloads are copied from production definitions, with only writable fields kept.

How to run it:
- Create one property first and check it with a GET. Then do the rest one at a time, logging the status of each.
- A 409 (already exists) is fine: skip it.
- On a 400, print the error and continue. On a 403, stop and report the scope HubSpot names.
- `ed_diversion_tier_calc` is a calculated property. The properties its formula uses (`ed_ind`, `ed_cnt`, `ed_avoidable_cnt`) already exist in the sandbox. If HubSpot rejects the formula through the API, report it so it can be created in the UI.
- Afterwards, read both portals again and confirm nothing in the list is still missing.

## Deliberately excluded
- `checkin_member_id` (production) already exists in the sandbox as `check_in_member_id`, with the same label. Don't create a duplicate.
- The 4 `hs_email_optout_*` properties come from email subscription types, so they can't be created through the properties API.
- These custom object settings differ but were probably changed on purpose for the sandbox apps: primary display property, required fields, the "Merge Review Row(s)" plural label and default group names. Leave them unless the user asks.
- The 4 sandbox-only custom objects (Deletion Event, Access Event, Export Event and Merge History Record). Leave them alone.

## Not compared yet (keys get 403)
Tickets, quotes, line items, products, leads and pipelines. Neither PAT has these scopes. If the sandbox service key (or a production read credential) can read them, compare their properties the same way and report the differences before creating anything.

## Report back to the user
List what was created, updated, skipped and failed, with the reason for each failure. Keep it short.
