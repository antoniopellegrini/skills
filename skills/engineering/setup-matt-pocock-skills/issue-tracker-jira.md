# Issue tracker: Jira

Issues and specs for this repo live as Jira Cloud work items. Use Atlassian's official [`acli`](https://developer.atlassian.com/cloud/acli/guides/install-acli/) CLI for all operations.

## Conventions

- **Configuration**: site `<SITE>.atlassian.net`, project key `<PROJECT>` (the key, e.g. `PROJ`, not the project name), work item type `<TYPE>`, subtask type `<SUBTASK_TYPE>`, Done status `<DONE_STATUS>`. Type and status names are per-project and localised, so use these exact strings.
- **Rich text**: formatted descriptions and comments are raw ADF documents (`{"version":1,"type":"doc","content":[...]}`), never stringified or wrapped in a text node. Plain text also works.
- **Long bodies**: descriptions are capped at 32767 characters. Keep the head in the description and put each remaining part in a comment whose first line is `[continued n/N]`.
- **Create an issue**: `acli jira workitem create --project <PROJECT> --type '<TYPE>' --summary '...' --description-file <FILE> [-l <LABEL>] --json`. Note `create` takes `-l`/`--label` while `edit` takes `--labels`.
- **Read an issue**: `acli jira workitem view <KEY> --fields 'key,summary,description,status,assignee,labels,issuelinks' --json`. Request only the fields needed; add `comment` when the conversation matters. `view` takes the key as an argument, not `--key`.
- **List issues**: `acli jira workitem search --jql '<JQL>' --fields 'key,summary,status,labels' --limit 25 --json`. The result is a bare JSON array, each item with `key` beside `fields`. `search` only serves `key`, `summary`, `description`, `status`, `labels` and `assignee`, and prints a plain-text error for any other field; read `parent`, `issuelinks`, `resolution` and `comment` with `view`. JQL itself can filter on any field: `parent = <KEY> AND resolution IS EMPTY` is valid. Filter on `resolution` or `statusCategory` rather than status names, which are localised.
- **Comment on an issue**: `acli jira workitem comment create --key <KEY> --body-file <FILE> --json`. The `create` is required: `comment --key` fails with `unknown flag`.
- **Apply / remove labels**: `acli jira workitem edit --key <KEY> --labels '<LABEL>' --yes --json` / `--remove-labels '<LABEL>'`.
- **Close**: post the explanation as a comment, then `acli jira workitem transition --key <KEY> --status '<DONE_STATUS>' --yes --json`. A status name with no allowed transition still exits 0, printing `{"status": "FAILURE", ...}`, so re-read `status` to confirm. `transition` cannot list the allowed statuses; use the configured name.
- **Auth**: `failed to fetch work item details` usually means an unauthenticated session, not a bad key. Check with `acli jira auth status`; re-authenticate with `acli jira auth login --web`. ACLI keeps its OAuth token in the OS credential store, which sandboxed harnesses (e.g. Codex) can't reach: there, request host access for the `acli jira` prefix from the first command instead of retrying inside the sandbox.
- **Gaps**: when ACLI lacks an operation, say so and use the Jira Cloud REST API v3 for that one operation only.

## When a skill says "publish to the issue tracker"

Create a Jira work item in `<PROJECT>`.

## When a skill says "fetch the relevant ticket"

Run `acli jira workitem view <KEY> --fields '<FIELDS_NEEDED>' --json`; include `comment` only when relevant.

## Wayfinding operations

Used by `/wayfinder`. The **map** is a single work item with **child** work items as tickets.

- **Map**: a work item labelled `wayfinder:map`, holding the Notes / Decisions-so-far / Fog description.
- **Child ticket**: created with `--parent <MAP_KEY> --type '<SUBTASK_TYPE>'` and labels `wayfinder:<type>` (`research`/`prototype`/`grilling`/`task`). Once claimed, the ticket is assigned to the driving dev.
- **Blocking**: Jira's native `Blocks` link, the canonical, UI-visible representation. The flag form and the JSON form run in opposite directions. Flags: `acli jira workitem link create --out <BLOCKER> --in <BLOCKED> --type 'Blocks' --yes`. For several edges, write `[{"inwardIssue":"<BLOCKER>","outwardIssue":"<BLOCKED>","type":"Blocks"}, ...]` to a file (schema from `link create --generate-json`) and apply it once with `link create --from-json <FILE> --yes`. The success line of `--from-json` prints the reverse of the edge that landed, and `link create` ignores `--json`, so confirm each edge from the blocked item's `view --fields issuelinks`: its blocker appears under `inwardIssue` (`type.inward` = "is blocked by"). Fix a reversed edge with `link delete --id '<ID>' --yes`, one ID per call (a comma-separated list deletes nothing), and re-create it.
- **Frontier query**: `search --jql 'parent = <MAP_KEY> AND resolution IS EMPTY ORDER BY created ASC'`, then `view --fields 'key,status,assignee,issuelinks'` each candidate in turn. Drop assigned tickets and tickets with an unresolved blocker under `inwardIssue` (entries under `outwardIssue` are downstream, not blockers); first in map order wins.
- **Claim**: `acli jira workitem assign --key <KEY> --assignee '@me' --yes --json`, the session's first write. Re-read `assignee` and stop if it is not the authenticated account.
- **Resolve**: comment the answer (the resolution is the last comment not starting with `[continued n/N]`), transition to `<DONE_STATUS>` and confirm `status`, then re-read the map and append a context pointer (gist + link) to its Decisions-so-far.
