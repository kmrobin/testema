---
name: refdemo
description: Replicate the RefDemo base Document Authoring site (aemxsc/refdemo-da) into the current site - all pages, fragments, sheets, assets, page images, structured content schemas, DA site configuration, editor mappings and Edge Delivery headers - with links rewritten to the new site. Use for "set up refdemo content", "replicate the refdemo site", "copy the base DA site", "copy refdemo content and config into my site", "bootstrap this demo site".
---

# RefDemo site replication

Copies the RefDemo base site into the user's current site in one pass, with at most one
question. The code base is https://github.com/AEMXSC/RefDemo-da; the content and
configuration come from Document Authoring `aemxsc/refdemo-da`
(https://da.live/#/aemxsc/refdemo-da).

All work is done by `scripts/replicate-refdemo.mjs` (next to this file). Run it from the
repository root. Do not copy files by hand, do not write your own copy loop, and do not
generate pages with an importer for this task.

## What is copied

| Item | Where it lives | How |
|---|---|---|
| Pages, nav, footer, fragments, data documents, block library docs | DA source | exact copy, links rewritten |
| Sheets (`config/placeholders`, `config/tags`, `config/mobile`, `docs/library/*.json`) | DA source | exact copy, links rewritten |
| Assets (`assets/**`: images, theme documents) | DA source | byte-for-byte |
| Images pasted into pages (hidden `.<page>/` folders, e.g. `.index/`, `.nav/`) | DA source | byte-for-byte |
| Structured content schemas (`.da/forms/schemas/*`) | DA source | exact copy |
| DA library settings (`.da/config.json`) | DA source | exact copy |
| DA site configuration (`data` and `library` sheets: AEM Assets repository, IMS org, editor paths, library entries, tools) | DA config API | merged into the target, links rewritten |
| Editor mappings for the base site (`editor.path` rows in the aemxsc org config: canvas for the site, form editor for `/fragments` and `/data`) | DA config API | rewritten, added to the target site config and appended to the target org config |
| Custom response headers (for example `access-control-allow-origin: *`) | Edge Delivery site config | merged into the target |

Never copied: `.trash/`, `.da/translation/` (the base site's in-flight translation jobs),
permission sheets, the rest of the aemxsc org configuration, and the Edge Delivery code or
content-source settings.

Link rewriting (in every HTML/JSON file and in the site configuration):
- `/aemxsc/refdemo-da/...` (content.da.live, da.live, editor paths) → `/<org>/<site>/...`
- `main--refdemo-da--aemxsc.aem.page|live` → `main--<site>--<org>.aem.page|live`, so library tools
  (Task Management, Content Assistant) open from the new site's own code
- `github.com/aemxsc/refdemo-da` → `github.com/<org>/<site>`
- **Kept on the base site:** `main--refdemo-da--aemxsc.aem.live/media_*` image URLs. Edge Delivery
  stores media per site, so these only resolve on the base site. They keep working as long as
  the base site stays published.

## Workflow

Run the steps in order without stopping between them. Ask nothing except the single
question in step 3, and only when that step says to.

### 1. Find the target site

1. If the user named a target DA site (for example `https://da.live/#/acme/demo` or `acme/demo`),
   use it.
2. Otherwise use the repository: `git remote get-url origin` → `github.com/<owner>/<repo>` →
   target `<owner>/<repo>`, lowercased.
3. Refuse if the target is `aemxsc/refdemo-da`. The base site is never written to.

Check that the site's code is RefDemo-based: `tools/content-assistant/` and
`tools/task-management/` should exist in the repository. If they are missing, say once that
library tools such as Content Assistant and Task Management won't load until the code is
based on AEMXSC/RefDemo-da, and continue.

### 2. Dry run (read-only)

```bash
node .claude/refdemo/scripts/replicate-refdemo.mjs --target <org>/<site> --dry-run --report migration-work/refdemo-replication/dry-run.json
```

The script first checks that the target's content source (from the site configuration) is
`https://content.da.live/<org>/<site>/`. If it stops with "Target content source is …", tell
the user the site isn't connected to its own Document Authoring folder, so it can't be
replicated into. An org admin has to fix the site configuration. Stop there. Use
`--force-target` only if the user explicitly asks for it.

Read the report: `files` (created / updated / unchanged), `conflicts` (target files that
exist and differ), `siteConfig`, `orgConfig.added` and `edsHeaders`.

### 3. Approval (at most one question)

Writing to Document Authoring and Edge Delivery configuration changes the user's remote
site, so it needs explicit approval.

- **No question** when the user's message already explicitly asks to copy, replicate or upload
  the RefDemo content into their DA site **and** the dry run shows no conflicts, or every
  conflict is one of the blank starter pages (`index.html`, `nav.html`, `footer.html`) of a
  new site. Go to step 4 with `--on-conflict overwrite`.
- **Otherwise** ask exactly one question. Include the counts from the dry run and list up to
  10 conflicting paths. Options:
  1. **Replicate and replace (Recommended)**. Description: "Copy all RefDemo content and
     configuration (N files, site configuration, editor mappings, custom headers) into DA site
     <org>/<site>, replacing the M existing files that differ. Nothing is previewed or
     published."
  2. **Replicate, keep existing**. Description: "Copy everything into DA site <org>/<site> but
     leave the M existing files that differ untouched. Nothing is previewed or published."
  3. **Cancel**. Description: "Change nothing."

  The chosen option approves the remote writes for the rest of this turn.

If a write is blocked by the platform's remote-content safety guard, do not retry it and do
not work around it. Tell the user the copy was blocked because their message didn't
explicitly ask for it (it's not a permission error), and give them the exact wording from
the guard's message to send.

### 4. Replicate

```bash
node .claude/refdemo/scripts/replicate-refdemo.mjs --target <org>/<site> --on-conflict <overwrite|skip>
```

Add `--preview` only if the user explicitly asked to preview the site in this request. Never
publish (go live) as part of this skill. Publishing is a separate, explicit request.

Before each upload, the script saves pages locally as `content/<path>.plain.html` and sheets
as `content/<path>.json`, so the local preview matches Document Authoring. It then uploads
from those files, reads every file back from the target and compares it. Timeout: allow up
to 10 minutes.

Exit codes: `0` success, `2` some files failed or didn't match on read-back (listed in
`failures` and `verify.mismatched`), `1` fatal error.

### 5. Handle partial failures

- **401 or 403 from admin.da.live or admin.hlx.page on everything**: credentials aren't reaching
  the services. Tell the user to open **Settings → LLM Permissions** and turn on "Allow LLM to
  use my Adobe credentials for admin.hlx.page and Document Authoring uploads". Never ask for a
  token in chat. After they confirm, rerun step 4. The script is idempotent: identical files
  are reported as `unchanged` and not uploaded again.
- **Some files failed**: rerun step 4 once. Report any that still fail.
- **`orgConfig.error`** (no CONFIG write on the org): not fatal. The editor mappings are already
  in the site configuration. Tell the user an org admin can add the rows listed in the report
  to the org configuration.
- **`edsHeaders.error`** (not a site admin): not fatal. Tell the user a site admin needs to add
  the listed headers.
- **Preview failures**: read `migration-work/preview-publish/last-job.json` before explaining.

### 6. Report to the user

Keep it short and plain. Don't mention scripts, file paths, localhost or tool names. Cover:

- what was copied: number of pages and documents, assets, schemas, and the configuration
  items, with any files skipped or failed
- which surfaces changed: **local workspace** (pages and sheets), **Document Authoring**
  (content, site configuration, org editor mappings if added), **Edge Delivery site
  configuration** (headers, if changed), **Edge Delivery preview** (only with `--preview`),
  **live** (never)
- that images published on the base site still load from it (see link rewriting above)
- that the AEM Assets repository and IMS org in the site configuration were copied as-is, so
  authors need access to that AEM author to use the asset picker
- next step: offer to preview the site if it wasn't previewed

## Options reference

| Option | Default | Purpose |
|---|---|---|
| `--target <org>/<site>` | required | Site to replicate into |
| `--source <org>/<site>` | `aemxsc/refdemo-da` | Base site |
| `--dry-run` | off | Read-only plan |
| `--on-conflict overwrite\|skip` | `overwrite` | Existing target files that differ |
| `--local-dir <dir>` | `content` | Local mirror of pages and sheets (`""` to disable) |
| `--no-org-config` | off | Don't append editor mappings to the target org config |
| `--no-eds-headers` | off | Don't merge Edge Delivery custom headers |
| `--preview` | off | Bulk-preview everything that was copied (explicit request only) |
| `--report <file>` | `migration-work/refdemo-replication/report.json` | Report location |
| `--concurrency <n>` | `4` | Parallel requests |
| `--force-target` | off | Skip the content-source check (explicit request only) |

Auth: on aemcoder, credentials are injected automatically once the user enables the opt-in.
Outside aemcoder, the script sends `DA_TOKEN` (IMS access token) from the environment, if set,
as a Bearer token. Never print or log it.

## Guarantees

- Read-only on the base site. Refuses to run with the base site as the target.
- Idempotent: rerunning uploads only what differs.
- Merges configuration and never replaces it: target-only rows, sheets and permissions are
  kept. The org configuration only ever gets `editor.path` rows appended.
- Every uploaded file is read back and compared.

## Limitations

- Sites whose content source is the Helix 6 source API (`https://api.aem.live/<org>/sites/<site>/source`)
  aren't supported. The script stops at the content-source check.
- Version history of the base site isn't copied, only current content.
- Hidden page-image folders are found by probing `.<page>` next to each page (DA hides
  dot-folders from listings). Images stored in other hidden folders aren't found.
