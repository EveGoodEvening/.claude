Use the ralph-loop plugin to review and fix implementation issues via an iterative review-fix loop.

Unlike `/ralph-fix` (which starts from a known problem), this command starts by asking Codex to resolve the review scope into reviewable chunks, then asks independent Codex reviewers to review those chunks, then fixes whatever they find.

## Review scope

$ARGUMENTS

## Loop behavior to write into the ralph prompt

1. **Resolve scope** — Run the `codex-ask` skill before any code review.
   - Pass the raw scope description directly to Codex. Do NOT resolve it yourself into concrete diff commands.
   - Ask Codex to resolve the requested scope into concrete, reviewable chunks.
   - The `codex-ask` Codex is a scope resolver only. It MUST NOT review code or report implementation findings.
   - Ask it to return, for each chunk:
     - the concrete commands/paths the reviewer should inspect;
     - a short goal/context summary;
     - clear boundaries that keep the chunk independently reviewable;
     - caveats if the raw scope is ambiguous or cannot be resolved.
   - Valid raw scope examples:
     - `uncommitted` or empty → uncommitted changes
     - `last 3 commits` → last 3 commits
     - `branch X vs branch Y` → diff between two branches
     - `<commit-sha>` → a specific commit
     - `<file-path>` → changes in a specific file
2. **Review chunks** — For each chunk returned by the resolver, run one separate `codex-review-code` skill invocation.
   - Use as many independent Codex reviewers as there are chunks.
   - Each reviewer gets only its assigned chunk plus enough overall context to understand the implementation goal.
   - Reviewers MUST review code. They MUST NOT broaden scope beyond their assigned chunk unless required to understand a concrete issue.
   - If every chunk reviewer finds no issues → output `<promise>ALL CLEAN</promise>` and STOP.
3. **Merge reports** — Combine all chunk reviewer reports.
   - Deduplicate overlapping findings.
   - Preserve which chunk/reviewer produced each retained finding.
4. **Evaluate** — Judge the merged review feedback. For each item, classify it as:
   - **Incorporate** — correct and valuable → fix it.
   - **Discard** — wrong, out-of-scope, duplicate, or low-priority → dismiss it.
5. **Decide:**
   - If nothing to incorporate (all discarded or no actionable feedback) → done, STOP without claiming `<promise>ALL CLEAN</promise>`.
   - Otherwise → proceed to step 6.
6. **Fix** — Apply the changes you decided to incorporate, run relevant checks, then `/commit-push`.
7. **Re-review** — Go back to step 1. On re-review iterations, tell the scope resolver to chunk the original scope plus any uncommitted working tree changes. Do NOT emit `<promise>ALL CLEAN</promise>` here. Only chunk reviewers in step 2 can justify ALL CLEAN — you cannot self-certify your own fixes.

## How to invoke ralph-loop

The ralph-loop skill runs a shell setup script that cannot handle backticks, special characters, or long multi-line arguments passed directly. To work around this:

1. **Clean up stale loop files first:**
   Remove any leftover files from previous runs so the user isn't prompted for permission on files that already exist:
   ```
   rm -f .claude/ralph-loop-prompt.local.md .claude/ralph-loop.local.md
   ```

2. **Write the prompt to a file:**
   Write the loop behavior and the raw scope description to `.claude/ralph-loop-prompt.local.md` using the Write tool. Do NOT include the "How to invoke" section — only the loop behavior and scope description.

3. **Invoke ralph-loop with a short, shell-safe argument:**
   Run the setup script directly via Bash tool: `CLAUDE_CODE_SESSION_ID="${CLAUDE_CODE_SESSION_ID:-}" "$HOME/.claude/plugins/marketplaces/claude-plugins-official/plugins/ralph-loop/scripts/setup-ralph-loop.sh" "See .claude/ralph-loop-prompt.local.md"`
   Do NOT pass the raw review scope text to ralph-loop — it will break if the text contains backticks, quotes, or other shell metacharacters.

## When the loop ends

After the review-fix cycle converges, clean up the loop files:
```
rm -f .claude/ralph-loop-prompt.local.md .claude/ralph-loop.local.md
```

**Important:** Do NOT start fixing directly. Write the prompt file, then invoke `/ralph-loop` and let it drive the review-fix cycle.
