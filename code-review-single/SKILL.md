---
name: code-review-single
description: Single-repo Code Review expert. Extracts the current branch diff via Git tracing, performs deep review of logic bugs, boundary gaps, readability, performance, and security issues. Generates a complete Code Review report covering critical issues, improvement suggestions, and elegant refactoring proposals.
trigger: Triggered when the user requests a Code Review, code audit, or review of the current commit in a single repository. Keywords: code review, review, code quality check, audit.


---

# Skill: code-review-single (Single-Repo Code Review Expert)

## Trigger Conditions

- User input contains: code review, review, code quality, audit
- User requests a review of changes on the current branch (single-repo scenario)

## Input Parameters

- `$ARGUMENTS`: Supports three modes (combinable):
  - Component path (optional, format `@component-name/`)
  - Commit hash (optional, 7–40 hex characters, e.g. `mf525235`, `abc1234`), diffs from that commit to HEAD
  - Commit count (optional, extract number from natural language, e.g. "last two commits" → 2, "latest 3" → 3)
  - Priority: commit hash > commit count > source branch

## Role: Senior Software Engineer performing Code Review

## Workflow

Execute strictly in the following two steps. Step 1 must be completed in a single terminal/code execution tool run:

### Step 1: Environment Resolution & Diff Extraction (single execution)

Use the terminal/code execution tool to run the following complete Bash script in one shot. Before running, replace the variables at the top of the script:

- `PARAM`: the user's `$ARGUMENTS` input, leave empty if none
- `COMMIT_COUNT_RAW`: number extracted from natural language, e.g. "last two" → 2, leave empty if none
- `COMMIT_HASH_RAW`: commit hash extracted from user input (7–40 hex characters), e.g. "mf525235" → mf525235, leave empty if none; **takes priority over commit count and source branch modes**

```bash
#!/bin/bash
PARAM="<replace with user input parameter, leave empty if none>"

# 1. Path resolution: strip @ and trailing /, then try to cd into it
if [ -n "$PARAM" ]; then
    CLEAN_PATH=$(echo "$PARAM" | sed -e 's/^@//' -e 's/\/$//')
    [ -d "$CLEAN_PATH" ] && cd "$CLEAN_PATH" || echo "Directory $CLEAN_PATH not found, staying in current directory"
else
    CLEAN_PATH=$(basename "$(pwd)")
fi

# Safety check: verify current directory is a valid Git repository
if ! git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
    echo "BLOCK: Current directory is not a valid Git repository. Aborting."
    exit 0
fi

REPO_NAME=$(basename "$(pwd)")

# 2. Get current branch and source branch
C_BR=$(git branch --show-current)

# Attempt to get source branch via reflog
O_BR=$(git reflog show "$C_BR" 2>/dev/null | awk '/Created from/ {print $NF; exit}')
O_BR_COMPARE="${O_BR#remotes/}"   
O_BR_COMPARE="${O_BR_COMPARE#origin/}" 

# [Key fix]: Handle reflog missing or pointing to remote self (e.g. after re-clone)
if [ -z "$O_BR" ] || [ "$C_BR" == "$O_BR_COMPARE" ]; then
    # Try to read the remote's configured default branch (usually main or master)
    DEFAULT_BR=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
    
    # If symbolic-ref fails, probe for common default branches
    if [ -z "$DEFAULT_BR" ]; then
        for br in main master develop; do
            # Check if remote has this branch
            if git rev-parse --verify "origin/$br" >/dev/null 2>&1; then
                DEFAULT_BR="$br"
                break
            fi
        done
    fi
    
    # Fall back to the detected default branch
    if [ -n "$DEFAULT_BR" ]; then
        O_BR="origin/$DEFAULT_BR"
        O_BR_COMPARE="$DEFAULT_BR"
    fi
fi

# 3. Block logic
if [ -z "$O_BR" ] || [ -z "$O_BR_COMPARE" ]; then
    echo "BLOCK: Source branch not found and no default branch matched. Aborting."
    exit 0
fi

if [ "$C_BR" == "$O_BR_COMPARE" ]; then
    echo "BLOCK: Current branch ($C_BR) is the source branch ($O_BR_COMPARE). No review needed. Aborting."
    exit 0
fi

# 4. Get Diff and assemble context
DIFF_TMP="/tmp/git_diff_raw_$(date +%s).txt"
FINAL_TMP="/tmp/git_diff_final_$(date +%s).txt"

# Get code diff (using origin-prefixed $O_BR for precise diff)
git diff "$O_BR" HEAD > "$DIFF_TMP"

# Check if there are substantive changes
if [ ! -s "$DIFF_TMP" ]; then
    echo "BLOCK: Current branch ($C_BR) differs from source ($O_BR) but has no substantive code changes. Aborting."
    rm -f "$DIFF_TMP"
    exit 0
fi

# Assemble context info and merge with diff for the model
echo "REPOSITORY_NAME: $REPO_NAME" > "$FINAL_TMP"
echo "TARGET_COMPONENT: $CLEAN_PATH" >> "$FINAL_TMP"
echo "BRANCHES: $O_BR -> $C_BR" >> "$FINAL_TMP"
echo "=========================================" >> "$FINAL_TMP"
cat "$DIFF_TMP" >> "$FINAL_TMP"

# Output final content and clean up
cat "$FINAL_TMP"
rm -f "$DIFF_TMP" "$FINAL_TMP"
```

