# AI Agent Instructions for Homebridge Ventair Ceiling Fan Plugin

> Single source of truth for all AI coding assistants working on this repository.

## Project Identity

| Key | Value |
| --- | --- |
| **Name** | Homebridge Ventair Ceiling Fan Plugin |
| **Repository** | `github.com/tinkerologie/homebridge-ventair-ceiling-fan` |
| **Default Branch** | `main` |
| **Description** | Homebridge plugin providing HomeKit integration for Ventair smart ceiling fans. |

## Mandatory Development Workflow for AI Agents

All AI coding assistants (Claude Code, Gemini, GitHub Copilot, Cursor, Roo, Devin, etc.) MUST strictly adhere to the following workflow when working on this repository:

1. Issue-First Requirement: Never start writing code or creating pull requests without an existing GitHub Issue.
2. Issue Template: Issues must be created using the official Issue Forms (`01-bug-report.yml` for bugs or `02-enhancement-request.yml` for enhancements).
3. CodeRabbit AI Plan: Every issue must have a generated Coding Plan from CodeRabbit AI (`@coderabbitai plan`). AI agents must read and implement strictly according to the approved plan.
4. Branch Discipline: Always create and work on a dedicated branch (`feat/<issue-id>-<description>` or `fix/<issue-id>-<description>`). NEVER commit directly to `main`.
5. Documentation & Changelog: Update `CHANGELOG.md` under `## [Unreleased]` for any user-facing or architectural changes.
6. Local Verification: Run all test, lint, format, and typecheck commands locally before committing. All tests must pass with zero errors and zero warnings.
7. PR Template Completion: Always use `.github/PULL_REQUEST_TEMPLATE.md`. Complete all fields, link the issue using `Closes #<id>` or `Fixes #<id>` when the PR fully resolves it, or `Part of #<id>` for partial work (only maintainers with write access may write `None`, for housekeeping), and check all Pre-Submission Governance boxes with real terminal output.
8. Draft PR First: ALWAYS open pull requests as DRAFT (`gh pr create --draft`).
9. CI Verification: Wait for CI workflows to run on the draft PR and verify that all status checks pass.
10. Ready for Review: Only mark the PR as ready for review (`gh pr ready`) once all checks pass and all due diligence is verified.
11. Review Clearance: Review and resolve all CodeRabbit AI and GitHub Copilot comments. Ensure Codecov reports show zero errors, zero warnings, and no coverage drop.


## Executable Verification Commands

- `npm run build` — Compile TypeScript
- `npm test` — Run unit tests
- `npm run lint` — Check lint rules
- `npm run test:coverage` — Run tests with coverage report
- `npm run typecheck` — `tsc -p tsconfig.test.json` (no emit; covers `src`, `test` and `vitest.config.ts`); `npm run lint` also runs it
- `npm run watch` — Build, `npm link`, then nodemon runs `homebridge -I -D` on every `src` change

`prepublishOnly` runs lint + test + build. The test suite does **not** need a prior build:
`vitest.config.ts` aliases `../dist/*.js` to `src/*.ts` for `homebridge-ui/server.js`, whose
runtime imports must keep pointing at `dist/` because that is all the published package ships.

## Architecture

Homebridge *dynamic platform* plugin bridging Tuya-protocol Ventair Skyfan DC ceiling fans
into HomeKit.

