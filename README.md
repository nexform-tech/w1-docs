# w1-docs

Product documentation for the **W1 wheeled mobile robot** (`NEXFORM W1`).

The manuals in this repository are the source of truth for the W1 documentation.
The public documentation centre re-publishes a copy of every manual to Feishu
whenever the bytes here change, so pushing a new PDF to `main` is the whole
release procedure for a document — there is nothing to build and nothing to
upload by hand.

> **Status:** the product manual has landed in both languages. The published
> site, the SDK guides and the rest of the documentation are still to come.

## Contents

| Path | Role |
| --- | --- |
| `NEXFORM_W1产品说明书.pdf` | Product manual, Simplified Chinese, 19 pages |
| `W1-Product-Manual.pdf` | Product manual, English, 20 pages |
| `.github/workflows/ci.yml` | Pull-request gate: Markdown lint, repository-standards checks |
| `.github/workflows/release.yml` | semantic-release on every merge to `main` |
| `.github/workflows/notify-docs.yml` | Asks the documentation centre to pick up a changed manual |
| `.nexform-standards.json` | Which plugin generated this repository and which CI template it uses |
| `.releaserc.json`, `.markdownlint.json` | Release and Markdown-lint configuration |
| `LICENSE` | Apache License 2.0 |

The PDFs are the deliverable. Their LaTeX/Markdown sources are authored outside
this repository and are not tracked here yet; `.gitignore` already reserves the
usual site output directories (`site/`, `_build/`, `build/`, `dist/`, `public/`)
for when a documentation toolchain lands.

Both editions carry the same content: specifications, interfaces, operation and
maintenance. Keep them in step — the English page of the documentation centre
links its own PDF, and a reader who finds only one language updated will file a
bug.

## How a manual reaches the documentation centre

The target is the `nexform-tech/nexform-guide` repository, which owns the page
Markdown, the `files/` copies and the Feishu publisher. Nothing in this
repository talks to Feishu directly.

1. **Notify.** A push to `main` that touches `*.pdf`, `*.PDF`, `*.STEP` or
   `*.step` starts `notify-docs.yml`. It mints a short-lived installation token
   for the GitHub App `nexform-docs-dispatch` (App ID `5608132`) and sends a
   `docs-updated` `repository_dispatch` event to `nexform-tech/nexform-guide`.
2. **Sync.** `nexform-guide`'s `Sync files` workflow runs `scripts/sync_files.py`,
   which reads `files.yaml` and downloads every listed file from the repository
   that owns it. A file whose bytes differ from the copy in `files/` is
   rewritten; identical files are left alone, so a run with no source change
   commits nothing. A changed copy is committed to `main` as
   `chore: sync files from source repositories`, and the workflow dispatches
   `Publish` (a `GITHUB_TOKEN` push starts no workflow on its own).
3. **Publish.** The `Publish` workflow runs `scripts/publish.py`, which walks
   every file block on the page, downloads the block's current upload, and
   compares its SHA-256 with the copy in `files/`. Differing bytes are
   re-uploaded and the block's `replace_file` is patched in place; identical
   bytes are skipped. The workflow then re-reads the document and verifies that
   no change is pending.

**Timing.** With the dispatch token in place, a push to `main` reaches Feishu in
a minute or two. `nexform-guide` also runs the sync nightly at `03:00 UTC`, so
without the token a change still lands, at worst a day later.

### The `files.yaml` entry for each manual

```yaml
  NEXFORM_W1产品说明书.pdf:
    repo: nexform-tech/w1-docs
    path: NEXFORM_W1产品说明书.pdf
```

- The key is the name of the copy under `files/` and must equal the link text on
  the page (`📎 [NEXFORM_W1产品说明书.pdf](files/NEXFORM_W1产品说明书.pdf)`), byte
  for byte, Chinese characters included.
- `repo` and `path` say where the bytes come from. `path` may differ from the
  key when the English page needs a translated file name — `W1-Product-Manual.pdf`
  happens to be the same in both places.
