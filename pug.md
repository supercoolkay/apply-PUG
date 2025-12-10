# Parachain Upgrade Guard (PUG)

## Project Overview

- **Tagline (one sentence):**  
  A small CI‑ready guardrail that standardises `try-runtime` migration checks for Polkadot SDK parachains.

- **Brief description of the project:**  
  Parachain Upgrade Guard (PUG) is a command line tool and CI integration that turns existing `try-runtime` storage migration checks into a repeatable, machine‑readable gate. It reads a simple configuration file, runs `try-runtime on-runtime-upgrade` against a state snapshot or live endpoint, and applies a small, explicit policy:

  - fail if `on-runtime-upgrade` returns a panic or error  
  - fail if reported migration weight exceeds a configurable share of the maximum block weight, where this information is available  
  - emit a structured JSON report and CI‑friendly exit code  

  This enables parachain teams to include an additional migration safety check in their pipelines instead of relying only on ad‑hoc, manual commands and scripts.

- **Relation to Polkadot (SDK) / Kusama:**  
  PUG is designed for teams running Polkadot SDK–based parachains on Polkadot or Kusama. It sits between building a new runtime and submitting it via governance. It uses the existing `try-runtime` binary that is part of the Polkadot SDK toolchain and does not replace or modify it. PUG standardises how `try-runtime` is called and how its results are interpreted for CI.

- **Why our team is interested:**  
  Our team has worked with Polkadot SDK–based chains and seen how sensitive runtime upgrades and storage migrations can be. Migrations that panic or use too much weight on real chain state can impact liveness and require corrective action. Today, many teams appear to use `try-runtime` manually or with project‑specific scripts. We want to provide a small, open‑source guard that makes this step easier to adopt, more repeatable, and easier to review.

---

### Project Details

- **Overview of the technology stack:**

  - Language: Rust  
  - CLI and config: `clap`, `serde`, `toml`  
  - Runtime testing: external `try-runtime-cli` binary from the Polkadot SDK  
  - Build and distribution:
    - Rust crate on crates.io  
    - Release binaries for Linux and macOS  
    - Optional Docker image for CI environments

- **Core components, architecture and protocols:**

  1. **Configuration Loader**  
     - Reads a configuration file (for example `pug.toml`) or command line flags with fields such as:  
       - `chain_label`  
       - `snapshot_path` or `rpc_url`  
       - `old_runtime_wasm` and `new_runtime_wasm`  
       - `max_weight_ratio` threshold  
     - Validates paths and values and prepares a `try-runtime` invocation plan.

  2. **Try Runtime Runner**  
     - Invokes `try-runtime on-runtime-upgrade` using the provided snapshot or live endpoint and runtime WASM paths.  
     - Captures exit status and process output.  
     - Attempts to extract migration weight information from the output where it is clearly available. If weight information is not exposed in a stable way, PUG records it as unknown instead of inferring it.

  3. **Policy Engine**  
     - Applies a small set of rules:
       - Rule 1. Any panic or non‑zero exit code from `try-runtime` is marked as a failure.  
       - Rule 2. If migration weight is available and exceeds `max_weight_ratio` of the configured maximum block weight, PUG marks this as a failure or warning according to configuration.
     - Produces a `report.json` file with fields such as:
       - `status` (for example pass, fail, error)  
       - `panic` (boolean and optional message)  
       - `weight_ratio` (number or `"unknown"`)  
       - raw log path or inline summary  

  4. **CLI Interface**  
     - `pug check --config pug.toml`  
       Runs `try-runtime on-runtime-upgrade` according to the config, applies the rules, writes `report.json`, and exits with `0` on pass and `1` on fail.  
     - `pug init --out pug.toml`  
       Generates a commented configuration template for a typical parachain upgrade.  
     - `pug print-report report.json`  
       Pretty‑prints a human‑readable summary of the JSON report.

  PUG does not spawn or manage nodes. It does not implement its own runtime execution. It relies on the existing `try-runtime` binary provided by the Polkadot SDK.

- **PoC/MVP or prior work:**  

  We have experimented with manual `try-runtime` flows on local development chains to understand:

  - how snapshots are produced and consumed  
  - how `on-runtime-upgrade` behaves in different scenarios  
  - how logs and exit codes can be used to detect panics and weight usage  

  This research informs the initial rule set and configuration format for PUG. The grant would fund the first publicly released implementation of PUG as a standalone tool.

- **Mockups/designs of any UI components:**  

  PUG is a CLI‑only tool. There is no graphical UI. We will design:

  - an example `pug.toml` configuration template  
  - example JSON reports  
  - example CI workflow snippets (GitHub Actions / GitLab CI)

