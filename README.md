# Docs as Code Template

A starting point for HMCTS teams who want a documentation site built with the
[GOV.UK Tech Docs Template](https://github.com/alphagov/tech-docs-gem)
([guidance](https://alphagov.github.io/tdt-documentation/)), published to GitHub Pages
and readable only by people signed in to the HMCTS GitHub organisation.

Write Markdown, raise a PR, merge — the site rebuilds and deploys itself.

## What you get

- **The GOV.UK Tech Docs look** — header carrying your service name and phase badge,
  site search, and a collapsible sidebar built from your folder structure.
- **Internal-only publishing** — an internal repository plus GitHub Pages private
  publishing means readers must be signed in to GitHub and a member of the org.
- **Deployment on merge** — push to `main` and the site rebuilds; no manual step.
- **PR builds** — every pull request is built, so a broken page is caught before merge.
- **A link checker** — html-proofer, wired to your repository automatically.
- **Zero-install preview** — open the repo in GitHub Codespaces and the preview
  server starts on its own.
- **A `docs-site` agent skill** — for GitHub Copilot and Claude Code. It configures a
  new site for you, and from then on adds pages with the right frontmatter, in the
  right folder, linked from the sidebar.

## Using this template

> **You cannot apply a template to a repository that already exists.** GitHub only
> offers it at creation time. If you have already made an empty repo, delete it and
> start at step 1 — otherwise you will end up copying files in by hand.

### 1. Create your repository from this one

Go to **[hmcts/docs-as-code-template](https://github.com/hmcts/docs-as-code-template)**
and click the green **Use this template** button at the top right, then
**Create a new repository**.

On the form that opens:

| Field | What to set |
|---|---|
| Owner | `hmcts` |
| Repository name | e.g. `platform-ai-gateway-docs` |
| Visibility | **Internal** — a public repo cannot have an access-controlled site |

Click **Create repository**, then clone it locally.

### 2. Configure it

In **GitHub Copilot Chat** (agent mode) or **Claude Code**, invoke the `docs-site`
skill, which was copied into your repo along with everything else:

```
/docs-site set up this docs site
```

It sets the site title from your repository name, asks you two questions, makes all the
file changes, and hands back a short list of GitHub settings to apply. Prefer to do it
yourself? **[SETUP.md](SETUP.md)** has the same steps by hand.

### 3. Turn on publishing

Whichever route you took, this part is done in the browser and cannot be automated:

**Settings → Pages** → Source: **GitHub Actions**, Visibility: **Private**.

Visibility is the one that matters — it is what makes the site readable only by
signed-in org members. Without it your documentation is on the public internet.

Then commit, push, and the site deploys itself.

## Repository layout

```
├─ source/                            ← all site content
│  ├─ index.html.md.erb               ← home page
│  ├─ example-section/                ← a section: an index plus its pages
│  │  ├─ index.html.md.erb
│  │  └─ example-page.html.md.erb
│  ├─ assets/images/                  ← images used in your pages
│  ├─ images/                         ← favicons (the layout expects them here)
│  └─ javascripts/, stylesheets/      ← asset entry points, leave as they are
├─ config/tech-docs.yml               ← site configuration (fill this in)
├─ config.rb                          ← Middleman configuration
├─ Rakefile                           ← build and link-check tasks
├─ Gemfile                            ← Ruby dependencies
├─ .devcontainer.json                 ← Codespaces preview
├─ .github/workflows/                 ← PR build and Pages deploy
├─ .github/page-template.md           ← canonical page structure (outside source/, so it never publishes)
├─ .github/skills/docs-site/          ← agent skill (GitHub Copilot)
└─ .claude/skills/docs-site/          ← agent skill (Claude Code)
```

## Writing content

Pages are Markdown with ERB, named `.html.md.erb`, and every page needs frontmatter:

```yaml
---
title: Error codes
weight: 20
last_reviewed_on: 2026-07-28
review_in: 6 months
---
```

`weight` orders pages in the sidebar within their folder — use gaps of 10 so you can
insert a page later without renumbering. A section's `index.html.md.erb` uses
`weight: 1` and its pages start at `10`, so an index never ties with a page in the
same folder. `last_reviewed_on` and `review_in` drive the review reminder shown on
the page.

A **section** is a folder under `source/` containing an `index.html.md.erb` that lists
its pages. The sidebar is built from that structure.

Link to built pages, not source files: `[Error codes](error-codes.html)`.

### Adding a page with an agent

In GitHub Copilot Chat (agent mode) or Claude Code:

```
/docs-site add a page about our deployment pipeline
```

The skill interviews you, creates the file from `.github/page-template.md`,
works out the right `weight`, updates the section index, and reports anything left
as a placeholder. Both copies of the skill are the same file — if you change one,
change the other.

## Local development

**Prerequisites:** Ruby 3.3.4 (`.ruby-version` works with rbenv or rvm)

```bash
bundle install
bundle exec middleman server     # http://localhost:4567, rebuilds as you save
```

Or use `./run.sh`. To build and check links as CI does:

```bash
bundle exec middleman build      # output in build/, gitignored
bundle exec rake check_urls
```

The stylesheet and JavaScript pipeline needs **Node** alongside Ruby. Without it the
HTML still builds but every asset errors and the site renders unstyled — Codespaces
installs Node for you, and GitHub's runners already have it.

### Dependency versions

`Gemfile.lock` is **committed**, so CI, Codespaces and every engineer build with
identical gem versions. Upgrade deliberately:

```bash
bundle update govuk_tech_docs
```

Then check the rendered site before merging — major versions of `govuk_tech_docs`
have changed the header layout and colours, so an upgrade is a visual change as well
as a dependency one.

## Keeping up with the template

`.template_version` records the version of this template your site started from. If
the template gains something you want later, compare against that tag rather than
diffing the whole repository.
