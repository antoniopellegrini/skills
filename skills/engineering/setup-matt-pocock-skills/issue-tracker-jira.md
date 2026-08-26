# Issue tracker: Jira

Issues and PRDs for this repo live as Jira Cloud work items. Use Atlassian's official [`acli`](https://developer.atlassian.com/cloud/acli/guides/install-acli/) CLI for all operations.

## Conventions

- **Configuration**: site `<site>.atlassian.net`, project `<PROJECT>`, subtask type `<SUBTASK_TYPE>`, and resolved status `<DONE_STATUS>`. Read the project's actual values from Jira; status and type names are localised.
- **Rich text**: When writing formatted Jira descriptions or comments, use raw ADF documents (`{"version":1,"type":"doc","content":[...]}`), never stringified or wrapped in a text node.
- **Long bodies**: descriptions are capped at 32767 characters. Keep the head in the description and put each remaining part in a comment whose first line is `[continued n/N]`.
- **Create an issue**: `acli jira workitem create --project <PROJECT> --type '<TYPE>' --summary '...' --description-file <FILE> [-l <LABEL>] --json`. `create` takes `-l`/`--label`; `edit` takes `--labels`.
- **Read an issue**: `acli jira workitem view <KEY> --fields 'key,summary,status,assignee,labels,issuelinks' --json`. Always request only the fields needed.
- **List issues**: `acli jira workitem search --jql '<JQL>' --fields 'key,summary' --limit 25 --json`. Bound the result. `search` serves `key`, `summary`, `description`, `status`, `labels` and `assignee`; read `parent`, `issuelinks`, `resolution` and `comment` with `view`. A rejected field prints a plain-text error, not JSON. JQL clauses take any field — `parent = <KEY> AND resolution IS EMPTY` is valid.
- **Comment on an issue**: `acli jira workitem comment create --key <KEY> --body-file <FILE> --json`.
- **Apply / remove labels**: `acli jira workitem edit --key <KEY> --labels '<LABEL>' --yes --json` / `--remove-labels '<LABEL>'`.
- **Assign**: `acli jira workitem assign --key <KEY> --assignee '@me' --yes --json`.
- **Close**: `acli jira workitem transition --key <KEY> --status '<DONE_STATUS>' --yes --json`.
- **Unsupported operations**: When ACLI lacks a required operation or Jira rejects a field, report the exact limitation and use Jira Cloud REST API v3 for that narrow operation, disclosing the fallback.

In a managed filesystem/process sandbox, run `acli jira ...` commands with narrowly scoped host access from the first command because ACLI OAuth credentials may live in an inaccessible OS credential store. Reuse the approved `["acli", "jira"]` prefix. A `failed to fetch work item details` error indicates an unauthenticated session rather than a bad key; confirm with `acli jira auth status` and re-authenticate with `acli jira auth login --web`.

## Output shape

`--json` output differs from the REST envelope.

- `workitem search` returns an object whose keys are indices (`{"0": {...}, "1": {...}}`); read its values rather than an `issues` array. Treat a zero result count as a parsing mismatch until the raw top-level keys confirm it.
- Each item carries its `key` alongside `fields`, not inside it.
- `fields.description` is an ADF document node.
- `fields.comment` is `{comments: [{author, body}]}`, each `body` an ADF document.

## When a skill says "publish to the issue tracker"

Create a Jira work item in the configured project.

## When a skill says "fetch the relevant ticket"

Run `acli jira workitem view <KEY> --fields '<FIELDS_NEEDED>' --json`; fetch comments only when relevant.

## Wayfinding operations

Used by `/wayfinder`. The **map** is a single work item with **child** work items as tickets.

- **Map**: a work item labelled `wayfinder:map`, holding the Notes / Decisions-so-far / Fog description.
- **Child ticket**: create with `--parent <MAP_KEY>` and the configured subtask type. Labels: `wayfinder:<type>` (`research`/`prototype`/`grilling`/`task`). Once claimed, assign it to the driving dev.
- **Blocking**: Jira's native issue link — **inward blocks outward**. For a single edge where A blocks B, run `acli jira workitem link create --in <A> --out <B> --type 'Blocks' --yes`. For several edges, write them as JSON (`[{"inwardIssue":"<A>","outwardIssue":"<B>","type":"Blocks"}]`, schema from `link create --generate-json`) and apply them in one call with `link create --from-json <FILE> --yes`; the command confirms in plain text and ignores `--json`, so a file is more reliable than looping the flag form through a parser. Confirm every edge from `view --fields 'key,parent,labels,issuelinks' --json`, reading `type.inward` and `type.outward`. Correct a reversed edge with `link delete --id '<ID>[,<ID>]' --yes`, then re-apply it.
- **Frontier query**: search `parent = <MAP_KEY> AND resolution IS EMPTY ORDER BY created ASC` with fields `key,summary`, `--limit 25`, and `--json`; inspect each candidate in turn with `view --fields 'key,status,assignee,issuelinks' --json`. Drop assigned tickets and tickets with unresolved inward `Blocks` links; outward links are downstream. First in map order wins. Stop when enough actionable candidates are found.
- **Claim**: `acli jira workitem assign --key <KEY> --assignee '@me' --yes --json` — the session's first write. Re-read `key,assignee` and stop if it does not match the authenticated account.
- **Resolve**: add the answer with `acli jira workitem comment create` — the resolution is the last comment whose first line is not `[continued n/N]` — transition to the configured Done-category status, verify `status,resolution`, then re-read the latest map description before appending the context pointer to Decisions-so-far.
