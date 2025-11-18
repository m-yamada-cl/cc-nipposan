---
description: プロジェクト全体の日報を生成
allowed-tools: Read, Bash, Task, Write, Glob, WebSearch
---

# All Daily Reports

Generate daily reports for all dates that have Claude Code sessions in this project.

## Task

### 1. Find Project Directory

**Session file location**: `~/.claude/projects/`

- Get current working directory
- List all directories in `~/.claude/projects/`
- Find the directory whose name contains the current working directory path
- Example: If cwd is `/Users/name/dev/my-project`, look for a directory like `my-project-abc123def456`

### 2. Extract All Session Dates

Use bash to extract all unique dates:

```bash
# Extract unique dates from all session files
find ~/.claude/projects/{PROJECT_DIR} -name "*.jsonl" -exec head -100 {} \; | grep -o '"timestamp":"[^"]*"' | sed 's/"timestamp":"//g; s/"//g' | cut -d'T' -f1 | sort -u
```

Store the output to determine which dates need processing.

### 3. Filter Out Finalized Dates

**Check for 確定版 (finalized reports)**:

For each date extracted in step 2:
- Check if `~/Documents/nipposan/{project-name}/daily/YYYY-MM-DD.md` exists
- If exists: exclude this date from processing (already finalized)
- If not exists: include this date for processing

**Result**: A filtered list of dates that need processing (dates without 確定版).

### 4. Prepare for Subagent Launch

**IMPORTANT - Language Setting**:
- すべての出力は日本語で行うこと
- Think in English, but generate responses in Japanese

**Template path reference** (subagents will load this themselves):
- Primary path: `~/.claude/commands/nipposan/templates/generate-report-prompt.txt`
- Fallback path: `./nipposan/templates/generate-report-prompt.txt` (when run from project repository)
- Subagents will try primary path first, then fallback if not found
- No need to load the template in this step - subagents will handle template loading

### 5. Split Dates into Groups

**Grouping strategy based on filtered date count**:
- 1-3 dates → 1 group (1 subagent, all dates in one group)
- 4-6 dates → 2 groups (2 subagents, ~3 dates each)
- 7-9 dates → 3 groups (3 subagents, ~3 dates each)
- 10-15 dates → 4 groups (4 subagents, distribute evenly)
- 16+ dates → 5 groups (5 subagents, distribute evenly, maximum)

Each subagent processes multiple dates sequentially within its assigned group.

**Display initial message**:
```
処理中: {filtered_count}件の日付を{num_groups}グループで並列処理 ({skipped_count}件は確定版のためスキップ、全{total}件中)...
```

### 6. Launch Parallel Subagents

**IMPORTANT**: Use Task tool to launch one general-purpose subagent for each group. Launch all subagents in parallel using multiple Task tool calls in a single message.

**For each group, construct a concise prompt**:

```
Process the following dates and generate daily reports for project: {project-name}

Dates to process:
- YYYY-MM-DD
- YYYY-MM-DD
- YYYY-MM-DD

Instructions:
1. Read the report generation template:
   - Primary path: ~/.claude/commands/nipposan/templates/generate-report-prompt.txt
   - Fallback path: ./nipposan/templates/generate-report-prompt.txt
2. Follow the template instructions exactly for each date
3. すべての出力は日本語で行うこと (Think in English, but generate responses in Japanese)
4. After processing all dates, report back with:

GROUP REPORT:
- Group ID: {group_number}
- Dates processed: {count}
- Processed successfully: {processed_count}
- Skipped (no data): {skipped_no_data}

Do NOT output individual progress messages during processing. Only provide the final GROUP REPORT.
```

**Note**: Replace `{project-name}`, `{group_number}`, and date list with actual values for each subagent.

### 7. Aggregate Results

After all subagents complete:

1. **Collect results from each subagent**:
   - Parse each subagent's GROUP REPORT
   - Extract counts for each category

2. **Calculate totals**:
   - Total processed = sum of all "Processed successfully"
   - Total skipped (no data) = sum of all "Skipped (no data)"

3. **Display summary**:

```
✅ 一括処理が完了しました
- 合計日数: {total}件
- 処理済み: {processed}件
- スキップ済み (確定版存在): {finalized_skip}件
- スキップ済み (データなし): {no_data}件
- 使用した並列グループ数: {num_groups}
```

Where:
- `{total}` = total dates found in step 2
- `{processed}` = sum of "Processed successfully" from all groups
- `{finalized_skip}` = count of dates filtered out in step 3
- `{no_data}` = sum of "Skipped (no data)" from all groups
- `{num_groups}` = actual number of subagents used

## Notes

- **Pre-filtering optimization**: Dates with finalized reports (確定版) are filtered out before launching subagents
- **Parallel processing**: This command uses Task tool to process dates in parallel via subagents
- **Dynamic subagent count**: Number of subagents based on date count (max 5)
  - 1-3 dates → 1 subagent
  - 4-6 dates → 2 subagents
  - 7-9 dates → 3 subagents
  - 10-15 dates → 4 subagents
  - 16+ dates → 5 subagents
- **Each subagent**:
  - Processes multiple dates sequentially within its assigned group
  - Uses the embedded template prompt exactly as provided
  - Returns a structured GROUP REPORT
- **File naming rules**:
  - `[DRAFT]YYYY-MM-DD.md`: Generated on the target date itself (tentative)
  - `YYYY-MM-DD.md`: Generated/updated after the target date (finalized)
- **Performance**: Significantly faster than sequential processing, with minimal waste
- **Running multiple times**: Safe to re-run; automatically skips finalized reports
