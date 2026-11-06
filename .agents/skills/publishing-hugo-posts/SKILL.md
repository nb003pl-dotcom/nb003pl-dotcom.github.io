---
name: publishing-hugo-posts
description: Use when new Markdown files have been copied into this Hugo repository and need to match the existing PaperMod post style, appear in newest-first order, and be committed and pushed.
---

# Publishing Hugo Posts

## Outcome

Turn newly copied Markdown into publishable `content/posts/` entries without changing the article's meaning, then verify, commit only the intended files, and push the current branch.

## Workflow

1. Run `git status --short --branch`. Treat untracked Markdown outside `content/posts/` as candidates, but never include unrelated changes.
2. Read each candidate and a recent working post. Extract the article title from its first H1 and preserve the body verbatim apart from the metadata/style changes below.
3. Move the file into `content/posts/`, keeping a readable `.md` filename. If the destination exists, stop and resolve the collision instead of overwriting it.
4. Add YAML front matter in the repository's established shape:

   ```yaml
   ---
   title: "Exact article title"
   date: YYYY-MM-DDTHH:MM:SS+08:00
   draft: false
   tags: [specific, article, tags]
   comments: true
   ---
   ```

   Use an explicit date stated in the article when available; otherwise use the current Asia/Shanghai time. Give same-day imports distinct times so Hugo's default ordering is deterministic and newest-first. Choose 3–5 tags grounded in the article.
5. Keep the opening H1 only when that matches the repository's current post convention. Never solve a per-post title problem by editing global theme CSS: PaperMod must receive `title` from front matter so it renders `.post-title` consistently.
6. Validate before publishing:

   - Parse every changed post's YAML and require `title`, `date`, `draft`, `tags`, and `comments`.
   - Confirm dates sort in the intended newest-first order.
   - Run `git diff --check`.
   - Run `hugo --minify` when Hugo is installed. If it is unavailable, report that limitation and rely on the repository's GitHub Pages workflow for the full build.
7. Review `git diff` and stage only the intended source/destination paths. Do not stage other user files.
8. Commit with a concise message such as `Add AgentRisk review post`, push the current branch to its configured upstream, then verify `git status --short --branch` and the latest commit.

## Quick Reference

| Situation | Action |
|---|---|
| Missing front matter | Add the full five-field block |
| Filename-derived title or wrong title font | Set `title`; do not patch theme CSS |
| Several posts share a date | Assign distinct, descending times |
| Existing front matter | Preserve valid fields; normalize only what is needed |
| Destination collision | Stop; do not overwrite |
| Unrelated dirty files | Leave unstaged and mention them |
| Push/network failure | Retry only after resolving access; never claim publication |

## Completion Evidence

Report the destination path, effective title/date, verification performed, commit hash, pushed branch, and any build limitation. A successful local commit without a successful push is not a remote publication.
