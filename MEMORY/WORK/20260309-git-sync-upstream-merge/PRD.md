---
task: Sync upstream updates, merge, update docs, push
slug: 20260309-git-sync-upstream-merge
effort: standard
phase: complete
progress: 14/14
mode: interactive
started: 2026-03-09T12:00:00Z
updated: 2026-03-09T12:00:00Z
---

## Context

Sync local fork (afanty2021/ironclaw) with upstream (nearai/ironclaw). Upstream is 20 commits ahead with new features including AWS Bedrock LLM provider, MCP transport abstraction (stdio/UDS), full image support, PID-based gateway lock, and routine improvements. Need to merge changes, update CLAUDE.md documentation to reflect new features, and push to origin.

## Criteria

- [x] ISC-1: Fetch all branches from upstream remote
- [x] ISC-2: Merge upstream/main into local main branch
- [x] ISC-3: Resolve Cargo.lock merge conflict (use upstream version)
- [x] ISC-4: Verify merge completed successfully
- [x] ISC-5: Update LLM Providers section to include AWS Bedrock
- [x] ISC-6: Add MCP transport abstraction note (stdio/UDS)
- [x] ISC-7: Document new image tools (image_gen, image_edit, image_analyze)
- [x] ISC-8: Add LLM_REQUEST_TIMEOUT_SECS to configuration
- [x] ISC-9: Document PID-based gateway lock feature
- [x] ISC-10: Note skill exclude_keywords veto feature
- [x] ISC-11: Verify CLAUDE.md syntax is valid
- [x] ISC-12: Push merged changes to origin/main
- [x] ISC-13: Verify origin reflects upstream changes
- [x] ISC-14: Local main branch tracking origin correctly

## Verification

- **ISC-1**: `git fetch upstream` completed successfully, fetched 20 commits ahead
- **ISC-2**: Merge commit created (77bf5c0)
- **ISC-3**: Cargo.lock conflict resolved using upstream version (AWS dependencies)
- **ISC-4**: `git log --oneline -3` shows merge commit and upstream commits
- **ISC-5**: Added "AWS Bedrock (requires `--features bedrock`)" to Multi-provider LLM feature
- **ISC-6**: Updated limitations section from "Only HTTP transport implemented" to "stdio, UDS, and HTTP transports available"
- **ISC-7**: Added image_gen.rs, image_edit.rs, image_analyze.rs to builtin tools list
- **ISC-8**: Added LLM_REQUEST_TIMEOUT_SECS=120 to configuration section
- **ISC-9**: Updated Web gateway feature to mention "PID-based lock prevents multiple instances"
- **ISC-10**: Added "Veto" step to Selection Pipeline describing exclude_keywords behavior
- **ISC-11**: `git diff CLAUDE.md` shows clean markdown changes
- **ISC-12**: Pushed to origin/main (00b96d2..db31c50)
- **ISC-13**: `git log` shows upstream commits now in local history
- **ISC-14**: `git branch -vv` shows main tracking origin/main
