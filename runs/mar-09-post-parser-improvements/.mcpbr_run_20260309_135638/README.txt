This directory contains the complete output from an mcpbr evaluation run.

Started: 2026-03-09 13:56:38
Config: supermodel-deadcode-pr.yaml
Benchmark: supermodel
Model: claude-sonnet-4-20250514
Provider: anthropic

Files:
- config.yaml: Configuration used for this run
- evaluation_state.json: Per-task results and state
- logs/: Detailed execution traces (MCP server logs)

To analyze results:
  mcpbr state --state-dir .mcpbr_run_20260309_135638

To archive:
  tar -czf results.tar.gz .mcpbr_run_20260309_135638
