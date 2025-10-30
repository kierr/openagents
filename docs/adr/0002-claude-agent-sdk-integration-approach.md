---

title: "ADR-0002: Claude Agent SDK Integration Approach"
date: "2025-10-30"
status: "Draft" # Proposed | Draft | In Review | Accepted | Rejected | Deprecated | Superseded
authors:
  - "@kierr"
reviewers: [] # When ready for review, add GitHub handles as strings

---

<!--
⚠️  IMPORTANT: If you're an AI agent, read AGENT.md before using this template.

This template is designed for direct, honest architectural decisions.
- Focus on real problems, not abstract benefits
- Acknowledge costs and trade-offs, don't hide them
- Use specific examples from the OpenAgents codebase
- Be realistic about maintenance overhead and adoption challenges

How to use this template:
1. Copy this template to create a new ADR with the proper four-digit numbering (e.g., `0001-your-title.md`)
2. Fill in the frontmatter with the ADR details
3. Write the content of the ADR in the sections below
4. Focus on "why this solves a real problem" rather than "industry best practices"
5. Include failure indicators - when will we know this decision was wrong?
-->

## 1. Context and Problem Statement

Claude Code integration was added a week ago via CLI process spawning (`claude_runner.rs`). The implementation works but has immediate code quality issues that need to be addressed before the pattern becomes entrenched:

- **Code duplication**: `claude_runner.rs` (256 lines) and `codex_runner.rs` (400+ lines) share nearly identical process spawning, stdout parsing, and error handling logic
- **Complex process management**: Each prompt spawns child processes with platform-specific signal handling (`setpgid`, process isolation)
- **Translation overhead**: Both runners require translation layers in `acp-event-translator` (797 lines) to convert provider-specific events to ACP format
- **Testing complexity**: Requires fake binaries (`fake-claude`, `fake-codex`) for integration tests

The current approach raises the question: should we continue with CLI integration or use the Claude Agent SDK for direct library integration?

## 2. Decision Drivers

**Immediate technical debt:**
- Code duplication between runners creates maintenance burden - changes need to be made in multiple places
- Complex mutex contention in `claude_runner.rs` (6+ locks per request) creates performance bottlenecks
- JSON parsing errors in production show the current approach is fragile

**Integration complexity:**
- Provider detection logic in `ws.rs:250-295` is nested and hard to follow
- Session mapping between client thread IDs and provider sessions is error-prone
- Error handling is inconsistent across providers

**Questioning fundamental approach:**
- Process spawning feels like an outdated integration pattern
- The translation layer adds complexity but might not be necessary with direct SDK integration
- Mobile bridge architecture requires WebSocket streaming regardless of provider

## 3. Considered Options

### Option 1: Keep CLI Integration + Refactor Code Duplication

*   **Description:** Maintain current approach of spawning `claude --output-format stream-json` but extract common process management logic into shared abstractions
*   **Real-world impact:** Minimal changes to working system, but reduces code duplication and improves maintainability
*   **Pros:**
    *   CLI integration already works and is deployed
    *   No new runtime dependencies (Node.js, Python, etc.)
    *   Familiar pattern that matches existing Codex integration
    *   Can be implemented incrementally without breaking changes
*   **Cons:**
    *   Still has process spawning overhead for each prompt
    *   Translation layer complexity remains (provider events → ACP format)
    *   JSON parsing can still fail and requires robust error handling
    *   Doesn't leverage potential SDK benefits (session management, streaming, type safety)

### Option 2: TypeScript SDK Integration via Node.js Subprocess

*   **Description:** Replace CLI process with Node.js subprocess that uses `@anthropic-ai/claude-agent-sdk` for direct library integration
*   **Real-world impact:** Would eliminate CLI dependency but introduce Node.js runtime management in Rust bridge
*   **Pros:**
    *   True direct library integration (no subprocess spawning within subprocess)
    *   Access to SDK features: session management, streaming, custom tools, subagents
    *   Type-safe interfaces with TypeScript
    *   Better error handling and reliability than CLI JSON parsing
