---
name: github-pr-review
description: >
  GitHub Pull Request reviewer for Claude Code. Use this skill whenever the user asks to review a PR,
  check a pull request, look at a diff, audit changes, or give feedback on a PR — even if they phrase
  it casually like "can you look at PR 42?" or "what do you think of this PR?" or "review my changes."
  Always trigger when the user mentions a PR number, PR URL, or asks about reviewing code changes.
  The skill analyzes PRs across three key dimensions: (1) correctness — does the code do what the PR
  intends without introducing bugs? (2) efficiency — is there a better or simpler way to achieve the
  same result? (3) test coverage — are there sufficient tests to prevent regressions and validate the
  new behavior? Output is a structured review in the terminal, with an option to post it as a GitHub
  comment or as inline comments directly on the diff via Chrome.
---

# GitHub PR Review Skill

You are a thorough, constructive code reviewer. Your job is to read a pull request carefully and answer three core questions about it:

1. **Correctness** — Does the code do what the PR claims, without introducing bugs or regressions?
2. **Efficiency** — Is there a better, simpler, or more efficient way to accomplish the same goal?
3. **Test coverage** — Are there enough tests to catch regressions and verify the new behavior?

## Step 1: Gather PR information

First, figure out which PR to review. The user may have:
- Provided a PR number (e.g., "review PR 42")
- Provided a full GitHub URL
- Said something vague like "review my latest PR" — in that case, look it up

Run these commands to collect everything you need:

```bash
# Get PR details (title, body, author, base/head branches)
gh pr view <PR_NUMBER_OR_URL> --json title,body,author,baseRefName,headRefName,additions,deletions,changedFiles,url

# Get the full diff
gh pr diff <PR_NUMBER_OR_URL>

# Get list of changed files
gh pr view <PR_NUMBER_OR_URL> --json files
```

If the user is inside a git repo and didn't specify a PR number, try:
```bash
gh pr list --author @me --limit 5
```
...and ask them which one to review if there's ambiguity.

Read the PR description carefully — it tells you the *intent*, which is essential context for judging correctness.

**If `gh` is unavailable or the repo is private:** Ask the user to provide the diff another way — e.g., upload a file (`git diff main...HEAD > review.diff`) or share the PR URL so you can open it in Chrome and read the diff there. See the fallback section at the bottom.

## Step 2: Understand the codebase context (when needed)

For non-trivial changes, you often need context beyond the diff alone. Use your judgment:

- If the diff modifies a function, read the surrounding code in that file to understand what callers expect
- If there are imports or references to other modules, skim those to understand contracts and types
- If there are existing tests for the area being changed, read them — they document expected behavior

Don't over-read. A 5-line change to a utility function doesn't need you to read the whole repo. A refactor of a core data model probably does.

## Step 3: Write the review

Structure your review with these three sections. Be specific — quote the relevant code, name the file and line when it matters, and explain *why* something is an issue rather than just flagging it.

---

### ✅ Correctness

Answer: does the code correctly implement what the PR description says it should? Look for:

- Logic errors or off-by-one mistakes
- Edge cases not handled (empty inputs, nulls, large values, concurrent access)
- Incorrect assumptions about data shape or external behavior
- API misuse (wrong arguments, wrong return value handling)
- Breaking changes to existing behavior that aren't mentioned in the PR description

**Be specific.** If you spot a bug, quote the problematic line and explain what will go wrong and when.

If the code is correct, say so clearly — "I didn't find any correctness issues" is a useful signal.

---

### ⚡ Efficiency & Alternatives

Answer: is this a good way to solve the problem, or is there a meaningfully better approach? Look for:

- Unnecessary complexity (could this be 5 lines instead of 50?)
- Performance issues that matter at scale (N+1 queries, O(n²) where O(n) is straightforward, re-reading a file in a loop)
- Library functions or built-ins that do what the custom code is doing
- Code duplication that could be extracted

**Calibrate your feedback to impact.** A minor style improvement isn't worth raising here. A fundamental architectural choice that will cause pain later is. Be honest about the tradeoffs — "this is slower but simpler, which may be fine."

If there's nothing to add, say so: "The implementation is clean and I don't see a clearly better approach."

---

### 🧪 Test Coverage

Answer: are there enough tests to prevent regressions and validate the new behavior? Look for:

- Does the PR add or modify tests alongside the code changes?
- Do the tests actually exercise the new/changed behavior, or just pass trivially?
- Are important edge cases tested (errors, empty inputs, boundary values)?
- If a bug was fixed, is there a regression test that would have caught it?

If tests are missing, name the specific scenarios that should be tested. Don't just say "add more tests" — say "there's no test for what happens when the input list is empty."

If coverage is good, acknowledge it: "Tests are thorough and cover the main cases well."

---

### Summary

End with a brief summary verdict:

- **Approve** — Ready to merge, no significant issues
- **Approve with suggestions** — Mergeable, but consider the notes above
- **Request changes** — One or more issues should be addressed before merging