- `src/index.ts` — registers the platform with Homebridge.
- `src/settings.ts` — `PLATFORM_NAME` (must match `pluginAlias` in `config.schema.json`) and `PLUGIN_NAME` (must match `name` in `package.json`).
- `src/config.ts` — Zod schema for the platform config. A bad device entry costs that one fan, never the bridge, and **never** unregisters an accessory (that would irreversibly discard its rooms, scenes and automations).
- `src/platform.ts` — reads `config.devices[]`, generates a HAP UUID from the Tuya device id, and constructs one `FanStateManager`, `CeilingFanAccessory`, and (when `api.isMatterEnabled()` is true) `MatterFanBridge` per device.
- `src/dps.ts` — the datapoint ↔ `FanState` translation, both directions. The only place DP numbers appear.
- `src/state.ts` — `FanStateManager`, the single source of truth for `FanState`, per-key optimistic-write versioning, `lastConfirmed`, and version-gated rollback shared by HAP and Matter.
- `src/accessory.ts` — HomeKit services and characteristic handlers consuming `FanStateManager`.
- `src/matter.ts` — Matter device descriptors (`Fan`, optional `DimmableLight` part, optional `OnOffSwitch` Sleep part) and command/state handlers consuming `FanStateManager`.
- `src/tuya/tuyapi.ts` — the transport: one `TuyAPI` connection per accessory, the reconnect supervisor, and the write/readback path. **Do not touch `verifyWrite`, `awaitEcho`, `writeOnce`, the echo-suppression window or the reconnect supervisor** without live-hardware verification; that code was tuned against real fans.
- `src/tuya/discovery.ts` — UDP broadcast discovery (ports 6666/6667, published AES-ECB key).
- `src/tuya/cloud.ts` — Tuya Cloud API client used only to fetch local keys, including the IPv4-preferring agent + alternate-family retry for tuya/tuya-homebridge#412.
- `homebridge-ui/` — the custom Homebridge UI: `server.js` (plain ESM JS, `/discover` and `/keys`) and `public/index.html` (one self-contained page, no build step).

### Tuya DPS Mapping

| DP | Meaning | Values |
|----|---------|--------|
| 1 | fan power | boolean |
| 2 | fan mode | `Normal` \| `Sleep` \| `Eco` — written capitalised, read back lowercase; the set is open (see below) |
| 3 | fan speed | 1–5 |
| 8 | rotation direction | `'forward'` \| `'reverse'` |
| 15 | light power | boolean (untested — no unit here has a light) |
| 16 | light brightness | device scale, mapped to 0–100 |
| 22 | countdown | present on the hardware, deliberately unimplemented |

Mode is **not** a two-value enum. An earlier write probe found that writing `nature`/`smart`
over the LAN lands on Sleep, and an earlier version of these instructions concluded from that
"there is no third mode". That conclusion was wrong: on 2026-07-29 a fan driven from the Smart
Life app reported mode `eco` on DP 2, cycling eco → sleep → eco → normal. So at least three
modes exist — the earlier probe showed only that those two *particular* strings are not
writable over the LAN, not that the mode set is closed.

(Case is not part of that: `toDeviceMode` capitalises on the way out and `toFanState`
lowercases on the way in, so the same mode is `Eco` in a write and `eco` in state.)

Whether `Eco` can be **written** over the LAN is still unknown: each fan accepts only one
LAN session at a time, so a probe cannot run while the plugin holds the connection.

This is exactly why `toFanState` lowercases and keeps whatever string arrives instead of
mapping onto a known enum — `eco` passed through cleanly with no crash and no wrong state.
Keep that behaviour. `exposeModeSwitches` still adds only a Sleep switch, so a fan in eco
shows the switch off, and toggling it writes Normal — HomeKit cannot currently show or
restore eco.

Speed is exposed as 0–100% with `minStep: 20`; conversion lives in `stepToPercent` /
`percentToStep` in `src/dps.ts` and nowhere else.

### Connection Lifecycle

`TuyapiDevice.connect()` is supervised: exponential backoff with jitter, capped at 60s,
with a single in-flight attempt (`inFlight`) so `error` and `disconnected` arriving together
cannot start two loops. A readback timeout recreates the whole underlying `TuyAPI` instance
— tuyapi 7.7.1 keeps a per-sequence resolver it never clears, and there is no cancellation
API — and the discarded instance is both `disconnect()`ed and `removeAllListeners()`ed.

`set()` runs with `shouldWaitForResponse: false`, so it can never reject; a write is
confirmed by the fan echoing the value back, or by a bounded readback poll. Echoes for a
datapoint with a write in flight are buffered, not dropped, and reconciled against an
authoritative read once the settle window closes — that is how a wall switch or the Smart
Life app still reaches HomeKit.

### Config

