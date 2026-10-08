# Setup

`scripts/fetch-pbi.mjs` needs Node 18 or later, `git`, and a way to sign in to Azure DevOps. It has no npm dependencies.

## Sign-in

The script tries these in order:

1. **`AZURE_DEVOPS_EXT_PAT`**: a personal access token with the **Work Items (Read)** scope. The same variable the Azure CLI's DevOps extension uses.
2. **Azure CLI**: if you are signed in with `az login`, the script asks it for a Microsoft Entra token (`az account get-access-token --resource 499b84ac-1321-427f-aa17-267ca6975798`). No PAT to create or rotate, and it follows your organisation's sign-in policies. Preferred where it works.

If neither works the script stops with exit code 3. HTTP 203, 401 or 403 means the credentials were sent but not accepted: the PAT expired or lacks the scope, or the `az` account is not in the organisation's tenant (`az login --tenant <tenant>`).

If `az` does not answer within 30 seconds, the script stops with exit code 3 instead of waiting. This happens when the Azure CLI wants an interactive sign-in, for example when an earlier session has expired. Run `az login` yourself and try again.

## Network

Every request to Azure DevOps times out after 30 seconds, and the script stops with exit code 3 and "could not reach". On a company network this usually means a proxy. Node's `fetch` ignores `HTTPS_PROXY` unless `NODE_USE_ENV_PROXY=1` is also set (Node 24 or later):

```sh
NODE_USE_ENV_PROXY=1 HTTPS_PROXY=http://proxy.example.com:8080 node scripts/fetch-pbi.mjs
```

## Organisation

Read from the `origin` remote. These forms are recognised:

| Remote | Organisation URL |
| --- | --- |
| `https://dev.azure.com/{org}/{project}/_git/{repo}` | `https://dev.azure.com/{org}` |
| `https://{user}@dev.azure.com/{org}/...` | `https://dev.azure.com/{org}` |
| `git@ssh.dev.azure.com:v3/{org}/{project}/{repo}` | `https://dev.azure.com/{org}` |
| `https://{org}.visualstudio.com/...` | `https://{org}.visualstudio.com` |

For code hosted elsewhere (GitHub, Azure DevOps Server on-premises), set `AZURE_DEVOPS_ORG_URL`, for example `https://dev.azure.com/contoso` or `https://tfs.example.com/DefaultCollection`.

## Branch names

By default the id is the first number of three or more digits that stands between separators (`/`, `-`, `_`, `#`, or the start or end of the name):

| Branch | Id |
| --- | --- |
| `feature/12345-login` | 12345 |
| `users/matt/12345` | 12345 |
| `bugfix/AB#12345` | 12345 |
| `pbi-12345_fix` | 12345 |
| `release/2026.10` | none |

For another convention, set `M7_ADO_BRANCH_PATTERN` to a regular expression whose first capture group is the id, for example `^PBI(\d+)` for `PBI12345-login`.

## Task or backlog item

The number in the branch may point at a task or at a backlog item. The script walks up the parent links: as long as the parent is not a portfolio item (by default **Epic** or **Feature**), the item is treated as part of that parent. So:

| Branch item | Parent | Treated as |
| --- | --- | --- |
| Task | Product Backlog Item | part of the PBI |
| Bug | User Story | part of the story |
| Product Backlog Item | Feature | the backlog item itself |
| Bug | Feature, or none | the backlog item itself |
| Task | none | the backlog item itself |

If your process uses other portfolio levels (for example *Initiative*), set `M7_ADO_PORTFOLIO_TYPES=Epic,Feature,Initiative`.

## Environment variables

| Variable | Purpose |
| --- | --- |
| `AZURE_DEVOPS_EXT_PAT` | Personal access token (Work Items: Read). |
| `AZURE_DEVOPS_ORG_URL` | Organisation URL when the remote is not on Azure DevOps. |
| `M7_ADO_BRANCH_PATTERN` | Regex with one capture group for the id. |
| `M7_ADO_PORTFOLIO_TYPES` | Comma-separated work item types above backlog items. Default `Epic,Feature`. |
| `HTTPS_PROXY` + `NODE_USE_ENV_PROXY=1` | Send requests through a proxy (Node 24+). |

## What is fetched

- The branch item and, if it is part of one, its backlog item: all fields and relations. The text fields read are Description, Repro steps, Acceptance criteria and System info.
- Every child of the backlog item: title, state, whether it is assigned to you, someone else or nobody, and a short description.
- The parents above the backlog item, and items linked as **Related** to the branch item or the backlog item: title, state and shortened text fields.

**Comments are never fetched.** The Discussion of a work item stays out of the skill entirely.

Text is converted from HTML to plain text. To keep the context small, long fields are truncated, and the output says so. Run with `--full` to turn truncation off.

## Personal information

Names and other personal details of colleagues never leave the script. Before printing anything, it collects every identity the API returned (Assigned To, Created By, Changed By and any other identity field) and then:

- **Never prints identity fields.** Assignment becomes *assigned to you*, *assigned to someone else* or *unassigned*. *You* is the signed-in user, found through `_apis/connectionData`.
- **Labels people.** `@mentions` in descriptions become stable labels: *Person A*, *Person B*, *You*. A mention of someone the script has not seen becomes `@[someone]`. Build and service accounts become *a service account*.
- **Scrubs free text.** Every name it has seen is replaced in titles, descriptions, tags and the branch name: the full display name, the reordered form ("Vries, Pieter de" as "Pieter de Vries"), each part of three letters or more except particles like *van* and *de*, and the account name. E-mail addresses become `[e-mail]` and phone numbers `[phone]`.
- **Shortens the branch name** to the part from the id on, because branches like `users/jdoe/12345` contain a user name.

Limits: a name typed in free text that belongs to no identity the API returned (someone outside the team, a nickname) cannot be recognised and stays in. The skill tells the agent not to repeat such names and to tell you, so the scrubbing can be extended. Scrubbing errs towards removing: a word that happens to equal someone's first or last name is replaced too.
