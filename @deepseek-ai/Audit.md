
# DeepSeek AI — extended execution audit

Partial source verification

Read-only inspection

Repository: [auraecosystem/deepseek-ai](https://github.com/auraecosystem/deepseek-ai?utm_source=chatgpt.com)\
Default branch: `main`

I retrieved additional commit diffs and successfully inspected the repository's agent command file and Node runtime decision record. The actual package source, root manifests, and workflow definitions remain inaccessible through the connected repository interface.

## 1. Verified findings

### Runtime and compatibility

The repository's engineering note, `.dhs/notes/implemented/process/node-24-runtime-floor.md`, specifies:

- Minimum supported Node.js: 24.9.0.
- Primary test coverage on Node 24.
- Compatibility testing on pinned Node 24.9 and Node 26.
- Node 22 installations must upgrade.
- Runtime dependencies include `node:sqlite`, native TypeScript stripping, and `import.meta.main`.
- The native addon has a separate runtime compatibility policy.

This is a deliberate compatibility strategy rather than an arbitrary version requirement.

Engineering concern: the note references private Node module-loader interfaces. These can change independently of public APIs. Keep compatibility tests around those interfaces and isolate their usage behind a small adapter.

### Build and testing strategy

The repository's command documentation describes these checks:

| Area                       | Documented command       |
| -------------------------- | ------------------------ |
| Unit tests                 | `pnpm run test`          |
| Coverage                   | `pnpm run test:coverage` |
| Type safety                | `pnpm run typecheck`     |
| Lint                       | `pnpm run lint`          |
| Build                      | `pnpm run build`         |
| Dependency/package hygiene | `pnpm run hygiene`       |
| Snapshot replay            | `pnpm run test:snapshot` |
| Real API integration       | `pnpm run test:e2e`      |
| Documentation              | `pnpm run test:docs`     |
| Website                    | `pnpm run website:build` |

The documented coverage target is 100% per file under `packages/*/*/src`.

These are documented test expectations, not verified test results. I did not execute the repository's build or test commands.

## 2. Issues identified

Finding 1 — Mixed-platform command file

High priority

`.agent/Command` combines an `ash` shebang and shell commands with a second PowerShell shebang and PowerShell instructions.

A file has one interpreter entry point. The second shebang does not switch interpreters halfway through execution.

Recommendation: split it into platform-specific files:

- `.agent/build.sh`
- `.agent/setup.ps1`
- `.agent/README.md`

Also clarify whether expressions such as `tsc -b <tsconfig.host.json references>` are illustrative placeholders or intended executable commands.

Finding 2 — Dynamic Git worktree creation

Security review required

The setup instructions fetch a repository and construct a Git worktree using environment variables:

```
git fetch "$DSH_SUPERAGENT_GITHUB_REPOSITORY" \
  "$DSH_SUPERAGENT_GITHUB_PR_HEAD:refs/dsh-superagent/$DSH_SUPERAGENT_TICKET"
```

Validate repository ownership, branch/ref syntax, destination paths, and allowed worktree roots. Untrusted values should not control arbitrary Git operations.

Finding 3 — Orchestrator configuration update

Security review required

The setup sequence reads a revision and sends a PUT request to a local orchestrator endpoint.

The revision-based update is a useful concurrency mechanism. However, the implementation should also verify:

- Authentication and authorization of the caller.
- Whether the request header is merely a routing marker or an actual security control.
- JSON-safe serialization of environment-provided values.
- Failure handling and revision conflicts.
- Whether the endpoint is protected against unauthorized local requests.

  Finding 4 — Runtime compatibility

  Architecture risk

  Private Node loader integration and a separately supported native addon introduce compatibility surfaces that need explicit version matrices.

Keep native binaries, Node runtime support, and loader-specific logic independently testable.

Finding 5 — Documentation completeness

Needs verification

The retrieved `@deepseek-ai/session-controller/README.md` is empty. This is a real documentation gap for that path, although I could not inspect the full documentation tree.

## 3. Architecture assessment

Based on the package responsibilities you supplied, this is a broad agent runtime with several important subsystems:

| Subsystem              | Responsibility              | Main engineering concern         |
| ---------------------- | --------------------------- | -------------------------------- |
| Core                   | Agent/session lifecycle     | State consistency                |
| API                    | Remote backend-for-frontend | Authentication and isolation     |
| Typert                 | Type graphs                 | Schema correctness               |
| LLM                    | Model providers             | Provider abstraction and secrets |
| PTC runtime            | Programmatic tool execution | Code isolation                   |
| Sandbox                | Execution containment       | Escape prevention                |
| Subagent               | Delegated agent execution   | Permission inheritance           |
| MCP                    | External tool connectivity  | Capability restrictions          |
| SSH / terminal         | Remote execution            | Command authorization            |
| Browser / computer use | GUI automation              | User consent and scope           |
| Storage / workspace    | Persistent state            | Tenant and path isolation        |
| Guard / credentials    | Security controls           | Policy enforcement               |

The most important security boundary is the transition from an agent's intent to an actual external action.

A model deciding to execute a command must not itself constitute authorization to execute that command.

## 4. Recommended security architecture

For a runtime with this many execution capabilities, I recommend a centralized authorization pipeline:

```
Agent / Subagent
      |
      v
  Task Planner
      |
      v
  Policy Evaluation
      |
      +---- Deny
      |
      +---- Require Approval
      |
      v
 Capability Validation
      |
      v
 Isolated Execution
      |
      v
 Output Validation
      |
      v
 Audit / Session Record
```

Every execution adapter should receive a scoped capability rather than unrestricted host access.

Examples:

- Filesystem adapter: authorized workspace paths only.
- Terminal adapter: command allowlists or constrained execution profiles.
- SSH adapter: approved hosts and credentials.
- Browser adapter: explicitly authorized sessions and domains.
- MCP adapter: individually granted tools and operations.
- Subagent adapter: permissions no greater than the delegating agent.

## 5. How this maps to LMLM

The architecture has useful patterns for LMLM, but the most important distinction must remain intact: LMLM should own its control plane.

## LMLM sovereign execution architecture

LMLM Control Plane

Policy · Identity · Planning · Orchestration · Audit

Model Fabric

Local models · Providers · Multimodal inference

Agent Runtime

Sessions · Plans · Subagents · Workflows

Execution Fabric

Sandbox · Terminal · Filesystem · Jobs

Memory Fabric

Context · Storage · Retrieval · State

Authorized Adapter Fabric

Internet · Devices · Networks · External platforms · Distributed agents

The key design rule: external systems may perform authorized work, but they must not become the authority that governs LMLM itself.

## 6. Execution status

| Audit activity             | Result         |
| -------------------------- | -------------- |
| Repository metadata        | Retrieved      |
| Commit history             | Retrieved      |
| Agent command instructions | Inspected      |
| Node runtime decision      | Inspected      |
| Root package manifest      | Blocked by 404 |
| Workspace package sources  | Blocked by 404 |
| CI workflow files          | Not verified   |
| Security implementation    | Not verified   |
| Build and tests            | Not executed   |
| Repository modifications   | None           |

Final assessment: The documented design shows a serious, ambitious agent-runtime architecture with a substantial testing strategy. The strongest areas to investigate next are execution isolation, delegated permissions, credential handling, and compatibility boundaries. The mixed-platform command file is the clearest directly evidenced maintenance defect.

I have not certified the repository as secure, production-ready, or passing CI. That would require inspecting the actual source files and running its checks.

Next practical step: provide a repository ZIP or source archive so I can inspect the actual workspace packages, trace the security-sensitive execution paths, and produce a complete implementation-level audit without guessing about inaccessible files.