- A key must be a plain file name: no `/`, no `\`, not `.` and not `..`.
- Both manuals are listed, and each one appears exactly once. A manifest entry
  with no link on any page is dead weight; a link with no entry fails the
  publisher with "add it to files.yaml and run scripts/sync_files.py".

### App credentials

`notify-docs.yml` is inert until the repository has both secrets:

| Secret | Value |
| --- | --- |
| `NEXFORM_APP_ID` | `5608132` |
| `NEXFORM_APP_PRIVATE_KEY` | Private key of the `nexform-docs-dispatch` App |

`NEXFORM_APP_ID` is set on this repository. The private key is stored as an
organisation-level secret shared with `litearm-docs` and `litegrip-docs`; an
organisation administrator has to add `w1-docs` to the repositories that secret
is visible to. Until then the workflow prints a notice and exits successfully
rather than failing on every documentation push, and the nightly sync covers the
change.

## Replacing or adding a manual

**Replacing a manual in place** — the common case — needs one commit:

1. `git switch -c docs/<short-description>`; never commit on `main`.
2. Replace the PDF, keeping its file name. The name is the join key between this
   repository, `files.yaml` and the page, so renaming a PDF is a different
   operation (below), not a side effect of updating it.
3. Open a PR, let CI go green, squash-merge.

**Adding a manual, or renaming one**, also needs a pull request against
`nexform-tech/nexform-guide` in the same round, changing `files.yaml` and every
page that links the file (`pages/w1.md`, `pages/en-w1.md`, and the same pair under
`pages/` for other products). Until that PR lands the publisher keeps serving the
old file, because the manifest still points at the old name.

## Verifying a change end to end

`gh` is the only tool needed. Run from anywhere with `GODEBUG=http2client=0` if
the connection stalls on HTTP/2.

```bash
# 1. the dispatch fired in this repository
gh run list --repo nexform-tech/w1-docs --workflow notify-docs.yml --limit 3

# 2. nexform-guide copied the new bytes; the log ends "<n> updated, <m> up to date"
gh run list --repo nexform-tech/nexform-guide --workflow sync-files.yml --limit 3
gh run view <run-id> --repo nexform-tech/nexform-guide --log | grep "updated,"

# 3. the publisher replaced the block and verified the document
gh run list --repo nexform-tech/nexform-guide --workflow publish.yml --limit 3
gh run view <run-id> --repo nexform-tech/nexform-guide --log | grep -E "w1:"
```

A healthy publish of the W1 page prints, per page key:

```text
w1: {"ensure_file": 1, "keep": 8}
w1: 0 media file(s) updated
w1: 0 block(s) centered
w1: verified, document matches the page
```

`1 media file(s) updated` on the run that carried the new PDF is the line that
says the Feishu block now holds the new bytes. `ensure_file` in the summary is
normal, not a defect: the planner lists the file block on every run, and
`_replace_file` downloads the block's current upload, compares SHA-256 and
uploads only when the bytes differ. That comparison is the same one that picks
up a new PDF.

## Working in this repository

These match the organisation's shared standards (`.nexform-standards.json`):

- **Never commit or push on `main`.** Branch first: `git switch -c <type>/<short-description>`.
- **Conventional Commits:** `<type>: <subject>`, lowercase imperative, no full
  stop, at most 72 characters. `feat` bumps the minor version, `fix` and `perf`
  bump the patch, `BREAKING CHANGE:` in the footer bumps the major, everything
  else releases nothing. The version prefix is what semantic-release keys on —
  a typo publishes the wrong version.
- **One concern per commit.** A documentation change and a workflow change are
  two commits.
- **PRs carry Summary / Changes / Testing / Issues**, and the PR title equals the
  commit subject so a squash merge reads correctly either way.
- **Squash merge only**, and CI has to be green first. Delete the branch after.
- **Never tag or release by hand.** `release.yml` and semantic-release own the
  tags, the version and the changelog.
- **No secrets, credentials or build output in a commit.** `*.pdf` is the one
  binary this repository is meant to carry.

Before pushing, run the same checks CI runs:

```bash
npx --yes markdownlint-cli2@0.23.3 "README.md"
test -s README.md && test -s LICENSE
python3 -c "import json; json.load(open('.releaserc.json')); json.load(open('.markdownlint.json'))"
```

The repository is documentation-only today, so `ci.yml` has no test suite to
run. When source code lands, replace the standards step with the real toolchain
and test command — and do not delete the Markdown lint while doing it.

## Related repositories

| Repository | Role |
| --- | --- |
| [w1-python](https://github.com/nexform-tech/w1-python) | Python SDK |
| [w1-cpp](https://github.com/nexform-tech/w1-cpp) | C++ SDK |
| [w1-ros2](https://github.com/nexform-tech/w1-ros2) | ROS 2 driver |
| [w1-ros1](https://github.com/nexform-tech/w1-ros1) | ROS 1 driver |
| [nexform-guide](https://github.com/nexform-tech/nexform-guide) | Documentation centre: pages, `files/` copies, Feishu publisher |

The same organisation publishes the LiteArm manipulator series (`litearm-*`)
and the LiteGrip gripper (`litegrip-*`). See the
[nexform-tech organisation](https://github.com/nexform-tech) for the full list.

## License

Copyright © 2026 NEXFORM ROBOTICS. Licensed under the
[Apache License 2.0](LICENSE).
