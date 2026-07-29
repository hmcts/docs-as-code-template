# Setup

What to do in the repository you have just created from this template.

**Most of this is automated.** In GitHub Copilot Chat (agent mode) or Claude Code:

```
/docs-site set up this docs site
```

The skill makes all the file changes — steps 2, 3 and 7 — asks you three questions, and
derives the rest from `git`. It then gives you a short checklist of the GitHub settings
it cannot change by editing files, which is **step 1** below.

**What the skill needs:** an agent that can run commands, and `git` (which every clone
has). If `gh` is also installed and authenticated it will offer to enable Pages and
check the visibility for you; without it, those simply stay on the closing checklist.
Ruby and Node are only needed for local preview — not for using the skill.

The rest of this page is the same work by hand. Steps 1 and 2 are the ones that
matter: until both are done your site either will not publish or will be readable by
anyone with the link.

Expect about fifteen minutes by hand, or a couple of minutes with the skill.

---

## 1. Make the site internal only

**This is the step that controls who can read your documentation.** GitHub Pages is
public by default, even from a private repository.

1. Confirm the repository visibility is **Internal** — Settings → General → Danger
   Zone → Change repository visibility.
2. Settings → **Pages**:
   - **Source:** GitHub Actions
   - **Visibility:** **Private**

With visibility set to Private, anyone opening the site is redirected to GitHub
sign-in and must be a member of the organisation with access to the repository.

> `prevent_indexing` in `config/tech-docs.yml` only asks search engines not to index
> the site. It is not access control. Set the Pages visibility.

**Check it worked.** From a logged-out browser, or:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://<org>.github.io/<repo-name>
```

A `301`/`302` to `github.com/login` means it is protected. A `200` serving your
content means the site is public — go back to Settings → Pages.

---

## 2. Fill in the site configuration

Edit `config/tech-docs.yml`. Every key in this table still points at the template
repository, so all six need a new value — the site builds happily either way, which is
what makes this easy to skip.

| Key | What to set it to |
|---|---|
| `host` | `https://<org>.github.io/<repo-name>` — no trailing slash |
| `service_name` | The name shown in the header bar |
| `header_links` | Point the GitHub link at your repository |
| `github_repo` | `<org>/<repo-name>` — also used by the link checker |
| `github_branch` | Your default branch, usually `main` |
| `default_owner_slack` | The channel a reader should ask in |

`phase` sets the badge next to the service name — `Alpha`, `Beta` or `Live`.

Nothing else needs editing: the `Rakefile` reads `github_repo` from this file, so the
link checker configures itself.

**Check it worked.** Once deployed, the "Report an issue with this page" link at the
bottom of any page should open *your* repository, not the template.

---

## 3. Check the workflows match your default branch

`.github/workflows/build.yaml` and `deploy.yaml` both trigger on `main`. If your
default branch is named something else, change it in both files — otherwise nothing
ever deploys and there is no error to tell you why.

---

## 4. Make the first deployment

Push to your default branch, or run **Deploy static content to Pages** manually from
the Actions tab. When it finishes, the environment URL appears on the workflow run
and under Settings → Pages.

---

## 5. Optional: enable link checking properly

The build workflow runs `rake check_urls` with `continue-on-error: true`, so broken
links are reported but do not fail the build.

To make it enforce:

1. Create a repository or organisation secret named `GH_TOKEN` with read access —
   without it, links to internal HMCTS repositories look like 404s.
2. Remove the `continue-on-error: true` line from `.github/workflows/build.yaml`.

Leave it non-blocking while you are drafting; turn it on once the site is stable.

---

## 6. Optional: protect the default branch

Settings → Branches → add a rule for your default branch:

- Require a pull request before merging
- Require the **Build and test** status check to pass

This is what stops a broken page reaching the published site.

---

## 7. Clean up the template content

- Replace `source/index.html.md.erb` with your own home page
- Rename `source/example-section/` to a real section and rewrite its `index.html.md.erb`
- Delete `source/example-section/example-page.html.md.erb` once you have real pages
- Keep `.github/page-template.md` — the `docs-site` skill reads it,
  and it is where you change the shape of your pages
- Update this repository's `README.md` to describe *your* site rather than the template

---

## Troubleshooting

**The deploy workflow succeeds but the site 404s.** Settings → Pages → Source must be
**GitHub Actions**, not "Deploy from a branch".

**Sidebar order looks random.** Two pages in the same folder share a `weight:`. Give
each a distinct value, in steps of 10.

**A page is not in the sidebar.** Its folder needs an `index.html.md.erb`, and the
page needs `title:` and `weight:` in its frontmatter.

**Links work locally but 404 when published.** You linked the source filename. Link
the built page: `[Error codes](error-codes.html)`, not `error-codes.html.md.erb`.

**`bundle install` fails.** Check your Ruby version matches `.ruby-version` (3.3.4),
or open the repository in Codespaces and skip local setup entirely.

**`Could not find a JavaScript runtime`.** The stylesheet and JavaScript pipeline needs
Node installed alongside Ruby. The HTML still builds, so it is easy to miss — the
symptom is an unstyled site with `error build/stylesheets/screen.css` in the output.
Install Node, or use Codespaces, where `.devcontainer.json` installs it for you.
GitHub's `ubuntu-latest` runners already have it, so CI is unaffected.
