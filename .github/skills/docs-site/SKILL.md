---
name: docs-site
description: "Set up and maintain a documentation site created from the docs-as-code template. Use when asked to: set up the docs site, configure the template, add a page, create a page, document something, add a section. Handles first-time setup (config, GitHub Pages, SSO verification, clearing example content) and ongoing page creation."
---

# Docs Site Skill

One skill, two jobs. Work out which is needed before doing anything.

## Step 0 — Which job is this?

Read `github_repo` from `config/tech-docs.yml` and compare it with the actual
repository (`gh repo view --json nameWithOwner --jq .nameWithOwner`).

- **They differ, or `github_repo` is still `hmcts/docs-as-code-template`** → the site
  has never been configured. Run **Part A — First-time setup**, then offer Part B.
- **They match** → the site is set up. Run **Part B — Add a page**.

If the repository *is* `hmcts/docs-as-code-template`, stop: that is the template
itself, which is not a published site and must not be configured as one.

---
---

# Part A — First-time setup

Get a freshly created repository to a published, SSO-only site. Most values come from
the repository itself; only three need a human.

## A1 — Derive what you can

Do not ask for any of this:

| Value | Command |
|---|---|
| `<org>/<repo>` | `gh repo view --json nameWithOwner --jq .nameWithOwner` |
| default branch | `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name` |
| repo visibility | `gh repo view --json visibility --jq .visibility` |

If `gh auth status` fails, say which steps you cannot run and give the manual
equivalent rather than failing part-way through.

If visibility is not `INTERNAL` or `PRIVATE`, warn plainly: a public repository cannot
have an access-controlled Pages site, so the documentation would be world-readable.

## A2 — Ask three questions

Propose a default for each so the user can simply accept it.

| Question | Default to propose |
|---|---|
| **Service name** — shown in the header bar | Repo name in title case: `platform-ai-gateway-docs` → "Platform AI Gateway Docs" |
| **Phase** — `Alpha`, `Beta` or `Live` | `Live` |
| **Slack channel** — where a reader asks for help | `platops-build-notices` |

## A3 — Write `config/tech-docs.yml`

```yaml
host: https://<org>.github.io/<repo>          # no trailing slash
service_name: <answer>
phase: <answer>
header_links:
  GitHub: https://github.com/<org>/<repo>
github_repo: <org>/<repo>
github_branch: <default branch>
default_owner_slack: <answer>
```

Leave every other key as it is. The `Rakefile` reads `github_repo` from this file, so
the link checker configures itself.

## A4 — Align the workflows

Both trigger on `main`. If the default branch differs, change it in
`.github/workflows/build.yaml` and `deploy.yaml` — otherwise nothing ever deploys and
there is no error explaining why.

Then delete the template guard from the `deploy` job in `deploy.yaml`. It is always
true outside the template repository, so it does nothing there but confuse a reader:

```yaml
    if: github.repository != 'hmcts/docs-as-code-template'
```

Leave `build.yaml` alone; it has no guard.

## A5 — Enable GitHub Pages

```bash
gh api -X POST repos/<org>/<repo>/pages -f build_type=workflow
```

A `409` means Pages is already enabled — fine, carry on. This sets the source to
GitHub Actions. It does **not** control who can read the site.

## A6 — STOP. The visibility step is manual

**GitHub exposes no API for Pages visibility.** It must be set in the browser, and it
is the step that decides whether the site is internal or world-readable.

Tell the user:

> Open **Settings → Pages → Visibility** and set it to **Private**, then tell me when
> that is done. It is the only step I cannot do for you, and the one that makes the
> site SSO-only.

Wait. Do not continue assuming it has been done.

## A7 — Verify it is actually private

Never take "done" as evidence:

```bash
gh api repos/<org>/<repo>/pages --jq .public        # must print: false
```

If the site has deployed, check from outside as well:

```bash
curl -s -o /dev/null -w '%{http_code}' https://<org>.github.io/<repo>
```

`301`/`302` to `github.com/login` means protected. `200` serving content means it is
**public** — send them back to A6 and do not report success. If `.public` is `true`,
say plainly that the site is currently readable by anyone with the link.

## A8 — Clear out the example content

- Rewrite `source/index.html.md.erb` from a one-line description, keeping the
  frontmatter shape (`title`, `weight: 1`, `last_reviewed_on` today, `review_in`)
- Rename `source/example-section/` to a real section and rewrite its `index.html.md.erb`
- Delete `source/example-section/example-page.html.md.erb`
- Rewrite `README.md` to describe this site rather than the template
- **Keep** `source/templates/page-template.html.md.erb` — Part B reads it
- **Keep** `source/images/` (favicons) and `source/javascripts/`, `source/stylesheets/`
  — the layout requires these and the build breaks without them