- **Data models / API specifications of the core functionality:**

  - **Config (TOML):**  

    ```toml
    chain_label = "example-parachain"

    snapshot_path     = "./snapshots/example.bin" # or: rpc_url = "wss://..."
    old_runtime_wasm  = "./runtime/old.wasm"
    new_runtime_wasm  = "./runtime/new.wasm"

    max_weight_ratio  = 0.4
    on_weight_exceed  = "fail"  # or "warn"
    ```

  - **Report (JSON):**

    ```json
    {
      "chain_label": "example-parachain",
      "status": "fail",
      "panic": {
        "occurred": true,
        "message": "on_runtime_upgrade panicked at ..."
      },
      "weight_ratio": 0.65,
      "max_weight_ratio": 0.4,
      "logs_path": "./pug-logs/example-upgrade.log",
      "timestamp": "2025-02-01T12:34:56Z"
    }
    ```

  - **CLI surface (examples):**

    ```bash
    # Run a check using a config file
    pug check --config pug.toml

    # Create a template config
    pug init --out pug.toml

    # Pretty print the last report
    pug print-report report.json
    ```

- **What the project is not / limitations:**

  PUG is intentionally narrow. It will **not**:

  - replace or extend `try-runtime`; it always calls the external `try-runtime` binary  
  - orchestrate or simulate networks; it does not start nodes or manage multi‑node setups  
  - provide storage diffing, detailed migration analytics, dashboards, or a graphical interface  
  - guarantee that a runtime upgrade is safe; it is an additional guard that enforces simple checks based on the information `try-runtime` provides  
  - act as a generic testing framework for runtimes; it focuses only on `on-runtime-upgrade` checks for storage migrations

---

### Ecosystem Fit

- **Where and how does your project fit into the ecosystem?**

  PUG fits into the runtime upgrade workflow for Polkadot SDK–based parachains. It lives between:

  1. Building a new runtime and writing migrations, and  
  2. Submitting that runtime through governance or another upgrade mechanism.

  It packages a step that is already recommended in documentation – running `try-runtime on-runtime-upgrade` on state – into a small, reusable tool with a clear pass/fail signal and a JSON report. This is relevant to parachains on Polkadot, Kusama, and test networks.

- **Who is your target audience?**

  - Parachain runtime developers working with the Polkadot SDK  
  - DevOps and infrastructure engineers responsible for upgrade pipelines  
  - Technical reviewers who want a simple artefact showing how `on-runtime-upgrade` behaved on a given snapshot

- **What needs does your project meet?**

  - Reduces the likelihood that a migration which panics or consumes too much weight on real chain state is deployed without being noticed through automated checks  
  - Reduces duplicated effort where each team writes its own small script to call `try-runtime` and manually interpret logs  
  - Provides a consistent JSON report and exit code that CI systems can consume and that can be attached to internal or governance‑level discussions

- **How did you identify these needs?**

  - By working with Polkadot SDK–based projects and observing the practical steps teams use when preparing upgrades  
  - By studying documentation that emphasises testing storage migrations and using `try-runtime` before deploying new runtimes  
  - By noting that integrations of `try-runtime` into CI are often project‑specific and not standardised

- **Are there any other projects similar to yours in the Polkadot/Kusama ecosystem? How is your project different?**

  Related tools and patterns include:

  - `try-runtime`: provides the core ability to execute `on-runtime-upgrade` against live or snapshot state. PUG uses this tool directly. It does not add new execution capabilities. It focuses on configuration, repeatable invocation, simple rules, and reporting.  
  - Network orchestration tools: start and manage local nodes or networks for testing. PUG does not start nodes or manage networks and therefore does not overlap with this scope.  
  - Storage inspection and runtime debugging tools: help explore how storage changes across an upgrade and where a chain can break. PUG does not inspect storage contents. It only checks for basic execution success, absence of panic, and weight headroom.

  We are not aware of another small, CI‑oriented CLI in the Polkadot ecosystem that is dedicated to turning `try-runtime on-runtime-upgrade` into a standardised, configurable guard with JSON output.

- **Are there any projects similar to yours in related ecosystems?**

  In other ecosystems, it is common to have migration and upgrade helpers that:

  - run migrations against a separate environment,  
  - apply simple policies (for example, “no errors, time below threshold”),  
  - and surface the result to CI.

  PUG takes a similar approach but is specific to Polkadot SDK runtimes and the `try-runtime` tool.

---

## Team

- **Team Name:** Dapps over Apps  
- **Contact Name:** Abdulkareem Oyeneye  
- **Contact Email:** team@dappsoverapps.com  
- **Website:** https://www.dappsoverapps.com  

