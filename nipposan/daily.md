---
description: 今日（または指定日）の日報を生成
allowed-tools: Bash, Task, Read, Write, Glob, WebSearch
argument-hint: [--date YYYY-MM-DD]
---

# Daily Report

Analyze all Claude Code sessions for the specified date and generate a comprehensive daily report.

## Task

### 1. Determine Target Date

- If `--date YYYY-MM-DD` is provided: use that date
- Otherwise: use today's date
- Format: YYYY-MM-DD

### 2. Get Project Name

- Get current working directory
- Extract project name (basename of the git repository root)

### 3. Check for Finalized Report

**Check if 確定版 (finalized report) already exists**:
- Path: `~/Documents/nipposan/{project-name}/daily/YYYY-MM-DD.md`
- If exists:
  - Display: "ℹ️ Report already exists (確定版) for YYYY-MM-DD. Skipping."
  - Exit (do not launch subagent)

### 4. Load Report Generation Template

**IMPORTANT - Language Setting**:
- すべての出力は日本語で行うこと
- Think in English, but generate responses in Japanese

**Read the template**:
- Primary path: `~/.claude/commands/nipposan/templates/generate-report-prompt.txt`
- Fallback path: `./nipposan/templates/generate-report-prompt.txt` (when run from project repository)
- Use Read tool to load the full template content
- Try primary path first, then fallback if not found

### 5. Launch Subagent for Report Generation

**IMPORTANT**: Use Task tool with subagent_type="general-purpose" to launch a subagent.

**Construct the prompt** by combining:

1. **Header**:
   ```
   Generate daily report for project: {project-name}, date: YYYY-MM-DD

   IMPORTANT: You MUST use the template provided below exactly as written.
   Do NOT create your own prompt or deviate from these instructions.
   ```

2. **Processing instructions**:
   ```
   Generate the report using this template:
   ```

3. **The full template content from step 4**

4. **Result reporting format**:
   ```
   After generation, report back with exactly this format:

   RESULT: {STATUS}
   PATH: {file_path_if_applicable}

   Where STATUS is one of: PROCESSED, NO_DATA
   - PROCESSED: Report file was created/updated
   - NO_DATA: No sessions or commits found for the date
   ```

**Note**: Replace `{project-name}` and `YYYY-MM-DD` with actual values throughout the prompt.

### 6. Display Result

Parse the subagent's RESULT and display appropriate message:

- **PROCESSED**: "✅ 日報を生成しました: {PATH}"
- **NO_DATA**: "ℹ️ YYYY-MM-DD のセッションまたはコミットが見つかりませんでした。日報生成をスキップします。"

## Notes

- **Pre-check optimization**: 確定版が存在する場合はサブエージェント起動前にスキップ
- This command uses a subagent to minimize context usage in the main session
- The subagent uses the embedded template exactly as provided
- This command processes **ALL sessions** for the day, not just the current one
- ファイル命名規則:
  - `[DRAFT]YYYY-MM-DD.md`: 対象日の当日に生成（暫定版）
  - `YYYY-MM-DD.md`: 翌日以降に生成または更新（確定版）
- Designed to be triggered manually or automatically via hooks