If they have no content yet, leave one example section so the sidebar is not empty, and
say that you have.

## A9 — Report

Run `bundle exec middleman build` to confirm it still builds, then report:

| Item | State |
|---|---|
| `config/tech-docs.yml` | configured for `<org>/<repo>` |
| Workflows | trigger on `<branch>`, template guard removed |
| Pages | enabled, source: GitHub Actions |
| **Visibility** | **private — verified `.public == false`** |
| Example content | removed / retained |
| Build | passes |

Then offer to add their first real page.

---
---

# Part B — Add a page

## B1 — Gather information

Do not proceed until **title** and **section** are known. Ask the rest one group at a
time; accept "unknown" or "TBC" as placeholders.

| Field | Notes |
|---|---|
| **Page title** | Sentence case, e.g. `Deployment pipeline`, `Error codes` |
| **Section** | An existing folder under `source/`, or a new one |
| **Short description** | One sentence — used in the section index table |
| **Your name / team** | Used in the Change history row |

List directories under `source/`, ignoring `assets/`, `images/`, `javascripts/`,
`stylesheets/` and `templates/`, to find the available sections.

> **If the section does not exist:** create the folder, add an `index.html.md.erb`
> modelled on an existing section index, and add a row for it to the table in
> `source/index.html.md.erb`.

Also ask: what a reader needs to know before the detail, the substance of the page,
related pages worth linking, and who owns it.

**Weight (sidebar order).** Do not ask. Instead:

1. Read every `.html.md.erb` in the target folder **except `index.html.md.erb`**
2. Find the highest `weight:`
3. Use that value + 10

A new folder, or one holding only an index, starts at `weight: 10`.

> **Why the index is excluded.** A section index's `weight` orders that *section*
> among its siblings, while page weights order pages *within* the section — but both
> live in the same folder. Counting the index would eventually hand a page the same
> weight as it, making the sidebar order unstable. Section indexes use `1`; pages
> start at `10`.

## B2 — Derive the filename

Lowercase kebab-case, then `.html.md.erb`:

- `Deployment pipeline` → `deployment-pipeline.html.md.erb`
- `Error codes` → `error-codes.html.md.erb`
- `How to get a token` → `how-to-get-a-token.html.md.erb`

## B3 — Create the file

Read `source/templates/page-template.html.md.erb` and use it as the structure — **do
not paste a copy from memory**; it is the site's canonical shape and teams change it.

Fill in `title`, the computed `weight`, `last_reviewed_on` (today, `YYYY-MM-DD`), and
keep the template's `review_in` unless told otherwise. Leave `\<placeholder\>` values
only where the user genuinely did not know, and drop any template section that has
nothing to say rather than leaving it empty.

## B4 — Update the section index

Add a row to the pages table in `source/<section>/index.html.md.erb`:

```
| [<Page title>](<filename-without-.erb>) | <short description> |
```

For example:

```
| [Error codes](error-codes.html) | What each gateway error means and how to resolve it |
```

The link uses the built `.html` name, not the `.html.md.erb` source filename.

## B5 — Review the result

Read the file back and check it against the page template. Report each section as:

- ✅ **Complete** — real content, no placeholders
- ⚠️ **Placeholder** — present but still has `\<placeholder\>` values; say which
- ❌ **Missing** — absent and should not be

Then:

| Check | Why |
|---|---|
| Frontmatter has all four keys | A missing `title` breaks the sidebar and the `<h1>` |
| `weight` does not clash with a sibling | Two equal weights give an unstable sidebar order |
| Internal links use `.html`, not `.html.md.erb` | Source filenames 404 on the built site |
| Images have real alt text | The link checker allows missing alt text, so nothing else catches it |
| The section index has a row for the new page | Otherwise the page is only reachable by URL |

List anything ⚠️ or ❌ and ask whether to fill the gaps now or leave them for the owner.

## B6 — Verify placement

- [ ] File exists at `source/<section>/<filename>`
- [ ] Section index links to it
- [ ] If the section is new, `source/index.html.md.erb` links to the section
- [ ] `bundle exec middleman build` succeeds

---

## Notes

- `source/templates/page-template.html.md.erb` is the canonical page structure. If the
  shape of pages changes, change it there — Part B follows it.
- `weight:` orders pages within a folder. Gaps of 10 leave room to insert later.
- Building locally needs **Node** alongside Ruby. Without it the HTML still builds but
  every stylesheet errors and the site renders unstyled — which looks like success.
  Codespaces installs it; GitHub's runners already have it.
- `Gemfile.lock` is committed deliberately, so every build uses identical gem versions.
  Do not delete it or add it to `.gitignore`.