`config.schema.json` drives the standard Homebridge form; `homebridge-ui/public/index.html`
drives the custom one. Adding a per-device option means touching the schema, the Zod schema
in `src/config.ts`, the custom UI, and the accessory — and keeping the wording consistent
across all of them.

## Boundaries and Safety Constraints

- Always: Write tests for any new or modified functionality, verify changes locally, and update CHANGELOG.md under `[Unreleased]`.
- Ask first: Major architectural changes, removing public APIs, or adding heavy external dependencies.
- Never: Commit secrets, credentials, API keys, or private certificates. Never commit directly to `main`.

## Conventions

- No credentials, local keys, real IPs, real device IDs or passwords in any file, test,
  comment or commit message. Fixtures use `192.0.2.x` and synthetic IDs like `bf0…000a`.
- This extends to the log: `TuyapiDevice` identifies itself by the configured fan name
  (`TuyapiOptions.label`), never by the device id — a short id suffix is only a fallback,
  and it is not guaranteed unique across one production batch.
  **Caveat this plugin cannot close:** running Homebridge with `DEBUG=TuyAPI` makes the
  tuyapi dependency print its own GET/SET payloads, the discovery id and device IPs — and,
  on protocol 3.4/3.5, the local/remote randoms and the negotiated **session key**. That
  output bypasses everything here, and session material matters more than the id, so treat
  a `DEBUG=TuyAPI` log as secret rather than redacting only the ids out of it.
- Commit messages carry no tool attribution or `Co-Authored-By` trailers.
- New behaviour gets a test that demonstrably fails without the change.

## Issue and Pull Request Governance

These rules apply to every contributor, including AI coding agents. The full org-wide rules are in
[GOVERNANCE.md](https://github.com/tinkerologie/.github/blob/main/GOVERNANCE.md).

1. **Issue first.** Open an issue with the bug or enhancement template before opening a pull
   request. Keep the title prefix `[Bug]:` or `[Enhancement]:`, followed by a space and the title.
   Automation applies the `bug` or `enhancement` label when it can tell the type, and CodeRabbit
   then posts a Coding Plan on the issue. Implement from that plan. The labeler skips issues labeled
   `question`, `duplicate`, `invalid`, `wontfix`, or `no-type`, and never re-adds a type label that
   a maintainer removed.
2. **Link the issue.** In the PR's **Related Issues** section, write `Closes #<number>` (or `Fixes`
   or `Resolves`) when the PR completely resolves the issue; these keywords close it on merge. Use
   `Part of #<number>` or `Related to #<number>` for partial work. The linked issue must exist; an
   issue in another public repository (`owner/repo#123`) is fine. Only authors with write access or
   higher (maintainers) may write `None`, for housekeeping such as CI,
   documentation, or dependency updates; PR governance rejects `None` from everyone else.
3. **One change at a time.** Contributors without write access can have at most 2 open, non-draft
   pull requests here. GitHub enforces this limit, and draft pull requests do not count toward it.
   Do not open PRs that edit the same files as another open PR, such as `CHANGELOG.md`,
   translations, `README.md`, or generated bundles. Wait for the first to merge, rebase, then
   continue.
4. **Act on the Tinkerologie-Bot report.** When PR governance fails, Tinkerologie-Bot posts one comment
   listing every problem as a numbered item. Fix each item by editing the PR description or pushing
   the missing change; the check re-runs and the comment updates to passed.
5. **Rebase before review.** Rebase on `main` before marking a PR ready for review.
6. **Resolve conflicts first.** When a PR gets the `conflicts` label, rebase on `main`,
   resolve the conflicts (keep every `CHANGELOG.md` entry under `[Unreleased]`), re-run the tests,
   and force-push before doing any other work.
7. **Stale items.** Issues and PRs with no activity for 14 days are labeled `stale` and closed 30
   days later unless there is new activity. These labels are exempt:
   - Issues: `pinned`, `security`, `confirmed`, `work-in-progress`, `priority: critical`, `priority: high`.
   - Pull requests: `pinned`, `security`, `work-in-progress`.
