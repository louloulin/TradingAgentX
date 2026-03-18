# Memories

## Patterns

### mem-1773832646-588c
> Docs in docs/ currently use the *.zh-CN.md suffix for Chinese design and implementation documents.
<!-- tags: docs, documentation | created: 2026-03-18 -->

## Decisions

## Fixes

### mem-1773837811-231c
> failure: cmd=rg -n "^## 3\.|^## 4\." docs/codebase-analysis.zh-CN.md, exit=1, error=rg returned no match because the chapter headings in this document do not use that exact pattern, next=use a broader heading query with '|| true' before inferring document state
<!-- tags: docs, tooling, rg, error-handling | created: 2026-03-18 -->

### mem-1773836217-ca8f
> failure: cmd=rg --files | rg '(^|/)(docker-compose|compose|helm|charts|k8s|kubernetes)(\.|/|$)|(^|/)\.env\.example$|(^|/)\.env.example$', exit=1, error=rg returns exit 1 for a no-match negative check, next=append '|| true' or use an existence-specific command when verifying absence
<!-- tags: tooling, error-handling, rg | created: 2026-03-18 -->

### mem-1773836217-3dab
> failure: cmd=python - <<'PY' ... , exit=1, error=zsh can't create temp file for here document in this environment, next=use python -c or a quoted one-liner instead of here-doc for ad hoc checks
<!-- tags: tooling, error-handling, zsh | created: 2026-03-18 -->

### mem-1773835668-70c2
> failure: cmd=sed -n '1,220p' sh/*, exit=1, error=zsh glob failed because top-level sh is a file path rather than a directory of scripts, next=inspect the sh path directly before mentioning it in documentation
<!-- tags: docs, tooling, error-handling | created: 2026-03-18 -->

### mem-1773835608-7326
> failure: cmd=rg --files tests test | sort, exit=2, error=rg returned non-zero because the repo has tests/ but no test/ directory, next=target the real tests/ directory directly when selecting verification commands
<!-- tags: docs, testing, error-handling | created: 2026-03-18 -->

### mem-1773835534-14f8
> failure: cmd=rg --files cli api .github | sort, exit=2, error=rg returned non-zero because the repo currently has no top-level cli/ directory although pyproject still declares a tradingagents CLI script, next=verify the actual runtime entrypoints from api/ and document the packaging mismatch explicitly
<!-- tags: docs, tooling, error-handling | created: 2026-03-18 -->

### mem-1773835509-8af8
> failure: cmd=rg --files tradingagents llm_clients config scripts deploy docker .github docs | sed 's#^#/#', exit=2, error=rg returned non-zero because llm_clients/config/scripts/deploy/docker are not top-level directories in this repo, next=search repo-root files directly and use actual paths like tradingagents/llm_clients plus workflow assets
<!-- tags: docs, tooling, error-handling | created: 2026-03-18 -->

### mem-1773835365-9882
> failure: cmd=sed -n '1,220p' tradingagents/agents/risk_mgmt/risk_manager.py, exit=1, error=sed: tradingagents/agents/risk_mgmt/risk_manager.py: No such file or directory, next=locate the actual risk judge module with rg --files and read tradingagents/agents/managers/risk_manager.py instead
<!-- tags: docs, review, error-handling | created: 2026-03-18 -->

### mem-1773834753-1561
> failure: cmd=git check-ignore -v docs/codebase-analysis.zh-CN.md .ralph/agent/scratchpad.md, exit=1, error=no output because the paths are not ignored, next=treat exit 1 as a negative check result and proceed with explicit git add for the target files
<!-- tags: git, error-handling, docs | created: 2026-03-18 -->

### mem-1773833895-1095
> failure: cmd=test -f .ralph/agent/scratchpad.md && sed -n '1,220p' .ralph/agent/scratchpad.md, exit=1, error=scratchpad file missing, next=create .ralph/agent/scratchpad.md before appending iteration notes
<!-- tags: tooling, ralph, error-handling | created: 2026-03-18 -->

### mem-1773833340-347d
> failure: cmd=git add docs/codebase-analysis.zh-CN.md && git commit -m "docs: correct chapter 1 report persistence wording", exit=128, error=fatal: Unable to create '.git/index.lock': File exists., next=check for active git process holding the lock and remove stale .git/index.lock before retrying commit
<!-- tags: git, error-handling, docs | created: 2026-03-18 -->

### mem-1773833047-8800
> failure: cmd=git add docs/codebase-analysis.zh-CN.md && git diff --cached -- docs/codebase-analysis.zh-CN.md, exit=1, error=documentation file under docs/ is ignored by repository gitignore rules, next=inspect ignore source with git check-ignore -v and use git add -f for this generated analysis document only
<!-- tags: docs, git, error-handling | created: 2026-03-18 -->

### mem-1773833020-0fb6
> failure: cmd=uv run pytest -q tests/test_api_smoke.py, exit=1, error=8 API smoke tests fail because /v1/auth/request-code returns Internal Server Error and test setup cannot parse dev_code JSON, next=inspect auth request-code path in api.main and auth service before using test_api_smoke.py as a verification gate
<!-- tags: testing, api, auth, error-handling | created: 2026-03-18 -->

### mem-1773832781-c43b
> failure: cmd=$RALPH_BIN tools task start task-1773832646-2713, exit=2, error=error: unrecognized subcommand 'start', next=use supported runtime task commands (list/ready/close/fail/show) and treat injected ready task as active work item
<!-- tags: tooling, error-handling, ralph | created: 2026-03-18 -->

### mem-1773832769-4327
> failure: cmd=test -f docs/codebase-analysis.zh-CN.md && sed -n '1,240p' docs/codebase-analysis.zh-CN.md, exit=1, error=target documentation file does not exist yet, next=create docs/codebase-analysis.zh-CN.md for chapter-based codebase analysis
<!-- tags: docs, documentation, error-handling | created: 2026-03-18 -->

### mem-1773832619-a28d
> failure: cmd=/Users/louloulin/.cargo/bin/ralph tools task ensure "编写章节1：项目概览与仓库结构" --key docs:chapter-01-overview -p 1 -d "为 docs/codebase-analysis.zh-CN.md 撰写第1章，说明项目目标、整体能力、仓库目录结构与后续章节导览" --format quiet, exit=2, error=error: unrecognized subcommand 'ensure', next=inspect '/Users/louloulin/.cargo/bin/ralph tools task --help' and use supported task subcommands
<!-- tags: tooling, error-handling, ralph | created: 2026-03-18 -->

### mem-1773832600-1b49
> failure: cmd=sed -n '1,220p' .ralph/agent/decisions.md, exit=1, error=sed: .ralph/agent/decisions.md: No such file or directory, next=create decisions.md only when a <=80 confidence decision must be recorded
<!-- tags: tooling, error-handling, ralph | created: 2026-03-18 -->

## Context

### mem-1773832646-59f9
> Current Ralph runtime task CLI in this repo supports add/list/ready/close/fail/show; ensure/start from the skill summary are not implemented, so coordination should use add plus task list/ready.
<!-- tags: ralph, tooling | created: 2026-03-18 -->
