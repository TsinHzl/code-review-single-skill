---
name: code-review-single
description: Single-repo Code Review expert. Extracts the current branch diff via Git, performs deep review of logic bugs, boundary gaps, readability, performance, and security issues. Generates a complete Code Review report covering critical issues, improvement suggestions, and elegant refactoring proposals.
trigger: Triggered when the user requests a Code Review, code audit, or review of the current commit in a single repository. Keywords: code review, review, code quality check.

---

# Skill: code-review-single (Single-Repo Code Review Expert)

## Trigger Conditions

- User input contains: code review, review, code quality
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

### Step 1: Environment Resolution & Diff Extraction (run all at once)

Use the terminal/code execution tool to run the following complete Bash script in one shot. Before running, replace the three variables at the top of the script:

- `PARAM`: the user's `$ARGUMENTS` input, leave empty if none
- `COMMIT_COUNT_RAW`: number extracted from natural language, e.g. "last two" → 2, leave empty if none
- `COMMIT_HASH_RAW`: commit hash extracted from user input (7–40 hex characters), e.g. "mf525235" → mf525235, leave empty if none; **takes priority over commit count and source branch modes**

```bash
#!/bin/bash
PARAM="<replace with user input parameter, leave empty if none>"
COMMIT_COUNT_RAW="<replace with commit count number extracted from user input, leave empty if none>"
COMMIT_HASH_RAW="<replace with commit hash extracted from user input, leave empty if none>"

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
C_BR=$(git branch --show-current)

# 2. Mode selection: commit hash > commit count > source branch
if [ -n "$COMMIT_HASH_RAW" ]; then
    # ── Commit hash mode ──
    if ! git cat-file -e "${COMMIT_HASH_RAW}^{commit}" 2>/dev/null; then
        echo "BLOCK: Commit $COMMIT_HASH_RAW not found. Aborting."
        exit 0
    fi
    FULL_HASH=$(git rev-parse "$COMMIT_HASH_RAW")
    DIFF_REF="$FULL_HASH"
    BRANCHES_LABEL="${COMMIT_HASH_RAW} -> HEAD (${C_BR})"
elif [ -n "$COMMIT_COUNT_RAW" ] && echo "$COMMIT_COUNT_RAW" | grep -qE '^[0-9]+$' && [ "$COMMIT_COUNT_RAW" -gt 0 ]; then
    # ── Commit count mode ──
    ACTUAL_COUNT=$(git rev-list --count HEAD)
    if [ "$COMMIT_COUNT_RAW" -gt "$ACTUAL_COUNT" ]; then
        echo "BLOCK: Total commits ($ACTUAL_COUNT) is less than requested $COMMIT_COUNT_RAW. Aborting."
        exit 0
    fi
    DIFF_REF="HEAD~${COMMIT_COUNT_RAW}"
    BRANCHES_LABEL="Last ${COMMIT_COUNT_RAW} commit(s) (${C_BR})"
else
    # ── Source branch mode (original logic) ──
    O_BR=$(git reflog show "$C_BR" | awk '/Created from/ {print $NF; exit}')
    if [ -z "$O_BR" ]; then
        echo "BLOCK: Source branch not found. Aborting."
        exit 0
    fi
    O_BR_COMPARE="${O_BR#remotes/}"
    O_BR_COMPARE="${O_BR_COMPARE#origin/}"
    if [ "$C_BR" == "$O_BR" ] || [ "$C_BR" == "$O_BR_COMPARE" ]; then
        echo "BLOCK: Current branch ($C_BR) and source branch ($O_BR) are the same logical branch or have no new commits. Aborting."
        exit 0
    fi
    DIFF_REF="$O_BR"
    BRANCHES_LABEL="$O_BR -> $C_BR"
fi

# 3. Get Diff
DIFF_TMP="/tmp/git_diff_raw_$(date +%s).txt"
FINAL_TMP="/tmp/git_diff_final_$(date +%s).txt"

git diff "$DIFF_REF" HEAD > "$DIFF_TMP"

if [ ! -s "$DIFF_TMP" ]; then
    echo "BLOCK: No substantive code changes found in the specified range. Aborting."
    rm -f "$DIFF_TMP"
    exit 0
fi

echo "REPOSITORY_NAME: $REPO_NAME" > "$FINAL_TMP"
echo "TARGET_COMPONENT: $CLEAN_PATH" >> "$FINAL_TMP"
echo "BRANCHES: $BRANCHES_LABEL" >> "$FINAL_TMP"
echo "=========================================" >> "$FINAL_TMP"
cat "$DIFF_TMP" >> "$FINAL_TMP"

cat "$FINAL_TMP"
rm -f "$DIFF_TMP" "$FINAL_TMP"
```

*(Note: to avoid escape conflicts, the closing code fence above has an extra space — remove it in actual use)*

### Step 2: Deep Review & Report Output

After obtaining the diff output from Step 1, **strictly apply the following logic**:

1. If Step 1 output contains `BLOCK:`, immediately end the Code Review and briefly explain the reason to the user (e.g. source branch not found, no valid new commits, etc.).
2. If no block string is present, proceed with the deep Code Review.

**Review requirements**: Skip auto-generated files (e.g. package-lock.json). Focus hard on logic bugs, boundary gaps, readability/naming, performance optimizations, and security vulnerabilities.

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

*(Fill in the following three categories as needed; omit any category with no findings)*

### 🚫 Critical Issues

*List only issues that cause crashes, security vulnerabilities, severe logic errors, or build failures.*

* **Issue**: [Precise description of the defect]

* **Potential Impact**: [Describe consequences, e.g. memory overflow, data leak]

* **Location**: Line L[start] - L[end]

* **Before vs After diff**:

  ```javascript
  // Original branch code
  [Extract original branch code block in full]
  
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

*List all optimization points related to code standards, readability, redundant logic, and best practices.*

* **Suggestion**: [Describe the suggestion, e.g. use Optional Chaining instead of nested if checks]

* **Rationale**: [Explain why this change is better]

* **Location**: Line L[start] - L[end]

* **Before vs After diff**:

  ```javascript
  // Original branch code
  [Extract original branch code block in full]
  
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

* **Before vs After diff**:

  ```javascript
  // Original branch code
  [Extract original branch code block in full]
  
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

* **Overall rating**: [Score 1–10, briefly describe the robustness and cleanliness of this commit]
* **Key risk**: [One sentence summarizing the most critical change to watch]

---

## Constraints

- Output must be in English.
- Be concise and direct. No filler or excessive pleasantries.