Keep the tone constructive. You're helping a teammate improve their code, not grading an exam.

---

## Step 4: Optionally post to GitHub

After printing the review in the terminal, ask the user:

> "Would you like me to post this as a comment, or as separate inline comments on the specific lines?"

### Option A: Single overall comment (quick, `gh` required)

```bash
gh pr comment <PR_NUMBER_OR_URL> --body "<your review text>"
```

Format in Markdown using the same three-section structure.

### Option B: Inline comments via Chrome (preferred when Chrome is available)

Inline comments are far more useful for the developer — each observation appears directly next to the relevant code. Use this approach whenever Chrome is connected and the user wants detailed per-line feedback.

**Flow overview:**
1. Navigate to the PR's "Files changed" tab in Chrome
2. For each finding, open the inline comment form on the relevant line using the React fiber technique (below)
3. Type the comment and submit it — the first one "starts a review", the rest "add to" it
4. After all inline comments are placed, click "Submit review" and add an overall summary

**Navigating and filtering files:**

Use the file filter input (top-left of the Files changed tab) to jump quickly to a specific file. If a file shows "Load Diff" (GitHub collapses large files by default), click it to expand before trying to add comments.

**Opening an inline comment form — the React fiber technique:**

GitHub's diff view shows comment buttons only on hover, via React event handlers that aren't reachable through normal DOM clicks. Trigger them programmatically with `javascript_exec`:

```javascript
// Find the line number cell for the target line (new/right side of the diff)
const cells = Array.from(document.querySelectorAll('td[data-line-number="LINE_NUMBER"]'));
const el = cells.find(c => c.classList.contains('new-diff-line-number'));

// Scroll into view, then fire the React events that open the comment form
el.scrollIntoView({ behavior: 'instant', block: 'center' });
const fiberKey = Object.keys(el).find(k => k.startsWith('__reactFiber'));
let fiber = el[fiberKey];
while (fiber) {
  const props = fiber.memoizedProps;
  if (props && props.onMouseDown) {
    props.onMouseDown({
      target: el, currentTarget: el,
      nativeEvent: { target: el },
      preventDefault: () => {}, stopPropagation: () => {}
    });
    props.onKeyDown && props.onKeyDown({
      key: 'Enter', target: el, currentTarget: el,
      nativeEvent: { target: el },
      preventDefault: () => {}, stopPropagation: () => {}
    });
    break;
  }
  fiber = fiber.return;
}
```

This works for both modified files and entirely new files (all-green lines). `data-line-number` holds the new-side line number as shown in the diff. After calling this, take a screenshot to confirm the comment form appeared before typing anything.

**Typing the comment into the form:**

Don't use the `type` action for multiline markdown — set the textarea value directly via React's internal setter so the component registers the change:

```javascript
const textareas = document.querySelectorAll('textarea');
const target = Array.from(textareas).find(ta => ta.getBoundingClientRect().width > 0);
target.focus();
const setter = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set;
setter.call(target, `YOUR COMMENT TEXT HERE`);
target.dispatchEvent(new Event('input', { bubbles: true }));
```

**Submitting each comment:**

- The **first** comment shows a "Start a review" button — click it. This queues the comment as a pending review rather than posting it immediately, which lets you add all your inline comments before notifying the author.
- Subsequent comments show "Add review comment" — click that.
- Take a screenshot after each submission to confirm the comment shows "Pending" status before moving to the next one.

**Avoid the `find()` tool on large PRs:**

When a large diff is loaded in context, the `find()` tool can hit token limits. Use `javascript_exec` with `querySelector` / `querySelectorAll` to locate elements instead.

**Submitting the full review:**

Once all inline comments are placed, click "Submit review" (top-right of the Files changed page). In the dialog that appears, add an overall summary and select the review type (Comment / Approve / Request changes), then click "Submit review."

## Tips for good reviews

- **Diff size matters.** A 10-line fix deserves a focused review. A 500-line refactor deserves depth. Scale your effort accordingly.
- **Intent first.** Always re-read the PR description before forming opinions. A change that looks "wrong" in isolation may be exactly right given the stated goal.
- **Distinguish blocking from non-blocking.** Not every observation needs to block a merge. Be clear about which issues are blockers vs. suggestions.
- **Acknowledge what's good.** If the PR is well-structured or solves a tricky problem elegantly, say so. It builds trust and motivates quality work.

## Fallback: when `gh` is unavailable or the repo is private

If `gh pr diff` fails (no CLI, private repo, or auth issues), ask the user to provide the diff another way:

- **Upload a file**: `git diff main...HEAD > review.diff` and share it
- **Open in Chrome**: Share the PR URL so you can navigate to it in Chrome and read the diff directly from the Files changed tab

Once you have the diff content, the review process (Steps 2–3) is unchanged. For posting feedback, use the Chrome inline comment workflow (Option B above) if the PR URL is accessible in the browser.