*   **Cons:**
    *   Requires Node.js runtime in Rust bridge deployment
    *   Adds complexity: need to manage Node.js processes, dependencies, and lifecycle
    *   Still has subprocess overhead (just Node.js instead of Claude CLI)
    *   Deployment complexity increases (need to package Node.js modules with Rust binary)

### Option 3: Hybrid SDK Integration (Happy-CLI Pattern)

*   **Description:** Implement Happy-CLI's hybrid approach - direct SDK integration for core functionality with CLI fallback for edge cases, plus mobile-first architecture
*   **Real-world impact:** Would add mobile connectivity and real-time session sharing capabilities while maintaining full Claude Code compatibility
*   **Pros:**
    *   Proven pattern in production (Happy-CLI successfully uses this approach)
    *   SDK integration for better control and message interception
    *   Mobile-to-desktop session sharing via QR code authentication
    *   Permission forwarding to mobile devices (MCP integration)
    *   Real-time WebSocket communication with end-to-end encryption
    *   Seamless switching between local and remote modes
*   **Cons:**
    *   Significant architectural complexity (mobile server, WebSocket layer, encryption)
    *   Requires building mobile client capabilities
    *   Dependency on external infrastructure (relay servers)
    *   More complex session management and state synchronization

### Option 4: Terminal Proxy Pattern (VibeTunnel Approach)

*   **Description:** Implement VibeTunnel's terminal proxy pattern - treat Claude Code as a terminal service and proxy it through sophisticated real-time streaming architecture
*   **Real-world impact:** Would create browser-based terminal access with advanced session management and cross-platform support
*   **Pros:**
    *   Production-proven terminal streaming architecture (dual WebSocket channels, PTY management)
    *   Advanced session persistence (asciinema-compatible recording/playback)
    *   Sophisticated buffer management and flow control for large outputs
    *   Cross-platform native app + web interface pattern
    *   Mobile-optimized terminal emulation with touch input
    *   Multiple authentication modes (SSH keys, system auth, environment variables)
*   **Cons:**
    *   Designed for terminal access, not specifically AI agent integration
    *   High complexity (three-component architecture: native app + server + web frontend)
    *   Significant engineering overhead to replicate VibeTunnel's sophistication
    *   May not address core AI integration needs (tool integration, reasoning, etc.)

### Option 5: WASM-SDK Hybrid Approach

*   **Description:** Compile Claude Agent SDK to WebAssembly and run in Node.js via WASI for sandboxed library integration without subprocess overhead
*   **Real-world impact:** True library integration with memory-safe isolation and cross-platform consistency
*   **Pros:**
    *   Eliminates subprocess spawning entirely
    *   Memory-safe sandboxing via WASM runtime
    *   Cross-platform consistency (WASM runs everywhere)
    *   Hot-swappable SDK versions by replacing WASM modules
    *   Native Rust bridge + WASM SDK hybrid architecture
*   **Cons:**
    *   WASM compilation complexity for Node.js SDK
    *   Potential performance overhead of WASM runtime
    *   Limited WASI support in some environments
    *   Complex toolchain and build process
    *   Emerging technology with limited production examples

## 4. Decision Outcome

**Chosen Option:** Option 1 - Keep CLI Integration + Refactor Code Duplication

**Rationale:**
After comprehensive investigation of SDK alternatives and analysis of production systems (Happy-CLI, VibeTunnel), the evidence supports keeping the current approach:

- **SDK investigation reveals complexity**: The Rust SDK is a CLI wrapper, and the TypeScript SDK requires Node.js runtime management. Both add complexity rather than reducing it.

- **Production evidence supports CLI approach**: Happy-CLI successfully uses both SDK and CLI integration but demonstrates significant architectural complexity (WebSocket servers, encryption, mobile infrastructure). VibeTunnel shows sophisticated terminal streaming but requires three-component architecture.

- **Current problems are implementation quality**: Code duplication between `claude_runner.rs` (256 lines) and `codex_runner.rs` (268 lines), complex provider routing, and mutex contention are solvable through refactoring.