### Team members

- Bolaji Ahmad – Full Stack Engineer (Polkadot Blockchain Academy alumni)  
- Abdulkareem Oyeneye – Tech lead 
- Gospel Ifeadi – Smart Contract Engineer  

#### LinkedIn Profiles (if available)

- https://www.linkedin.com/in/bolajahmad  
- https://www.linkedin.com/in/abdulkareem-oyeneye  
- https://x.com/gospel70800

### Team Code Repos
- RegionX:
Worked on RegionX-Node, coretime-notification codebase
https://github.com/RegionX-Labs/Coretime-Notifier
https://github.com/RegionX-Labs

- Contributed to Pop CLI codebase
https://github.com/r0gue-io/pop-cli

- Contributed to Subsocial node, grillchat
https://github.com/dappforce/subsocial-parachain
https://github.com/dappforce/grillchat

- Contributed to Polkadot SDK codebase
https://github.com/paritytech/polkadot-sdk

Other Projects: (Non Polkadot):

- We built a local testing patch that adds native support for Arbitrum precompiles (ArbSys at 0x64, ArbGasInfo at 0x6c) and transaction type 0x7e (deposits) to Hardhat and Foundry (Anvil).
Project Website For more Info: www.ox-rollup.com

- We created a Retrieval Utility tool for Filecoin that test CID retrieval performance across multiple public gateways. Link: https://www.retrievaltester.com/

- We created a professional VoxEdit to Unity/Roblox Asset Converter for Sandbox creators and gamers.

Website: www.voxbridge.info

- We built a VS code extension for Arbitrum Stylus designed to provide a superior coding experience for Arbitrum Stylus smart contract development.
Link: https://github.com/Supercoolkayy/Abitrum-stylus-extension/tree/main

- We built a Portable Sandbox Identity Toolkit hat lets you convert your Sandbox avatars to VRM format for use across metaverse platforms like Unity, VRChat, and more. Includes blockchain-based ownership verification to ensure you own the avatar NFT before conversion.


### Team’s experience

The team has experience with:

- Polkadot SDK–based chains and runtimes  
- Developer tooling for Web3 (command line tools, CI integration, notifications)  
- Frontend and backend development for Polkadot ecosystem projects  
- Documentation, hackathon participation, and developer onboarding


---

## Development Status

We have not yet started implementing PUG as a public repository. Current work is at the research and design stage:

- Experimenting with manual `try-runtime` workflows on local development chains  
- Reviewing documentation and examples of runtime upgrades and storage migrations  
- Identifying the minimum useful set of checks and configuration options

This grant would fund the initial open source implementation of PUG and the supporting documentation and examples.

---

## Development Roadmap

### Overview

- **Estimated Duration:** 3 months  
- **Full‑Time Equivalent (FTE):** approximately 1.5 FTE over 3 months  
- **Total Costs:** 25 000 USD (balanced scope; lean 20 000 USD and extended 30 000 USD variants are possible)

### Milestones

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 0a. | License | All code will be released under **Apache 2.0**. |
| 0b. | Documentation | We will provide inline Rust documentation and a basic tutorial that explains how to install PUG, create a configuration file, and run `pug check` against a sample snapshot. |
| 0c. | Testing and Testing Guide | Core functions (configuration parsing, policy evaluation, report generation) will be covered by unit tests. We will provide a guide explaining how to run unit and integration tests locally. |
| 0d. | Docker | We will provide a Dockerfile that bundles `try-runtime-cli`, PUG, and dependencies so that all functionality delivered with this milestone can be tested in CI using a single image. |
| 0e. | Article | We will publish an article or forum post that explains what PUG does, how it relates to `try-runtime`, what was done as part of the grant, and how parachain teams can adopt it. |
| 1. | PUG Core CLI v0.1 | We will implement the core `pug check` command which reads a configuration file, calls `try-runtime on-runtime-upgrade`, applies basic rules (no panics, weight headroom where weight is available), produces a JSON report, and returns a CI‑friendly exit code. An example repository with a “good” and “bad” migration will demonstrate expected behaviour. |
| 2. | PUG v1.0 with init, report printing, and CI examples | We will add `pug init` and `pug print-report` commands, extend configuration options, and provide ready‑to‑use CI templates (GitHub Actions and GitLab CI) that run PUG as part of a runtime upgrade pipeline. Documentation and examples will be updated accordingly. |

#### Milestone 1 – PUG Core CLI v0.1

- Duration: 6 weeks  
- Cost: 12 500 USD  

**Deliverables**

