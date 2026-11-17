# Development process

Work is planned in Linear and shipped through GitHub pull requests.

- **Linear project:** [evan-personal-data](https://linear.app/proudlyhumantm/project/evan-personal-data-1b8116258582) (team *Evan Personal*, issue IDs `EVA-…`)
- **Repo:** [EvanCWoods/nsw-crash-risk](https://github.com/EvanCWoods/nsw-crash-risk)

## How work is organised

| Linear | Means |
|---|---|
| **Milestone** | One build stage (1. Setup → 7. README). We finish them in order. |
| **Issue** | One piece of work small enough for a single pull request. |
| **Backlog** | Planned, not ready yet. |
| **Todo** | Ready to start next. |
| **In Progress** | Someone is working on it, and it has a branch. |
| **Done** | Its pull request is merged into `main`. |

Work is broken into **small steps**. Each step does one thing, takes minutes to
review, and has a **Done when** check you can run yourself. Steps appear in Linear in
one of two ways:

- **Sub-issues** under a parent issue (for example, EVA-63 to EVA-68 under EVA-38).
- **Checklist items** inside a parent issue (for example, steps 1–6 in EVA-39). We use
  these because the workspace has reached the free plan's 250-issue limit. They work
  the same way: one step, one branch, one PR.

Some steps are **"You do this"**: installing tools, or running `gcloud`/`bq` commands.
Those have no branch or PR. Run them, check the "Done when", then tick the step or
move it to Done.

The plan runs to a **checkpoint**: milestone 2 ends when Claude Desktop answers a
question about NSW crashes through our MCP server. Later milestones get split into
small steps once we reach them.

## The workflow for one issue

```
Todo ──► In Progress ──► branch ──► commits ──► pull request ──► review ──► merge ──► Done
```

1. **Pick** the top *Todo* issue in the current milestone and move it to *In Progress*.
2. **Branch** from an up-to-date `main`, using the issue's git branch name. In Linear,
   press `Cmd+Shift+.` to copy it.
   ```bash
   git switch main && git pull
   git switch -c evan/eva-38-run-airflow-3-locally-with-docker-compose
   ```
3. **Commit** in small steps. Start every message with the issue ID:
   `EVA-38: Add trimmed docker-compose for LocalExecutor`
4. **Open a pull request** whose title starts with the issue ID. The description says:
   - `Fixes EVA-38`
   - What changed and why, in plain language
   - How it was tested (commands run and what they showed)
5. **Review.** Evan reads the PR before merging, and asks about anything unclear. Being
   able to explain every line is the point of this project.
6. **Merge** with *Squash and merge*, so `main` gets one clean commit per issue. Then
   delete the branch.
7. **Close out:** the issue moves to *Done*. The stage explanation and interview
   questions go into [interview-notes.md](interview-notes.md).

### Checklist steps

A checklist step uses its parent issue's ID, plus the step number:

- Branch: `evan/eva-39-2-xlsx-to-parquet`
- Commits: `EVA-39: Add xlsx to Parquet helper`
- PR title: `EVA-39 (step 2/6): xlsx to Parquet helper`
- PR description: `Part of EVA-39`. Use `Fixes EVA-39` only on the **last** step, so
  the parent issue closes when the final PR merges.

After each merge, tick the step in the parent issue.

## Who does what

| Evan | Claude |
|---|---|
| Writes the code and tests | Explains each step, points to the docs, helps when stuck |
| Runs every command (git, gcloud, bq, docker) | Gives the commands and explains what each one does |
| Moves issues, creates branches, opens and merges PRs | Reviews PRs, and handles doc and housekeeping edits when asked |
| Decides anything that could cost money | Flags anything that could cost money before it happens |

## Definition of done

An issue is Done only when all of these are true:

- [ ] The code works, and was actually run and checked, not just written
- [ ] Tests pass, where the issue has testable code
- [ ] Docs are updated (README, `infra/README.md`, dbt docs)
- [ ] No secrets in the diff
- [ ] The plain-language explanation and 2–3 interview questions are in `docs/interview-notes.md`
- [ ] The PR is merged into `main`

## Commit identity

Commits are authored by Evan and signed with his SSH key, so GitHub shows them as
**Verified**. Commits Claude helps with end with a `Co-Authored-By: Claude` line.