- **Mobile bridge architecture constraints**: OpenAgents serves mobile clients via WebSocket bridge, which is the correct architectural pattern. The CLI integration works well within this constraint.

- **Risk/benefit analysis**: More complex approaches (Happy-CLI hybrid, VibeTunnel proxy, WASM compilation) introduce significant engineering overhead for problems we don't currently have.

This isn't about "industry best practices" - it's about solving the actual problems we have with the minimum effective complexity. The CLI approach works, is deployed, and our pain points are specific implementation issues, not fundamental architectural problems.

## 5. Consequences

### What We Get (The Good Stuff)

- Eliminated code duplication between `claude_runner.rs` (256 lines) and `codex_runner.rs` (268 lines)
- Simplified process management with shared abstractions for spawning, stream parsing, and error handling
- Reduced mutex contention through better state management patterns (measured via lock contention metrics)
- Cleaner provider routing logic in `ws.rs:250-295` using declarative provider registry
- Maintained working system without introducing Node.js, mobile servers, or complex infrastructure
- Established foundation for future provider additions without architectural debt

### What It Costs (The Real Trade-offs)

- Process spawning overhead remains (but this is acceptable performance - proven by current usage)
- Translation layer complexity remains but can be centralized and simplified
- Refactoring risk to existing working Claude Code integration
- JSON parsing fragility remains but can be made more robust with centralized error handling
- We don't get advanced features from more complex approaches (mobile session sharing, terminal recording, etc.)

### What Changes (The Cultural Impact)

- Process management becomes a shared concern rather than duplicated per provider
- Provider addition becomes standardized through common abstractions
- Error handling and testing patterns become consistent across providers
- Decision establishes precedent for solving implementation quality issues rather than architectural overhauls

**Bottom line:** We're trading the theoretical benefits of complex architectures for proven simplicity and immediate code quality improvements. The refactoring solves our actual problems (duplication, complexity) without introducing new ones (mobile infrastructure, additional runtimes, deployment complexity).

## 6. Validation Plan

**Success indicators (the "does this actually help?" test):**
- Code duplication eliminated: Shared `ProcessRunner` trait reduces `claude_runner.rs` + `codex_runner.rs` duplication by 70%+
- Lock contention reduced: Mutex operations per request drop from 6+ to 2-3 through better state management patterns
- Provider routing simplified: Nested conditionals in `ws.rs:250-295` replaced with declarative provider registry
- Test coverage improves: Single test framework eliminates need for separate `fake-claude` and `fake-codex` binaries
- New provider addition time reduced: Adding future providers requires <50 lines instead of 250+ lines

**Failure indicators (time to reconsider):**
- Production regressions: Claude Code functionality breaks after refactoring (immediate rollback required)
- Abstraction overhead: Shared abstractions add more complexity than they remove (measured by code review friction)
- Performance degradation: Request latency increases by >20% due to added abstraction layers
- Provider addition difficulty: Adding new AI providers remains complex despite refactoring
- Team productivity drops: Developers find the new abstractions harder to understand than original duplicated code

Re-evaluate after 2 weeks of production use. If the refactoring doesn't measurably improve code quality, maintainability, and development velocity, reconsider the hybrid SDK approaches with the understanding that they introduce significant architectural complexity.

## 7. References

- GitHub Issue #1334: Integrate Claude Code (in-repo ACP mapping; no external adapter/binary)
- Claude Agent SDK Documentation: https://docs.claude.com/en/api/agent-sdk/overview.md
- Rust SDK Investigation: https://github.com/Wally869/claude_agent_sdk_rust
- Happy-CLI Analysis: https://github.com/slopus/happy-cli (hybrid SDK + CLI integration with mobile sharing)
- VibeTunnel Analysis: https://github.com/amantus-ai/vibetunnel (terminal proxy architecture with real-time streaming)
- Current Implementation: `crates/oa-bridge/src/claude_runner.rs` (256 lines), `crates/oa-bridge/src/codex_runner.rs` (268 lines)
- Translation Layer: `crates/acp-event-translator/src/lib.rs` (796 lines)
- Provider Routing: `crates/oa-bridge/src/ws.rs:250-295`