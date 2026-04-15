# Gemini+PR Large Repo Limitation

> Date: 2026-04-08
> Workflow: https://n8n.zigbang.in/workflow/coJnXZrkq4CE4yp8 (a_at_pr)
> Target Repo: zigbang/zigbang-client (master)

---

## 1. Issue

Gemini+PR workflow failed at commit step with `nothing to commit` error.
Gemini was unable to identify and modify the target files.

## 2. Root Cause

### 2.1 File Discovery Limitation

Current file listing command:
```bash
find . -maxdepth 3 -type f -not -path "./.git/*" | head -50
```

zigbang-client repo has **4,846 files**. The above command only returns the first 50 files (mostly from `/patches` directory), missing the actual target files entirely.

### 2.2 Gemini Response

```
"수정할 파일이 없으므로, 수정된 파일 내용 없이 응답합니다."
```

Gemini could not find the relevant files from the limited file list, so it returned no modifications.

### 2.3 Execution Flow

| Step | Result |
|------|--------|
| 2.2 git clone | OK (4,846 files) |
| 3.1 File listing | 50 files returned (patches only) |
| 3.2 Gemini | "No files to modify" |
| 3.3 Apply | Modified 0 files |
| 3.4 Diff | (empty) |
| 4.2 Commit | **FAIL: nothing to commit** |

## 3. Claude Code vs Gemini Approach

| | Claude Code CLI | Gemini (n8n) |
|---|---|---|
| File Access | Direct filesystem access (`Read`, `Glob`, `Grep`) | Text-only (file list as string input) |
| File Discovery | `Glob("**/*.tsx")`, `Grep("keyword")` autonomously | Depends on pre-defined `find` command |
| Code Modification | Direct file write (`Edit`, `Write`) | Returns text → parse → write (fragile) |
| Large Repo | Handles well (incremental search) | Limited by prompt size and file listing |
| Autonomy | Multi-turn, self-directed exploration | Single-shot, depends on given context |

### Key Difference

Claude Code CLI runs **inside the filesystem** and can autonomously explore, search, and modify files across the entire repo.

Gemini receives only a **text summary** of the repo and must infer everything from that limited context. For large repos (4,000+ files), this approach breaks down because:

1. File listing is truncated (50/4,846 files)
2. No way to search file contents (grep)
3. No iterative exploration (single-shot)
4. Prompt size limit prevents sending full file tree

## 4. Possible Improvements

### 4.1 Targeted File Discovery

Replace generic `find` with specific path search based on user input:

```bash
# Use grep to find relevant files
cd /tmp/n8n-repo && grep -rl "keyword" --include="*.tsx" --include="*.ts" | head -20
```

### 4.2 Two-Phase Approach

1. **Phase 1**: Ask Gemini to identify file paths based on the task description
2. **Phase 2**: Read those files and ask Gemini to modify them

### 4.3 Scope Limitation

Add a `target_path` field to the form so users can specify the directory:

```
target_path: packages/home-screens/src/screens/ZigbangHomeScreen
```

### 4.4 Use Claude Code CLI Instead

For large repos, Claude Code CLI is fundamentally better suited because it has direct filesystem access and can autonomously search and modify files.

## 5. Conclusion

Gemini+PR workflow works well for **small repos** (like zigbang/n8n with 4 files) but is not suitable for **large repos** (like zigbang-client with 4,846 files) without significant improvements to the file discovery and context-building steps.

| Repo | Files | Gemini+PR | Claude+PR |
|------|-------|-----------|-----------|
| zigbang/n8n | ~4 | OK | OK |
| zigbang/zigbang-client | 4,846 | **FAIL** | OK |
