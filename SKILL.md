---
name: credential-exposure-map
description: Map all credentials your OpenClaw agent can access. Scans env vars, config, memory, skills, MCP servers, git history. Generates exposure report with risk scoring. Activate when user says "credential audit", "exposure map", "security audit", or "what can my agent access".
version: 1.1.0
---

# Credential Exposure Map

> See every credential your agent can touch. Score the blast radius. Shrink the attack surface.


## Onboarding（首次使用）

当用户第一次触发本 skill 且 `memory/credential_exposure_map_state.json` 不存在时，执行一次性引导：

1. **介绍能力**（一句话）：_"我可以扫描你的 agent 能访问的所有凭证（env/config/memory/git history），生成风险评分报告。"_

2. **快速配置：**

1. **扫描范围**（默认：全部）
   - 全部（env + config + memory + skills + MCP + git history）
   - 仅环境变量和配置文件
   - 仅 git history
2. **风险阈值**（默认：中）
   - 低：列出所有暴露（包括安全的）
   - 中：中风险和高风险（推荐）
   - 高：仅高风险（可能遗漏）
3. **是否保存上次扫描结果**（默认：是，用于下次对比新增暴露）

3. **写入初始状态文件** `memory/credential_exposure_map_state.json`（JSON 格式，含 `onboarded: true` 和时间戳）

4. **确认完成**：_"设置完成！以后直接用就行。随时可以说'reset settings'重新配置。"_

**Onboarding 只执行一次**，后续每次激活直接读取状态文件。用户说 "reset settings" 时删除状态文件重新引导。

---

## When to Activate

- User says: "credential audit", "exposure map", "security audit"
- User says: "what can my agent access", "credential scan"
- User says: "credential cleanup", "redact credentials"

Do NOT activate for normal work conversations.

## Audit Flow

When user asks for a credential audit or exposure map:

1. Run the scanner:

```bash
python3 skills/credential-exposure-map/scripts/scan_exposure.py
```

2. Present the output in a clean format:
   - Summary: total findings, risk distribution
   - Credential Inventory: table with preview, risk level, source, location
   - Skill Capability Matrix: which skills have exec/read/network/write
   - Top recommendations for reducing exposure

3. Highlight CRITICAL findings prominently.

## Recommendation Generation

After presenting the scan, offer specific recommendations:

- **CRITICAL credentials in env**: "Move to secrets vault or remove from agent context"
- **Credentials in MEMORY.md**: "These persist across sessions. Consider redacting."
- **Skills with full exec+network**: "Review if this skill needs both capabilities"
- **Secrets in git history**: "Use git filter-branch or BFG to scrub history"
- **MCP servers with auth**: "Verify each MCP server's permission scope"

## Output Format

### Summary

```
Scan Complete: X finding(s)
Risk: N Critical | N High | N Medium | N Low
```

### Credential Inventory

Table showing each credential with:
- Preview (first 8 chars + ***)
- Risk level (CRITICAL/HIGH/MEDIUM/LOW)
- Source (env/config/memory/skill/mcp/git)
- Location (file:line or config path)

### Skill Capability Matrix

Table showing each installed skill with exec/read/network/write permissions and risk score.

## Security Considerations

- NEVER output full credential values. Always mask as `first8chars***`
- Report is saved to `~/.openclaw/credential-exposure-report.json` with 600 permissions
- Scan is read-only, does not modify any files
- Skill capability inference is conservative (assumes exec = full filesystem access)

## Detection Coverage

| Source | What It Finds |
|--------|--------------|
| Environment variables | API keys, tokens, passwords in process.env |
| openclaw.json | Provider keys, webhook secrets, MCP auth |
| .env files | Any .env in workspace or ~/.openclaw |
| MEMORY.md / memory/*.md | Credentials written into agent memory |
| memory/*.json | Credential caches, state files with tokens |
| Installed skills | Skills with exec, read, or network capabilities |
| MCP server config | Connected services with auth scope |
| Git history | Secrets committed in last 50 commits |
