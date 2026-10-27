---
name: apply-recommendations
description: Apply Pinnacles SEO recommendations from the pinnacles MCP server in the user's own site repository, then report what was actually applied. Use when the pinnacles MCP server is connected and the user asks to apply, review, or check on SEO recommendations.
---

# Apply Pinnacles recommendations

## Before you start

- Confirm the `pinnacles` MCP server is connected: call `list_sites`. If it returns no sites, tell the user and stop.
- Work only in the repository the user points you at.
- Never write to the live site through Pinnacles. It cannot. You edit the repository; the user deploys.
- If a call fails with 401 or 403, the connection's token is missing, unrecognized, revoked, or the plan does not include MCP. Relay the error text to the user: the fix is a new connection in Pinnacles Settings under Agent access (MCP) and a replaced Authorization header in the client config, or a plan upgrade. Reconnecting the server does not help. Stop until the user has done it.

## The loop

1. Call `get_recommendations` once. It returns a compact row for every open recommendation (`action_id`, type, status, title, page `url`, priority, `off_site`, and the change `targets`), with no paging. It does not carry the changes themselves.
2. For each recommendation, read `off_site` first. If it is true, list the recommendation for the user and do not attempt it. Otherwise call `get_recommendation_detail` with its `action_id` to get the changes. Read each change's `operation` (`set`, `append`, `remove`, or `procedure`) and `guidance`. If `guidance` is `null`, show the recommendation to the user rather than guessing. If `value_inline` is `false`, call `get_drafted_content` for the value.
3. Find the file in the repository that renders the page at `url` (the framework's page, template or layout, the robots or sitemap source, the redirect config). Tell the user which file and why.
4. Make the edit. Our `suggested_value` is a default you may adapt to fit the page; keep the point of the change (the `why` and `issue_codes`).
5. Build the site (or run the user's check command) before reporting anything. If the build fails, fix or revert. Never report an unbuilt change.
6. Call `mark_applied` with what you actually wrote: `applied` keyed by the change `target`s you touched. `set` and `append` entries are `{ "applied_value": "<the exact text now in the file>" }`; `remove` and `procedure` entries are `{ "applied": true }` with an optional `note`. Never include a `verified` key. For `page_create`, pass `published_url`, the URL the page will live at on the site's own domain, and only after the user confirms it is deployed.
7. If you decide not to apply a recommendation, call `dismiss_recommendation` with a reason a person could act on ("page is intentionally noindexed", "this URL is a vendor-hosted form"). Never skip silently, and never dismiss without telling the user.

## Partial applies are legitimate

If you applied some of a recommendation's changes, report exactly those targets. Do not retry the rest in a loop; say what is left and why.

## Verification: the quiet period is site-wide

Pinnacles re-checks live pages every two hours, and does not start until two hours have passed since the most recent recommendation was marked done on the site, by you or in the dashboard. A batch is therefore checked about two hours after the last call, not the first.

Do not poll `verification_status` after a batch. Tell the user to ask again later (at least two hours after the last `mark_applied`, typically two to four), then read it once, passing the `action_id`s you marked applied as `actions`. That returns one full entry per id: the verdict, `gate`, `checks`, and `observations`. Called without `actions`, the tool returns a compact entry (verdict and `observations`, no `gate` or `checks`) for every recommendation completed since the site's last audit plus anything still `pending`, and `since` tells you the cut-off; use that when you do not have the ids, then call again with `actions` for the entries you need to act on.

- `pending` means not checked yet.
- `not_detected` means an issue is still on the page, or Pinnacles could not load it. Read `gate` (which issue codes failed — populated for metadata fixes), `checks.failed` (which verifier step failed, as Pinnacles' internal step names — the detail for every other recommendation type), and `observations` (issues the applied value introduced) from the full entry, fix those in the repository, and tell the user to click "Mark as not done" in the dashboard before you call `mark_applied` again.
- `skipped` is final: Pinnacles cannot check that kind of change.
- `verified` may still carry `observations`. Fix them on the next pass.

## Never

- Never commit, push, or deploy without the user.
- Never invent a `published_url`.
- Never dismiss on the user's behalf for taste.
- Never mark applied a change you did not build.
