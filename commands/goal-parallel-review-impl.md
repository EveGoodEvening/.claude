/goal Review and fix implementation issues via a Codex scope-resolution step, parallel Codex chunk reviews, then the existing review-fix loop.

Review scope:
$ARGUMENTS

Outcome:
Run an iterative review-fix loop until Codex reviewers report the implementation is clean for every reviewable chunk, or until there are no correct/actionable findings to incorporate.

Review orchestration:
1. **Resolve scope with `codex-ask` first.**
   - Run the `codex-ask` skill before any code review.
   - Ask Codex to resolve the raw review scope into concrete, reviewable chunks.
   - Pass the scope description directly to this Codex. Do not resolve it yourself into concrete git diff commands.
   - This Codex is the scope resolver only. It MUST NOT review code or report findings.
   - Ask it to return:
     - the concrete commands/paths each reviewer should inspect;
     - a short goal/context summary for each chunk;
     - chunk boundaries that are independently reviewable and small enough for focused review;
     - any caveats if the raw scope is ambiguous or cannot be resolved.
2. **Review chunks with `codex-review-code`.**
   - For each chunk returned by the scope resolver, run one separate `codex-review-code` skill invocation.
   - Use as many independent Codex reviewers as there are chunks.
   - Each reviewer gets only its assigned chunk plus enough overall context to understand the implementation goal.
   - Reviewers MUST review code. They MUST NOT broaden the scope beyond their assigned chunk unless required to understand a concrete issue.
3. **Merge review reports.**
   - Combine all chunk reviewer reports before evaluating findings.
   - Deduplicate overlapping findings.
   - Preserve which chunk/reviewer produced each retained finding.

Scope examples for the resolver:
- empty or `uncommitted` → uncommitted changes
- `last 3 commits`
- `branch X vs branch Y`
- `<commit-sha>`
- `<file-path>`

Re-review:
- On the first review, ask the scope resolver to chunk the original scope.
- On every re-review after fixes, ask the scope resolver to chunk the original scope plus any uncommitted working tree changes.
- Then run fresh chunk reviews for the resolver's current chunks.

Evaluate:
- For each merged Codex finding, classify it as:
  - Incorporate — correct, relevant, and worth fixing.
  - Discard — incorrect, out of scope, duplicate, low-priority, or not worth changing.
- Briefly record the classification and reason.

Fix:
- If at least one finding is classified Incorporate, apply one focused set of fixes.
- Do not make unrelated refactors or opportunistic cleanup.
- Preserve intended behavior unless the incorporated finding requires a behavior change.
- After fixing, run relevant checks.
- Then run `/commit-push`.

Iteration policy:
- After every fix and `/commit-push`, return to Review orchestration.
- Do not self-certify the implementation as clean.
- Do not emit `<promise>ALL CLEAN</promise>` after making fixes.
- Only the chunk reviewers, during a Review step, can justify declaring the implementation clean.

Verification:
- Evidence must include the scope resolver output and every chunk reviewer output for the current iteration.
- Evidence for fixes must include the relevant test/check commands or an explanation if no automated check applies.
- Re-review must include the current repo state, including uncommitted changes, so reviewers can see the fixes.

Constraints:
- Fix only implementation issues found by chunk reviewers that are classified Incorporate.
- Do not change files outside the reviewed chunks unless required to correctly fix an incorporated issue.
- Do not bypass hooks, skip checks, force-push, or use destructive git operations.
- Do not emit `<promise>ALL CLEAN</promise>` unless all chunk reviewers report no issues in the Review step.

Boundaries:
- Allowed writes: files necessary to fix incorporated findings.
- Forbidden writes: unrelated documentation, unrelated refactors, generated artifacts, secrets, dependency changes unless explicitly required by an incorporated finding.

Stop when:
- The scope resolver succeeds and every chunk reviewer finds no issues in a Review step; then output exactly:
  `<promise>ALL CLEAN</promise>`
- Or merged Codex findings are all classified Discard with no actionable fixes remaining; summarize why and stop without claiming ALL CLEAN.

Pause if:
- The scope resolver cannot produce reviewable chunks from the requested scope.
- A finding requires a product decision, risky migration, destructive action, dependency downgrade/removal, security-sensitive change, or scope expansion.
- Checks fail for reasons unrelated to the incorporated fixes.
- The loop exceeds 5 review-fix iterations.