---
name: docs-site
description: "Set up and maintain a documentation site created from the docs-as-code template. Use when asked to: set up the docs site, configure the template, add a page, create a page, document something, add a section. Edits files only — first-time configuration and ongoing page creation — and hands back a short checklist of GitHub settings to apply by hand."
---

# Docs Site Skill

One skill, two jobs. Work out which is needed before doing anything.

## Step 0 — Which job is this?

Read `github_repo` from `config/tech-docs.yml` and compare it with the actual
repository:

```bash
git remote get-url origin | sed -E 's#^.*github\.com[:/]##; s#\.git$##'
```

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

**Shape of this run:** this skill edits files. That is all it does. It makes no GitHub
API calls, needs no credentials, and does not use `gh`. The two GitHub *settings* that
cannot be changed by editing files go on a checklist at the end (A6) — applying them by
hand takes under a minute.

### Do not improvise around missing tools

If a command is unavailable, **abandon that approach**. Do not:

- install anything — `brew`, `npm`, `apt` or any other package manager
- reach for alternative tooling — MCP servers, other CLIs, hand-rolled API calls
- search the environment, shell profiles or `.env` files for tokens or credentials
- retry a failed command with different authentication

Nothing here needs any of that. Ask the user, or put the item on the closing checklist.

## A1 — Confirm the repository details

Two values are needed. **Ask the user.** If `git` offers a sensible default, propose it
so they only have to confirm:

```bash
git remote get-url origin | sed -E 's#^.*github\.com[:/]##; s#\.git$##'   # <org>/<repo>
git symbolic-ref --short refs/remotes/origin/HEAD | sed 's#^origin/##'    # default branch
```

| Value | Example |
|---|---|
| `<org>/<repo>` | `hmcts/platform-ai-gateway-docs` |
| default branch | `main` |

These read local git config — no network, no credentials. If `git` is unavailable or
there is no remote, just ask. Do not infer either value any other way.

## A2 — Set the title, then ask two questions

**The service name is not a question.** Derive it from the repository name and apply it.
Leaving it as the template's name is the most visible way to get this wrong — it puts
"Docs as Code Template" in the header bar of someone else's site.

Split the repo name on hyphens, title-case each word, and correct known casings
(`AI`, `API`, `HMCTS`, `BCDR`, `CNP`, `PlatOps`):

```
platform-ai-gateway-docs  →  Platform AI Gateway Docs
platops-bcdr-runbooks     →  PlatOps BCDR Runbooks
```

Tell the user what you set and offer to change it. Do not ask first.

Then ask these two, proposing the default so they can simply accept:

| Question | Default to propose |
|---|---|
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

## A5 — Clear out the example content

- **Replace `source/index.html.md.erb` entirely.** The one shipped with the template is
  a setup checklist, not a home page — if it survives, the published site tells readers
  the site is unfinished. Write a real home page from a one-line description of what the
  site covers, and set `title:` to the service name from A2. Keep the frontmatter shape
  (`title`, `weight: 1`, `last_reviewed_on` today, `review_in`).
- Rename `source/example-section/` to a real section and rewrite its `index.html.md.erb`
- Delete `source/example-section/example-page.html.md.erb`
- Rewrite `README.md` to describe this site rather than the template
- **Keep** `.github/page-template.md` — Part B reads it. It lives outside `source/` deliberately: anything under `source/` is published as a page.
- **Keep** `source/images/` (favicons) and `source/javascripts/`, `source/stylesheets/`
  — the layout requires these and the build breaks without them

If they have no content yet, leave one example section so the sidebar is not empty, and
say that you have.

## A6 — Report, then hand over the manual tasks

First report what you changed:

| Item | State |
|---|---|
| `config/tech-docs.yml` | configured for `<org>/<repo>` |
| Workflows | trigger on `<branch>`, template guard removed |
| Example content | removed / retained |

Optionally confirm the build. This needs **Ruby and Node** locally, which many people
will not have — if `bundle` is missing, say so and move on. The PR build verifies it
anyway, and Codespaces has both:

```bash
bundle exec middleman build
```

### Manual tasks to complete

Always end with this checklist. These are GitHub *settings* and cannot be changed by
editing files. Do not attempt them yourself — hand them over.

> **1. Enable Pages.** Settings → Pages → Source: **GitHub Actions**
> **2. Check visibility.** Settings → Pages → Visibility should be **Private**. GitHub
>    usually sets this by default for internal repositories, so it may already be right.
> **3. Commit and push.** The site deploys on push to the default branch.
> **4. Confirm.** Signed out, opening the site should redirect you to GitHub sign-in.
>    If it serves the page instead, visibility is not set — go back to 2.

Step 4 is the one that matters: it is the only proof the documentation is not
world-readable. Say so, rather than listing it as an afterthought.

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
`stylesheets/`, to find the available sections.

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

Read `.github/page-template.md` and use it as the structure — **do
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

- `.github/page-template.md` is the canonical page structure. If the
  shape of pages changes, change it there — Part B follows it.
- `weight:` orders pages within a folder. Gaps of 10 leave room to insert later.
- Building locally needs **Node** alongside Ruby. Without it the HTML still builds but
  every stylesheet errors and the site renders unstyled — which looks like success.
  Codespaces installs it; GitHub's runners already have it.
- `Gemfile.lock` is committed deliberately, so every build uses identical gem versions.
  Do not delete it or add it to `.gitignore`.
