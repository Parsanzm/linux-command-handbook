# diff — Practical Examples

> Real-world patterns for code review, patching, backups, and automated testing.

---

## Table of Contents

- [Basic Comparisons](#basic-comparisons)
- [Unified Diff for Code Review](#unified-diff-for-code-review)
- [Ignoring Whitespace and Case](#ignoring-whitespace-and-case)
- [Quick Yes/No Comparisons](#quick-yesno-comparisons)
- [Comparing Directories](#comparing-directories)
- [Creating and Applying Patches](#creating-and-applying-patches)
- [Side-by-Side Comparison](#side-by-side-comparison)
- [Using diff in Scripts and Tests](#using-diff-in-scripts-and-tests)
- [Combining diff with Version Control](#combining-diff-with-version-control)
- [Real-World Recipes](#real-world-recipes)

---

## Basic Comparisons

```bash
# Basic comparison, default (normal) format
diff old_version.txt new_version.txt

# Compare two config files
diff /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak

# Compare a file against itself after editing
cp important.conf important.conf.orig
vim important.conf
diff important.conf.orig important.conf
```

---

## Unified Diff for Code Review

```bash
# The most common, human-friendly format
diff -u old_code.py new_code.py

# With extra context lines for better readability
diff -u5 old_code.py new_code.py

# Zero context — just the exact changed lines, no surrounding text
diff -u0 old_code.py new_code.py

# Colorized unified diff, piped through a pager for easy reading
diff --color=always -u old_code.py new_code.py | less -R
```

---

## Ignoring Whitespace and Case

```bash
# Ignore ALL whitespace differences (useful when only indentation changed)
diff -w script_v1.py script_v2.py

# Ignore differences in the AMOUNT of whitespace, but not its presence entirely
diff -b script_v1.py script_v2.py

# Ignore case differences (useful for comparing case-inconsistent data exports)
diff -i names_v1.txt names_v2.txt

# Ignore blank-line-only changes (useful when reformatting added/removed
# empty lines without touching actual content)
diff -B file1.txt file2.txt

# Combine multiple ignore flags at once
diff -wiB old.txt new.txt
```

---

## Quick Yes/No Comparisons

```bash
# Just report whether files differ, without showing HOW
diff -q file1.txt file2.txt
# Files file1.txt and file2.txt differ
# (or nothing printed at all, if they're identical)

# Explicitly confirm when files ARE identical too
diff -s file1.txt file2.txt
# Files file1.txt and file2.txt are identical

# Use the exit status directly in a conditional, without printing anything
if diff -q file1.txt file2.txt > /dev/null; then
  echo "Same"
else
  echo "Different"
fi
```

---

## Comparing Directories

```bash
# Recursively compare two directory trees
diff -r project_v1/ project_v2/

# Quick summary: just WHICH files differ, not the full diffs
diff -rq project_v1/ project_v2/

# Treat missing files as empty (shows a FULL diff for added/removed files,
# rather than just "Only in X: filename")
diff -rN project_v1/ project_v2/

# Exclude specific patterns from the comparison
diff -r --exclude="*.pyc" --exclude="__pycache__" --exclude=".git" project_v1/ project_v2/

# Compare directories, showing unified-style diffs for each differing file
diff -ruN project_v1/ project_v2/
```

---

## Creating and Applying Patches

```bash
# Generate a patch file
diff -u original.py modified.py > changes.patch

# Apply the patch to a fresh copy of the original
cp original.py test_copy.py
patch test_copy.py < changes.patch
diff test_copy.py modified.py    # should show NO differences now

# Generate a patch for an entire directory tree
diff -ruN project_original/ project_modified/ > full_project.patch

# Apply a directory-wide patch (from within the target directory)
cd project_original/
patch -p1 < ../full_project.patch

# Preview a patch's effect before committing to it
patch --dry-run -p1 < full_project.patch

# Reverse-apply a patch (undo changes it previously applied)
patch -R original.py < changes.patch
```

---

## Side-by-Side Comparison

```bash
# Two-column visual comparison
diff -y file1.txt file2.txt

# Wider output for long lines (default width is often too narrow)
diff -y --width=150 file1.txt file2.txt

# Show ONLY the differing lines, hiding identical ones (cleaner review)
diff -y --suppress-common-lines file1.txt file2.txt

# Combine with less for scrolling through a long side-by-side diff
diff -y --width=200 file1.txt file2.txt | less
```

---

## Using diff in Scripts and Tests

```bash
# Classic "does the output match expected?" test pattern
./generate_report.sh > actual_output.txt
if diff -q actual_output.txt expected_output.txt > /dev/null; then
  echo "PASS"
else
  echo "FAIL — differences:"
  diff -u expected_output.txt actual_output.txt
  exit 1
fi

# Verify a config file hasn't drifted from a known-good template
if ! diff -q /etc/app/config.yaml /etc/app/config.yaml.reference > /dev/null; then
  echo "WARNING: config has diverged from reference template"
fi

# Regression testing: compare current output against a saved "golden" file
./run_analysis.sh > current_results.txt
diff -u golden_results.txt current_results.txt || {
  echo "Results changed! Review the diff above before proceeding."
  exit 1
}

# Detect if a deployment actually changed anything
diff -rq /srv/app/current /srv/app/staged > deployment_diff.txt
if [ -s deployment_diff.txt ]; then
  echo "Changes detected, proceeding with deployment"
  cat deployment_diff.txt
else
  echo "No changes — skipping deployment"
fi
```

---

## Combining diff with Version Control

```bash
# git diff is built on the same underlying concepts as standalone diff
git diff HEAD~1 HEAD -- myfile.py

# Compare a file's current state against a specific commit using
# plain diff (after checking out a copy for comparison)
git show HEAD~3:myfile.py > old_version.py
diff -u old_version.py myfile.py

# Compare two branches' versions of a file
git show main:config.yaml > main_config.yaml
git show feature-branch:config.yaml > feature_config.yaml
diff -u main_config.yaml feature_config.yaml

# Use diff to review changes BEFORE committing (equivalent-ish to git diff)
diff -u <(git show HEAD:src/app.py) src/app.py
```

---

## Real-World Recipes

```bash
# --- Verifying a Config Change Before Deploying ---

diff -u /etc/nginx/nginx.conf /tmp/new_nginx.conf
read -p "Apply this change? [y/N] " confirm
[ "$confirm" = "y" ] && cp /tmp/new_nginx.conf /etc/nginx/nginx.conf

# --- Comparing Two Server's Configuration ---

ssh server1 "cat /etc/app/config.yaml" > server1_config.yaml
ssh server2 "cat /etc/app/config.yaml" > server2_config.yaml
diff -u server1_config.yaml server2_config.yaml

# --- Auditing Changes Between Two Backup Snapshots ---

diff -rq /backups/2024-01-01/ /backups/2024-02-01/ | grep -v "^Only in"
# Shows files that CHANGED between snapshots, filtering out purely
# added/removed file noise for a focused "what actually changed" view

# --- CI Pipeline: Detecting Unintended Formatting Changes ---

./format_code.sh
if ! diff -q --exclude=".git" -r src/ src_formatted_reference/ > /dev/null; then
  echo "Formatting changes detected — review before merging"
  diff -ru src/ src_formatted_reference/
fi

# --- Creating a Minimal Bugfix Patch to Share ---

cp buggy_module.py buggy_module.py.orig
# ... make the fix ...
diff -u buggy_module.py.orig buggy_module.py > bugfix.patch
# share bugfix.patch with the team, or attach to a ticket
```

---

> See also: [`README.md`](README.md) · [`edge-cases.md`](edge-cases.md) · [`interview-questions.md`](interview-questions.md)
