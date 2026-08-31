# Changelog

All notable changes to the [ibl.ai API skills](https://github.com/iblai/api).

## [Unreleased]

### Changed

- `iblai-api-agent-tool`: note that `available-tools/` now returns the same base-agent tool set for every agent. It used to vary by agent backend; the Google ADK/A2A, OpenAI Assistants and OpenAI Agents SDK backends were retired in DM 4.362.0, so an agent that previously ran on one of them now sees a larger list.
- `iblai-api-agent-llm`: note that `mentor-llms/?mentor_id=` scopes and validates the agent but no longer narrows the provider/model cards to a single provider. That narrowing only applied to the retired, provider-locked backends; every agent now gets the org's full authorized list.

### Added

- `iblai-api-agent-memory`: document the shared **agent knowledge** store (`MentorMemory`) — `GET`/`POST` `…/orgs/{org}/mentors/{mentor}/agent-memories/` and `PATCH`/`DELETE` `…/agent-memories/{memoryId}/` — org+agent-scoped, non-categorized memories injected into every user's chat with the agent.

## [0.2.5] - 2026-08-29

### Documentation

- add shared agent knowledge (MentorMemory) endpoints

## [0.2.4] - 2026-08-26

### Documentation

- document edX user-role catalog and bulk sync

## [0.2.3] - 2026-08-25

### Documentation

- note tenant-admin cross-user access on memsearch-settings

## [0.2.2] - 2026-07-28

### Documentation

- add platform screenshots to the voice-agent walkthrough

## [0.2.1] - 2026-07-28

### Documentation

- add tutorials/ with an end-to-end voice-agent walkthrough

## [0.2.0] - 2026-07-27

### Added

- adding skill for the support
- fold all developer docs into skills — per-skill references/ + infrastructure & ecosystem skills (lossless)
- merge remaining developer docs into skills (code-verified) + references/ split
- add iblai-api-inference and merge developer docs into the skills
- fold MCP configuration into iblai-api-agent-mcp, drop the separate skill
- add iblai-api-apply skill — the platform application gate (code-verified)
- added skill for mcp configuration
- 8 new DM skills (code-verified) + repo-wide format standardization (#9)

### Fixed

- made sure everything merged / created is complete
- bring iblai-api-mcp-configuration onto repo conventions (code-verified)
- rename learner and mentor to user and agent respectively
- revert mcp name change
- add CRM schema

### Documentation

- list iblai-api-mcp-configuration in the README skill index
- document org key + secret as a no-browser auth path
- add "Updating the skills" section to README
- simplify login onboarding to two clear paths
- normalize IBL.ai -> ibl.ai in agent-chat server README; align license to MIT