*(Note: to avoid escape conflicts, the closing code fence above has an extra space — remove it in actual use)*

### Step 2: Deep Review & Report Output

After obtaining the diff output from Step 1, **strictly apply the following logic**:

1. If Step 1 output contains `BLOCK:`, immediately end the Code Review and briefly explain the reason to the user (e.g. source branch not found, no valid new commits, etc.).
2. If no block string is present, proceed with the deep Code Review.

**Review requirements**: Skip auto-generated files (e.g. package-lock.json). **Before reviewing, extract all changed files from the diff, review file by file in alphabetical order. Within each file, process every `+` line change independently — do not merge or skip any change point. For each `+` line change, scan strictly in the following 6-item order, each item must yield a clear conclusion (found/not found), none may be skipped**:
1. Security vulnerabilities (injection, privilege escalation, sensitive data exposure, OWASP Top 10)
2. **Crashes & exceptions (zero tolerance)**: null pointer dereference, array/collection out-of-bounds access, forced type cast failure, division by zero, uncaught exceptions, unreleased resources (file handles/database connections/memory leaks), thread safety issues (race conditions/deadlocks), stack overflow, infinite recursion, any code that may cause a crash or throw an unhandled exception
3. Logic errors & boundary gaps (including conditionals, loops, concurrency)
4. Performance issues (time complexity, redundant computation, memory leaks)
5. Code standards & readability (naming, redundant logic, magic numbers)
6. Refactoring opportunities (abstractable logic, duplicate code)

**Classification rules (strictly enforced, no subjective judgment)**:
- 🚫 Critical Issues: meets ANY of the following → (a) can cause program crash/abnormal exit (including null pointer, out-of-bounds, uncaught exception, deadlock, stack overflow, and all runtime crash risks) (b) security vulnerability exists (c) data loss or corruption (d) build/compilation failure. Scan items 1 and 2 results **always** go here, must not be downgraded to improvement suggestions.
- ⚠️ Improvement Suggestions: does not meet critical issue criteria, but belongs to issues found in scan items 3, 4, 5.
- 💡 Elegant Refactoring: scan item 6 results, or structural optimizations to existing implementations.

**Ordering rules**: Within the same category, order by filename alphabetically; within the same file, order by line number ascending. Each independent code location corresponds to one opinion — do not merge or split.
**Only report issues explicitly present in the diff. Do not speculate or add content beyond the diff.**
**Output requirements (mandatory, non-skippable)**:

- **Must** use the Write file tool (do NOT output content to chat as a substitute for writing the file)
- Write path: `<working directory at script execution time>/<TARGET_COMPONENT>-code-review.md` (i.e. the `pwd` directory, not the system root)
- If the current component has no changes, skip and do not generate
- After writing, output one confirmation line to the user: `Generated: <full file path>`

Use the following structure strictly in the generated Markdown file — do not force-fill sections where there are no findings:

# 📦 Repository: [read REPOSITORY_NAME from log]

**Branch comparison**: `[read source branch from log]` -> `[read current branch from log]`

## 📝 Change Summary

[Concise summary of the main code changes for this component/repo]

## 🚨 Deep Review Findings

*(All three categories must be output. If a category has no findings, write: `No issues of this type found in this change.` — do not omit the category heading.)*

### 🚫 Critical Issues

*List only issues meeting the critical classification criteria (crash/unhandled exception/security vulnerability/data corruption/build failure). **Any code that may cause a runtime crash or throw an unhandled exception must be listed here and must not be downgraded.** Issues not meeting criteria go to Improvement Suggestions.*

* **Issue**: [Precise description of the defect]

* **Potential Impact**: [Describe consequences, e.g. memory overflow, data leak]

* **Location**: Line L[start] - L[end]

* **Branch diff comparison**:

  ```javascript
  // Source branch code
  [Extract source branch code block in full]
  
  // Current branch changed code
  [Extract current branch changed code in full]
  ```

* **Fix comparison**:

  ```javascript
  // ❌ Original code
  [Extract the full problematic code block]
  
  // ✅ Fixed code
  [Provide refactored code following Clean Code and high-performance principles]
  ```

* **Fix highlights**: [Briefly describe the core advantage of the fix, e.g. reduced cyclomatic complexity]

### ⚠️ Improvement Suggestions

*List all issues not meeting critical criteria but belonging to logic errors/boundary gaps/performance issues/code standards. Order by filename alphabetically, then line number ascending.*

* **Suggestion**: [Describe the suggestion, e.g. use Optional Chaining instead of nested if checks]

* **Rationale**: [Explain why this change is better]

* **Location**: Line L[start] - L[end]

* **Branch diff comparison**:

  ```javascript
  // Source branch code
  [Extract source branch code block in full]
  
  // Current branch changed code
  [Extract current branch changed code in full]
  ```

* **Fix comparison**:

  ```javascript
  // ❌ Original code
  [Extract the full problematic code block]
  
  // ✅ Fixed code
  [Provide refactored code following Clean Code and high-performance principles]
  ```

* **Fix highlights**: [Briefly describe the core advantage, e.g. reduced cyclomatic complexity]

### 💡 Elegant Refactoring

*Must provide concrete refactoring proposals based on context. Strictly follow the comparison format below:*

* **Location**: Line L[start] - L[end]

* **Branch diff comparison**:

  ```javascript
  // Source branch code
  [Extract source branch code block in full]
  
  // Current branch changed code
  [Extract current branch changed code in full]
  ```

* **Fix comparison**:

  ```javascript
  // ❌ Original code
  [Extract the full problematic code block]
  
  // ✅ Refactored code
  [Provide refactored code following Clean Code principles, with higher performance or robustness]
  ```

* **Fix highlights**: [Briefly describe the core advantage, e.g. reduced cyclomatic complexity / improved O(n) efficiency]

## 🏁 Summary

* **Overall rating**: [Calculate by formula: 10 - (critical count × 3) - (improvement count × 0.5) - (refactoring count × 0.2), minimum 1, one decimal place. Format: X.X (critical × N, improvement × N, refactoring × N)]
* **Key risk**: [One sentence summarizing the most critical change to watch]

---

## Constraints

- Output in the same language the user is using.
- Be concise and direct. No filler or excessive pleasantries.
