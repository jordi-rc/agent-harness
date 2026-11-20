---
name: create-merge-request
description: Write the title and the description of a GitLab merge request following the team template, with before/after evidence of the change (screenshots, videos, outputs and architecture diagrams). Use before you create or edit a merge request, and after you push new commits to a branch with an open merge request.
disable-model-invocation: false
user-invocable: true
metadata:
  author: Codely <support@codely.com> (https://codely.com)
  version: "1.0"
  license: MIT
---

# How to write a merge request

Write the title and the description in ASD-STE100 Simplified Technical English. Follow the rules in [`references/ste100.md`](references/ste100.md). Also apply these rules when you update the merge request.

Use `glab` for all the GitLab operations. Never hardcode a project or a host: `glab` resolves them from the Git remote, so the skill works on gitlab.com and on self-managed instances. Run `glab auth status` first. If `glab` is not authenticated for the host of the remote, stop and tell the user.

## 1. Title

- Use the Conventional Commits format of [`commit-messages.md`](../git-commit/references/commit-messages.md): `type(scope): summary`.
- If the repository restricts the scopes (a script, a config file or a CI check that validates the title), use one of them.
- Do not add the `Draft:` prefix by hand. Use `--draft` if the user asks for a draft.

## 2. Agent label, assignee and reviewers

If the project has `agent:*` labels, an agent that creates the merge request adds exactly one, with its own name. Automations such as a cross-agent review can read it.

| Agent       | Label          |
| ----------- | -------------- |
| Claude Code | `agent:claude` |
| Codex       | `agent:codex`  |
| Cursor      | `agent:cursor` |

- Add the label with `glab mr create --label agent:<name>`.
- If a person creates the merge request without an agent, do not add an `agent:*` label.
- GitLab does not fail when a label does not exist: it creates it in the project. Check the labels first with `glab label list`. If `agent:<name>` is not in the list, create the merge request without the label and tell the user.

Assign the merge request to the person who directed the agent (the person who wrote the prompt). `glab` needs a username, it does not accept `@me`:

- If `glab` uses that person's credentials (local CLI, desktop), get the username with `glab api user | jq -r .username` and add `--assignee <username>`.
- If `glab` uses a bot, a project access token or a CI job token (cloud agents, GitLab CI) and you know the GitLab username of the person who gave the task, add `--assignee <username>`.
- If you do not know the username, do not guess. Create the merge request without an assignee and tell the user.

Add reviewers with `--reviewer <username>` only if the user names them or the project has a rule for them (for example `CODEOWNERS` with approval rules). Do not guess reviewers.

## 3. Description

Do not use `--fill`: it fills the description with the commit messages. Build the description from a template explicitly:

1. Find the template of the project in `.gitlab/merge_request_templates/`:
   - If there is one template, use it.
   - If there are several, use `Default.md` if it exists. Otherwise, pick the one whose name matches the type of change (for example `Bug.md` for a `fix`). If none matches, ask the user in an interactive session, or use the bundled template in an unattended run.
   - If the folder does not exist, use [`references/merge-request-template.md`](references/merge-request-template.md).
2. Copy it to a temporary file outside the repository, for example `"$TMPDIR/mr-description.md"`.
3. Fill each section with facts from the diff and from your session.
4. Delete the sections that do not apply and all the HTML comments.
5. If an issue motivates the change, add `Closes #<iid>` (or `Related to #<iid>` if the merge request does not finish it) in "🎯 Why". Some self-managed instances disable automatic issue closing: never promise in the description that the merge closes the issue.
6. Pass the file with `--description-file`. If your glab does not know the flag (`glab mr create --help`), use `--description "$(cat "$TMPDIR/mr-description.md")"`.

Rules for each section of the bundled template. With the template of the project, apply the rules of the sections it shares:

- **🎯 Why:** write the problem, not the solution. Link the issue or the discussion if you know it.
- **🛠️ What changes:** 2–5 bullets with decisions, trade-offs and non-obvious effects. Do not repeat the commit list.
- **📸 Evidence:** see "4. Evidence".
- **✅ How I verified it:** write the commands that you ran and their real result. Write what you did not check. Do not write a check that you did not do. Do not write that the pipeline passes until it passes: check it with `glab ci status`.
- **🧩 Complexity:** one line, `**Level:** low / medium / high · <what makes it so>`, for example `**Level:** medium · a new endpoint, use case and table`. The level is how hard the change is to review, not a verdict.
- **🚀 Deploy steps:** optional. Add it only if someone must do something by hand to ship the merge request (run a query or a script in production, change a flag or an environment variable, run a manual pipeline job, configure an external service…). Otherwise, delete it. Write one step for each action, in the order to run them, as a numbered checkbox (`1. [ ] …`) so whoever runs it can check it. Start each step with when to run it in bold (**Before merging:**, **After deploying:**…), then where to run it and the exact command, query or flag in a code block. Never check a box yourself unless you ran that step.
- **↩️ Rollback:** the steps to undo the merge request in production, in the order to run them. If reverting the merge request is enough, write only "Revert this merge request." If the change touches state outside the code (database migrations, backfills, feature flags, queues, crons, external config, published events), write each step with the exact command, migration name or flag, and say what data is lost or kept. Undo the deploy steps too, for example with the inverse query. Put the steps that undo state before the revert when the old code cannot run with the new state, and after it otherwise. If a part cannot be undone (deleted data, sent emails, events already consumed), say so and write what to do instead. Read the migrations of the diff to write this, never guess them.
- **🤖 Authorship:** one sentence. Name the harness and the model with its version, then who started the work, see "Who started the work" below. For example "Claude Code with Claude Opus 5.5, guided by Jane Doe (@janedoe)." If you added an `agent:*` label, name the same agent.

If your tool adds an attribution footer (for example "🤖 Generated with Claude Code"), keep it as the last line.

### Who started the work

Never write "the user" and never guess a name. Pick the first case that applies:

| How the agent runs                                                  | End of the Authorship sentence, after "Claude Code with Claude Opus 5.5,"                                                 |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| With the person's own `glab` credentials (local CLI, desktop)       | "guided by Jane Doe (@janedoe)." Get it with `glab api user \| jq -r '"\(.name) (@\(.username))"'`.                       |
| Started by a comment, an issue or a mention (`@claude`, `@codex`…)  | "guided by @janedoe from [this comment](link)." Use the author of the comment or the person who assigned the issue.       |
| Started by a scheduled pipeline or a routine                        | "triggered by the scheduled pipeline <name>, owned by @janedoe." The owner of a pipeline schedule is in its settings.     |
| Started by an automatic event with no person behind it              | "triggered automatically by <event> ([link](link)). No human directed it."                                                |

With a bot, a project access token or a CI job token, `glab api user` returns the bot, or fails with the job token. Take the person from the event instead (in GitLab CI, `GITLAB_USER_LOGIN` is the user who started the pipeline). Ask who it is only in an interactive session. If the agent runs unattended and the event does not name a person, use the last row.

## 4. Evidence

The evidence proves the change that "🎯 Why" describes. A reviewer must see that change without running the code.

- Capture the main change, not a side effect of it. For example, if the fix restores a tool that agents call, a screenshot of a new loading style does not prove the fix: the tool call and its result do.
- For a fix, the "before" with the failure is the most important part. Never skip it because the page looks the same: capture the failure in the form that shows it (console error, response, tool result).
- Pick the formats for the kind of change. A merge request can need more than one:
  - **Visual change** (UI, pages, emails, copies, OG images…): screenshots. See "Screenshots".
  - **Flow** (several steps or an interaction): a short video or GIF. See "Videos".
  - **Behavior without a visual change** (agent tools, API endpoints, scripts, workers…): the same input on each side and the output of each side (tool call and returned JSON, request and response, command and output) in code blocks. Trim long outputs to the relevant fields.
  - **Architecture change** (how components call each other, a decision or routing flow, the order of calls, the dependencies between modules): before/after diagrams. See "Architecture diagrams". Add them to the other evidence, do not replace it.
- Delete the section only if nobody can observe the change outside the tests and the architecture does not change (YAML, config, CI, docs, tests, a refactor inside one component…).
- If the project has review apps, you can use the review app URL of the branch as the "after" environment. Write the URL in the first line of the section.

### Structure of the section

1. Start with one line that tells where each side comes from: the base commit, the branch, the environment, the URL or state, and the shared input. For example: "Before is `main` at `a3d91f2` and after is this branch, both on the local dev server at `/pricing` with the same seed data."
2. Put the main change first. Then add the scenarios and the variants.
3. Write one block for each thing to prove: a `###` heading and one sentence that tells the reviewer what to look at. Then the evidence.
4. Pick the layout of each block:

| Layout                      | Use it for                                                                                        |
| --------------------------- | ------------------------------------------------------------------------------------------------- |
| Before/after table          | The main change.                                                                                  |
| Close-up crop               | A defect of a few pixels. Use the same crop and zoom on both sides.                               |
| State grid                  | One column for each state that the reviewer cannot infer from the main case (empty, error, long). |
| Variant matrix              | One row for each variant (width, theme, language) and one column for each side.                   |
| One subsection per platform | The same check on several platforms, browsers or environments.                                    |
| Inline GIF and linked video | A short flow as a GIF, and the full walkthrough as a video.                                       |
| Pair of diagrams            | An architecture change, with Before above After.                                                  |

### Screenshots

- For a visual change on the web, always capture desktop (1440 px) and mobile (390 px), on both sides. Show them in a variant matrix.
- If the change exists at one width only (for example, a menu that shows only on mobile), capture that width and write why the other one is missing.
- For emails, also capture both widths.
- Add other variants (dark mode, tablet, languages…) only if the change affects them.
- Use the same viewport and the same crop on both sides of each width.

### Videos

Read [`references/video-recording.md`](references/video-recording.md) before you record.

- Show all the relevant part of the screen: the place of the action and its result. If the result shows far from the click (a toast, a side panel), keep both in the frame.
- Show where each click happens: a visible cursor and a mark on each click.
- Show what you type: type at a readable speed and show the keyboard shortcuts on the screen.
- Wait about 1 s after each action, so the reviewer sees the result before the next step.
- Keep each video under 60 s. Split a longer flow into several videos.
- Write one sentence above each video with the steps that it shows and its duration.
- If the flow changes, record each side with the same steps, or put both sides next to each other.
- Use the same viewports as the screenshots. Record mobile too if the flow changes there.

### Architecture diagrams

Read [`references/architecture-diagrams.md`](references/architecture-diagrams.md) before you draw.

- Add diagrams when the merge request changes how components call each other, a decision or routing flow, the order of calls, or the dependencies between modules. Omit them if the architecture does not change.
- Pick the diagram for the change:
  - **Decision or routing:** `flowchart TD` with decision nodes.
  - **Order of messages between components:** `sequenceDiagram`.
  - **Structure or dependencies between modules:** `flowchart LR`.
- Draw a Before diagram and an After diagram with the same node names, one above the other. Mermaid does not render inside a table.
- Mark what is added in green, what is removed in red with a dashed line (in Before only) and what changes in amber. Add the legend line above the pair.
- For a new flow with no "before", draw only After, with all of it marked as added.
- Draw only the affected part, with about 12 nodes at most. Draw one pair for each affected flow.

### Produce the evidence

1. Create a temporary directory, for example `mkdir -p "$TMPDIR/mr-evidence"`. Never commit the screenshots or the videos.
2. Capture the "before" state from the target branch and the "after" state from the source branch, with the same input.
3. Capture both widths, as in "Screenshots".
4. If the change is a flow, record it as in "Videos".
5. If the architecture changes, write the diagrams as in "Architecture diagrams". They are Markdown, so they go in the description and need no upload.

Sources for each type of change:

- **Web pages:** use a browser automation tool (for example Playwright) against the local dev server or the review app of each side.
- **Emails:** render the email template with the same sample params on each side, write the result to an HTML file and screenshot it.
- **Architecture:** read the code of each side (`git show origin/<target>:<path>` for the "before") and draw the components that the diff touches.

## 5. Attach the images and the videos

Upload each file to the project with the uploads API and put the Markdown that GitLab returns in the description. This places each file where it belongs, works on every GitLab version and on self-managed instances:

```bash
glab api projects/:fullpath/uploads --form "file=@$TMPDIR/mr-evidence/before-desktop.png" | jq -r .markdown
# ![before-desktop](/uploads/<secret>/before-desktop.png)
```

- `--form` needs a recent glab. If `glab api` does not know the flag, update glab, or use the experimental `--attach` flag of `glab mr create` / `glab mr update`. `--attach` only appends the files at the end of the description, so move each one to its place afterwards.
- Run `glab` from the repository root, so `:fullpath` resolves to the project of the remote.
- The returned path (`/uploads/<secret>/<file>`) is relative to the project and renders only in that project. Do not copy it to another project.
- Replace the alt text of the returned Markdown with a description of the image, for example `![Course card before, desktop](/uploads/<secret>/before-desktop.png)`.
- A video uses the same syntax, and GitLab renders it as a player. Write the sentence with its steps above it, see [`references/video-recording.md`](references/video-recording.md).
- The default size limit for each file is 100 MB, but an instance can set a lower one. Keep each video under 10 MB anyway, so the description loads fast.

Example:

```bash
for side in before after; do
  for width in desktop mobile; do
    glab api projects/:fullpath/uploads --form "file=@$TMPDIR/mr-evidence/$side-$width.png" | jq -r .markdown
  done
done
```

```markdown
|             | Before                                                          | After                                                         |
| ----------- | --------------------------------------------------------------- | ------------------------------------------------------------- |
| **Desktop** | ![Course card before, desktop](/uploads/a1b2/before-desktop.png) | ![Course card after, desktop](/uploads/c3d4/after-desktop.png) |
| **Mobile**  | ![Course card before, mobile](/uploads/e5f6/before-mobile.png)   | ![Course card after, mobile](/uploads/g7h8/after-mobile.png)   |
```

```bash
glab mr create \
  --title "fix(courses): align course card price" \
  --description-file "$TMPDIR/mr-description.md" \
  --target-branch main \
  --label agent:claude \
  --assignee janedoe \
  --yes
```

- Push the branch before (`git push -u origin HEAD`), or let `glab mr create` push it. Do not use `--fill`.
- Do not pass `--remove-source-branch` or `--squash-before-merge` unless the user asks: without them, GitLab uses the defaults of the project.
- Never merge the merge request and never add `--auto-merge`. The user decides when to merge.

Open the merge request with `glab mr view --web`, or read it with `glab mr view`, and make sure that no local path stays in the description, that each image or video shows, and that each diagram renders.

## 6. Update the merge request after new commits

After each push, run `glab mr view -F json | jq -r .description`. If the branch has an open merge request, update its description:

1. Write the current description to `"$TMPDIR/mr-description.md"`.
2. Compare each section with the full diff against the target branch (`git diff origin/<target>...HEAD`).
3. Update the sections that changed:
   - **🛠️ What changes:** the current changes of the whole merge request.
   - **📸 Evidence:** capture new screenshots or videos if the visual change or the flow is different. Update the diagrams if the architecture is different. Add the section if a visual or architecture change appears. Delete it if none remains.
   - **✅ How I verified it:** the checks for the current state.
   - **🧩 Complexity:** the level and its reason for the current state.
   - **🚀 Deploy steps:** add the section if a manual step appears, and delete it if none remains. Keep the boxes that someone already checked.
   - **↩️ Rollback:** the steps for the current state, including new migrations or flags.
   - **🤖 Authorship:** new contributors.
4. Upload the new images or videos as in "5. Attach the images and the videos", and put their Markdown in the description.
5. Apply it with `glab mr update --description-file "$TMPDIR/mr-description.md"`. Then make sure that no local path stays in the description.

The description always describes the current state of the whole merge request. Do not append a commit log or a "changes since last review" list. Do not change the `agent:*` label, the assignees or the reviewers. Do not mark a draft as ready (`--ready`) unless the user asks.