- A Rust CLI that:

  - supports `pug check --config pug.toml`  
  - reads and validates configuration fields (`snapshot_path` or `rpc_url`, `old_runtime_wasm`, `new_runtime_wasm`, `max_weight_ratio`)  
  - calls `try-runtime on-runtime-upgrade` with the configured parameters  
  - detects panics and errors from exit status and output  
  - extracts migration weight when clearly exposed and records `"unknown"` otherwise  
  - writes `report.json` containing status, panic information, weight information, and log reference  
  - exits with `0` on pass and `1` on fail  

- An example repository containing:

  - a small example parachain or development chain  
  - one “good” migration that passes PUG checks  
  - one “bad” migration that triggers a panic or error and fails PUG  

- Tests and documentation:

  - unit tests for configuration parsing and policy logic  
  - an integration test that runs PUG against the example repository and checks expected pass/fail outputs  
  - README “Getting started” instructions

**Acceptance Criteria**

- Running PUG on the example “good” migration yields `status = "pass"` and exit code `0`.  
- Running PUG on the example “bad” migration yields `status = "fail"` and non‑zero exit code.  
- Documentation is sufficient for a developer familiar with Polkadot SDK to reproduce these results.

#### Milestone 2 – PUG v1.0 with init, report printing, and CI examples

- Duration: 6 weeks  
- Cost: 12 500 USD  

**Deliverables**

- Extended CLI with:

  - `pug init` command that generates a commented `pug.toml` template  
  - `pug print-report` command that prints a human‑readable summary of a JSON report  
  - configuration options to:

    - set `max_weight_ratio`  
    - choose whether exceeding the ratio results in a warning or a failure  

- CI templates:

  - a GitHub Actions workflow example that:

    - builds a runtime  
    - generates or downloads a snapshot  
    - runs `pug check`  
    - fails the job when `status = "fail"`  

  - a GitLab CI snippet that performs equivalent steps

- Example repository updates:

  - CI workflows integrated into the example repository  
  - documentation showing that modifying the example migration to an unsafe version causes the CI job to fail

- Documentation and article:

  - full README update covering all commands and configuration fields  
  - short guide for integrating PUG into existing pipelines  
  - publication of the article / forum post describing the project and how to adopt it

**Acceptance Criteria**

- `pug init`, `pug check`, and `pug print-report` work as documented on the example repository.  
- The GitHub Actions and GitLab CI examples run successfully and fail when the “bad” migration is enabled.  
- The article is published and links to the repository and example workflows.

### Budget Breakdown (balanced 25 000 USD)

| Category  | Item                        | Cost (USD) | Amount (FTE over 3 months) | Total (USD) | Description |
|----------|-----------------------------|-----------:|---------------------------:|------------:|-------------|
| Personnel | Rust engineer / tech lead   | 12 000     | ~0.75 FTE                  | 12 000      | Architecture, core CLI, policy engine, integration tests |
| Personnel | Rust engineer / QA and CI   | 8 000      | ~0.50 FTE                  | 8 000       | Example repository, fixtures, CI templates, extra tests |
| Personnel | Documentation and outreach  | 5 000      | ~0.25 FTE                  | 5 000       | README, user guide, article, community support |
|          |                             |            |                            | **25 000**  |             |

(Lean 20 000 USD and extended 30 000 USD versions can be provided if requested.)

---

## Future Plans

- **Long‑term maintenance and development:**  
  After the grant, we plan to maintain PUG as a small, stable tool by:

  - tracking major Polkadot SDK and `try-runtime` changes that affect PUG  
  - addressing bug reports and minor feature requests that fit the scope  
  - accepting community contributions for additional rules if they remain within the narrow migration check focus

- **Short‑term use, enhancement, and promotion:**  

  - support early adopters (parachain teams) in wiring PUG into their upgrade pipelines  
  - refine default thresholds and messages based on feedback  
  - share usage examples and CI snippets with the broader community via documentation and posts

- **Long‑term plans:**  

  - propose PUG as an optional tool in Polkadot SDK runtime upgrade documentation, so new teams can easily discover it  
  - keep the project small and focused so it remains easy to adopt and maintain  
  - collaborate with tooling maintainers where appropriate so that patterns proven in PUG can also inform official guides and examples

---

## Additional Information

- The scope of PUG was chosen to avoid overlap with existing tools and to remain feasible within a three‑month, 20 000–30 000 USD grant.  
- All code will be open source. Teams are free to adopt, fork, or extend PUG according to their needs.  
- We have not submitted this specific project for funding to any other entity. If we seek complementary funding or support in the future, we will disclose it and ensure there is no double funding for the same work.
