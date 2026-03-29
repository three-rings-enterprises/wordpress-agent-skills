# Add Git Version Control to Design Site Workflow

## Summary

Integrate git initialization and milestone-based commits throughout the `design-site` workflow. The project directory is initialized as a git repo at the start, and commits are made at each phase boundary and key milestone — giving durable change history, per-page rollback granularity, and easy diffing between phases.

## Problem

The current workflow has no version control integration. Versioning is handled entirely through filename suffixes (`-v1`, `-v2`) and `gallery.json` state tracking. This works within a single session but provides:
- No durable change history across sessions
- No ability to roll back to a known-good state (e.g., if Phase 5 fidelity fixes go sideways)
- No way to diff between phases (e.g., what changed between approved mockups and the WordPress build)
- No collaboration story

## Changes

All changes are in `claude-code/wp-site-creator/commands/design-site.md`.

### New top-level section: Version Control
- Documents commit message convention: `design(<phase>): <description>`
- Establishes rules: orchestrator owns git (except Phase 5 build agent), never commit verification screenshots, always add specific files

### Git init — Phase 0.5 (Studio Setup)
- `git init` after site directory is created
- Creates `.gitignore` (excludes `design/verification/`, `.warm`, `node_modules/`, OS files, `wp-content/debug.log`)
- Initial commit with `.gitignore` and gallery mu-plugin

### Commit — Phase 1 (Spec Locked)
- Commits `site-spec.md` and any user-supplied images/logos

### Commit — Phase 2 (Design Tokens Locked)
- Commits style tiles, `design-tokens.json`, `design-patterns.html`, and `gallery.json`

### Commit — Phase 3 (Layout Selected)
- Commits page layouts and `gallery.json`

### Commits — Phase 4 (Per-Page Approval)
- **Per-page commits**: each time a single page draft is promoted to `approved/`, it gets its own commit
- This gives per-page rollback granularity — if a later approval reveals issues with an earlier one, you can revert individually
- Draft revisions (`-v2`, `-v3`) during the iteration loop are NOT committed — file versioning handles that

### Commit — Phase 4.5 (Content Extraction)
- Commits all extracted content JSON files after validation

### Commits — Phase 5 (WordPress Build)
- **Initial build commit**: after theme activation and page deployment (made by the build subagent)
- **Per-fidelity-fix commits**: after each fidelity comparison round that requires theme changes (also made by the build subagent)
- The Phase 5 build agent is a single serial agent, so it can safely own its own git commits without race conditions

## Design Decisions

**Why the orchestrator owns git (mostly):** Parallel subagents (Phase 2 tiles) could create race conditions. Centralizing git in the orchestrator avoids this. Phase 5 is the exception — it's a single serial agent, so letting it commit directly is simpler than signaling back to the orchestrator after each fix round.

**Why not commit draft revisions:** The Phase 4 iteration loop is exploratory. Users may create many revisions before approving. The file-based versioning (`-v1`, `-v2`) already provides in-session history, and committing every revision would create noise. The approval promotion is the meaningful checkpoint.

**Why per-page commits instead of batch:** Individual commits per approved page let you `git revert` a single page without losing the rest. More useful than one big "approve all" commit.

## Test Plan

- [ ] Run `/design-site` end-to-end on a new site — verify git repo is initialized and commits appear at each milestone
- [ ] Verify `.gitignore` correctly excludes verification screenshots and temp files
- [ ] Run `git log --oneline` after Phase 5 — confirm commit history reads as a coherent timeline
- [ ] Test Phase 4 partial approval (approve some pages, iterate on others) — verify per-page commits
- [ ] Test Phase 5 fidelity fix loop — verify each fix round produces its own commit
- [ ] Test with an existing git repo in the site directory — verify it skips `git init` and continues
