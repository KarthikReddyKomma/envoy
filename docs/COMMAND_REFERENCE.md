# Command Reference Guide

This document captures the commands used for documentation creation, git management, and code formatting in the Envoy project.

## Table of Contents

1. [Documentation Creation](#documentation-creation)
2. [Git Operations](#git-operations)
3. [Code Formatting](#code-formatting)
4. [File Operations](#file-operations)
5. [Quick Reference](#quick-reference)

---

## Documentation Creation

### Finding Documentation Files

```bash
# Count all markdown files in docs directory
find docs -type f -name "*.md" | wc -l

# List all documentation directories
find docs -type d -maxdepth 1 | sort

# Count markdown files in specific directory
ls -1 docs/testing/*.md | wc -l

# List documentation by category
for dir in admin-operations extension-development testing memory-performance debugging deployment quic-udp; do 
  echo "=== docs/$dir ===" 
  ls -1 docs/$dir/*.md | wc -l
done
```

### Viewing Documentation Statistics

```bash
# Total line count across all new documentation
wc -l docs/{admin-operations,extension-development,testing,memory-performance,debugging,deployment,quic-udp}/*.md | tail -1

# Total size of documentation directories
du -ch docs/{admin-operations,extension-development,testing,memory-performance,debugging,deployment,quic-udp} | tail -1

# Human-readable size per directory
du -sh docs/*
```

---

## Git Operations

### Checking Git Status

```bash
# Full status
git status

# Short status (compact view)
git status --short

# Show only modified files
git status --short | grep '^M'

# Show only new (untracked) files
git status --short | grep '^??'

# Show only staged files
git status --short | grep -E '^(A|M)'
```

### Staging Files Selectively

```bash
# Stage all markdown files in docs directory
git add docs/**/*.md

# Stage all markdown files in source directory
git add source/**/*.md

# Stage specific directory
git add docs/admin-operations/*.md

# Stage individual file
git add docs/INDEX.md

# Count staged markdown files
git status --short | grep -E '^(A|M)' | grep '\.md$' | wc -l

# Verify no source files are staged
git status --short | grep -E '^(A|M)' | grep -E '\.(h|cc|bazel)$' | wc -l
```

### Viewing Staged Changes

```bash
# Show what's staged for commit
git diff --cached

# Show staged files and their stats
git diff --cached --stat

# Show staged files and total changes
git diff --cached --stat | tail -1

# Show only staged filenames
git diff --cached --name-only

# Show staged markdown files only
git diff --cached --name-only | grep '\.md$'
```

### Unstaging Files

```bash
# Unstage all files
git reset

# Unstage specific file
git restore --staged <file>

# Unstage all files of a type
git restore --staged "*.h"
git restore --staged "*.cc"

# Unstage entire directory
git restore --staged docs/admin-operations/
```

### Comparing Branches

```bash
# Show files changed from main branch
git diff --name-only main

# Count files changed from main
git diff --name-only main | wc -l

# Show only C++ files changed from main
git diff --name-only main | grep -E '\.(h|cc)$'

# Count C++ files changed from main
git diff --name-only main | grep -E '\.(h|cc)$' | wc -l

# Show markdown files changed from main
git diff --name-only main | grep '\.md$'
```

### Viewing Commit History

```bash
# Recent commits (one line per commit)
git log --oneline -10

# Recent commits with full details
git log -5

# Commits with file changes
git log --stat -5

# Commits on current branch (not in main)
git log main..HEAD --oneline
```

---

## Code Formatting

### Clang-Format Configuration

#### View Current Configuration

```bash
# View .clang-format file
cat .clang-format

# View specific settings
grep 'ColumnLimit' .clang-format
grep 'TabWidth' .clang-format
grep 'IndentWidth' .clang-format
```

#### Current Settings (After Update)

The `.clang-format` file has been configured with:

```yaml
Language: Cpp
ColumnLimit: 300      # Increased from 100
TabWidth: 4           # Set to 4 spaces
UseTab: Never         # Use spaces, not tabs
IndentWidth: 4        # 4 spaces for indentation
```

#### Modify Configuration

```bash
# Edit .clang-format file
vim .clang-format
# or
nano .clang-format

# Verify changes
git diff .clang-format
```

### Running Code Formatter

#### Format Changed Files (Recommended)

```bash
# Format files changed since main branch
./tools/local_fix_format.sh -main

# Format unstaged/uncommitted files only
./tools/local_fix_format.sh

# Format with verbose output
./tools/local_fix_format.sh -verbose -main
```

#### Format Specific Files

```bash
# Format specific files
./tools/local_fix_format.sh file1.cc file2.h

# Format files in a directory
./tools/local_fix_format.sh source/common/network/*.cc

# Format single file
./tools/local_fix_format.sh source/server/admin/admin.cc
```

#### Format All Files (Slow)

```bash
# Format entire repository (WARNING: very slow)
./tools/local_fix_format.sh -all
```

#### Using Docker for Formatting

```bash
# Run formatter using Docker (useful on Windows/WSL)
./tools/local_fix_format.sh -docker -main

# Docker with verbose output
./tools/local_fix_format.sh -docker -verbose -main
```

#### Skip Bazel (Use Local Tools)

```bash
# Use local clang-format instead of Bazel
./tools/local_fix_format.sh -skip-bazel -main

# Requires environment variables:
export CLANG_FORMAT_BIN=/path/to/clang-format
export BUILDIFIER_BIN=/path/to/buildifier
export BUILDOZER_BIN=/path/to/buildozer
```

### Manual Clang-Format

```bash
# Format single file directly
clang-format -i source/server/admin/admin.cc

# Format multiple files
clang-format -i source/common/network/*.cc

# Preview formatting without modifying (dry run)
clang-format source/server/admin/admin.cc | diff source/server/admin/admin.cc -

# Format and show diff
clang-format source/server/admin/admin.cc | diff -u source/server/admin/admin.cc -
```

### Check Formatting Without Fixing

```bash
# Check format using bazel
bazel run //tools/code_format:check_format -- check

# Check specific files
bazel run //tools/code_format:check_format check file1.cc file2.h
```

---

## File Operations

### Finding Files

```bash
# Find all header files
find source -name "*.h" -type f

# Find specific pattern in filenames
find source/extensions -name "*admin*" -type f

# Find files modified recently
find . -name "*.md" -mtime -1

# Find large files
find . -size +1M -type f

# Find by extension
find . -name "*.cc" -o -name "*.h" | head -20
```

### Counting Files

```bash
# Count all C++ files
find source -name "*.cc" | wc -l
find source -name "*.h" | wc -l

# Count markdown files
find docs -name "*.md" | wc -l

# Count files in directory
ls -1 docs/testing/ | wc -l

# Count by type
find . -name "*.cc" | wc -l && find . -name "*.h" | wc -l
```

### Creating Directories

```bash
# Create single directory
mkdir docs/new-section

# Create nested directories
mkdir -p docs/new-section/subsection

# Create multiple directories at once
mkdir -p docs/{admin-ops,testing,deployment}
```

### File Content Search

```bash
# Search for text in files
grep -r "search_term" source/

# Search in specific file types
grep -r "search_term" --include="*.h" source/

# Search and show line numbers
grep -rn "search_term" source/

# Search for pattern in files
grep -r "addHandler\|addStreamingHandler" source/server/admin/*.cc

# Case-insensitive search
grep -ri "admin" source/

# Search and show only filenames
grep -rl "search_term" source/
```

---

## Quick Reference

### Common Workflows

#### 1. Stage Only Markdown Files

```bash
# Stage all documentation
git add docs/**/*.md
git add source/**/*.md

# Verify staging
git status --short | grep -E '^(A|M)'

# Verify no source files staged
git status --short | grep -E '^(A|M)' | grep -E '\.(h|cc|bazel)$' | wc -l
# Should output: 0
```

#### 2. Format C++ Code

```bash
# Update .clang-format with your settings
# Edit ColumnLimit, TabWidth, IndentWidth as needed

# Format all changed files
./tools/local_fix_format.sh -main

# Check results
git status --short
```

#### 3. Selective Commit Workflow

```bash
# 1. Check what changed
git status

# 2. Stage only markdown documentation
git add docs/**/*.md source/**/*.md

# 3. Verify staging
git diff --cached --stat

# 4. Commit documentation
git commit -m "Add comprehensive Envoy documentation

- Admin interface and operations
- Extension development guides
- Testing infrastructure
- Memory and performance
- Debugging and error handling
- Deployment patterns
- QUIC/UDP implementation"

# 5. Format remaining code changes
./tools/local_fix_format.sh -main

# 6. Stage formatted files
git add .

# 7. Commit formatted code
git commit -m "Format code with updated .clang-format settings"
```

#### 4. Review Before Commit

```bash
# See what will be committed
git diff --cached

# See stats (insertions/deletions)
git diff --cached --stat

# See only filenames
git diff --cached --name-only

# Review specific file
git diff --cached docs/INDEX.md
```

#### 5. Working with Branches

```bash
# Check current branch
git branch

# Compare with main
git diff main --stat

# See commits ahead of main
git log main..HEAD --oneline

# Create new branch
git checkout -b feature/new-docs

# Switch branches
git checkout main
git checkout add_docs
```

---

## Environment and Tools

### Check Tool Versions

```bash
# Git version
git --version

# Clang-format version
clang-format --version

# Bazel version
bazel --version

# Python version (for scripts)
python3 --version
```

### Bazel Commands

```bash
# Run format checker
bazel run //tools/code_format:check_format

# Run format fixer
bazel run //tools/code_format:check_format fix

# Run spelling checker
bazel run //tools/spelling:check_spelling_pedantic

# Clean bazel cache
bazel clean
```

---

## Tips and Best Practices

### Git Tips

1. **Use `git status --short`** for cleaner output
2. **Use `git diff --cached`** to review before commit
3. **Stage incrementally** with `git add` rather than `git add -A`
4. **Check staged files count** before committing large changes
5. **Use `git diff --stat`** for quick overview of changes

### Formatting Tips

1. **Format before commit**: Run formatter before each commit
2. **Use `-main` flag**: Format only changed files for speed
3. **Test with verbose**: Use `-verbose` to see what's being formatted
4. **Backup important changes**: Commit before formatting entire repo
5. **Verify settings**: Check `.clang-format` matches your preferences

### Documentation Tips

1. **Count files first**: Use `find | wc -l` to know scope
2. **Check line counts**: Use `wc -l` to verify completeness
3. **Use `du -h`**: Check documentation size
4. **Verify links**: Ensure internal documentation links work
5. **Stage separately**: Keep docs and code in separate commits

---

## Troubleshooting

### Formatting Issues

**Problem**: Formatter fails or produces unexpected results

```bash
# Check clang-format version
clang-format --version

# Verify .clang-format exists and is readable
cat .clang-format

# Run with verbose to see errors
./tools/local_fix_format.sh -verbose -main

# Use Docker if local tools mismatch
./tools/local_fix_format.sh -docker -main
```

**Problem**: Bazel errors

```bash
# Clean bazel cache
bazel clean

# Run bazel sync
bazel sync

# Check bazel version
bazel --version
```

### Git Issues

**Problem**: Too many files staged

```bash
# Unstage all
git reset

# Re-stage selectively
git add docs/**/*.md
```

**Problem**: Need to undo last commit

```bash
# Undo commit but keep changes
git reset --soft HEAD~1

# Undo commit and changes (CAREFUL!)
git reset --hard HEAD~1
```

**Problem**: Wrong files committed

```bash
# Amend last commit (if not pushed)
git add <correct-files>
git commit --amend

# Or create new commit to fix
git add <correct-files>
git commit -m "Fix: Add missing files"
```

---

## Additional Resources

### Envoy Documentation Structure

```
docs/
├── admin-operations/      # Admin interface, runtime config, operations
├── extension-development/ # HTTP/network/listener filter development
├── testing/               # Unit, integration, benchmark testing
├── memory-performance/    # Threading, buffers, optimization
├── debugging/             # Error handling, logging, debugging
├── deployment/            # Deployment patterns, K8s, hardening
├── quic-udp/              # QUIC/HTTP3, UDP listeners
├── filter-chain/          # Filter chain architecture
├── request-flow/          # Request processing flow
├── flows/                 # UML and flow diagrams
├── http_filters/          # HTTP filter documentation
├── istio-envoy-config/    # Istio integration
├── access-logs/           # Access logging
├── security/              # TLS, SDS, RBAC
└── INDEX.md               # Master documentation index
```

### File Naming Conventions

- Use lowercase with hyphens: `admin-interface.md`
- Number documents in series: `01-overview.md`, `02-details.md`
- Include README.md in each directory
- Use descriptive names: `error-handling-patterns.md`

### Commit Message Guidelines

```
# Format
<type>: <short summary>

<detailed description>

# Types
- docs: Documentation changes
- feat: New feature
- fix: Bug fix
- refactor: Code refactoring
- test: Test changes
- chore: Build/tooling changes

# Examples
docs: Add comprehensive extension development guide

- HTTP filter development patterns
- Network filter implementation
- Listener filter usage
- Bazel integration examples

format: Update .clang-format with custom settings

- Increase column limit to 300
- Set tab width to 4 spaces
- Format all source files
```

---

## Summary of Session

### What Was Created

- **36 documentation files** across 7 new categories
- **~28,000 lines** of comprehensive documentation
- **788 KB** of content
- Updated **INDEX.md** with all new documentation

### What Was Staged

- **43 markdown files** (docs + source)
- **35,555 insertions**
- **0 source files** (only documentation)

### What Was Configured

- **.clang-format** updated with:
  - ColumnLimit: 300
  - TabWidth: 4
  - IndentWidth: 4
  - UseTab: Never

### What's Running

- **Code formatter** running in background on 712 C++ files

---

## Last Updated

This document was created on: 2026-04-26
Branch: `add_docs`
