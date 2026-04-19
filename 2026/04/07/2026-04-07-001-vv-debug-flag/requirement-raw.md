# Requirement

Add `--debug` command-line parameter to the `vv` agent. When enabled, it should:

1. Show detailed AI model input/output information (full prompts sent to the LLM and full responses received).
2. Show detailed tool input/output information (arguments passed to each tool call and results returned).

The flag should apply across all run modes (CLI and HTTP) where applicable, and should not affect default (non-debug) output.
