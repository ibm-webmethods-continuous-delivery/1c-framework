# Readiness for Continuous Delivery

## TL;DR

Whether a piece of software can be delivered continuously depends less on the team's ambition than on the properties of the software and of its tooling. This article defines those properties as **readiness aspects**, proposes **indicators** to measure each of them, and describes how they interact with **branching strategies**, **ephemeral agents and environments**, and **permanent environments**.

The aspects describe the technology. The [DORA metrics](https://dora.dev/) describe the outcome. The aspects are used as predictors of the outcome: a technology rated well on the aspects makes good DORA results possible, a technology rated poorly makes them unlikely whatever the team does.

**Table of Contents**
- [Readiness for Continuous Delivery](#readiness-for-continuous-delivery)
  - [TL;DR](#tldr)
  - [Context and scope](#context-and-scope)
  - [Readiness aspects](#readiness-aspects)
  - [Rating scale](#rating-scale)
  - [Branching strategies](#branching-strategies)
  - [Ephemeral agents and environments](#ephemeral-agents-and-environments)
  - [Permanent environments and their distance](#permanent-environments-and-their-distance)
  - [Outcome metrics](#outcome-metrics)
  - [Anti-patterns](#anti-patterns)
  - [Applying the aspects](#applying-the-aspects)

---

## Context and scope

In this framework "continuous" means that a piece of independently deployable software can be delivered at least once a day, in a secure and repeatable manner (see [Software Delivery Positioning](./SoftwareDeliveryPositioning.md)). The strategies by which organizations deliver software, and why continuous delivery with ephemeral qualification is the recommended target, are discussed in [Software Delivery Strategies](./SoftwareDeliveryStrategies.md).

This article answers a different question: **what makes a given software technology, product or implementation approach appropriate for that way of delivering?**

It is deliberately limited to properties of the software and its tooling. Team organization, culture and collaboration practices are covered by the [Continuous Delivery Checklist](./CDChecklist.md) and are out of scope here.

The aspects apply at different granularity levels: a product (for example an integration runtime), a capability of a product (for example configuring a connection), or an implementation pattern (for example a way of structuring a service). The same aspects can be used for all of them. Not every aspect is relevant to every subject; whoever applies them picks the relevant ones and says which were left out (see [Applying the aspects](#applying-the-aspects)).

## Readiness aspects

Each aspect is a question, followed by indicators. Indicators are meant to be observed or computed, not guessed. Where a target is given it is a reference point for "ready", not a universal rule.

### 1. Automatability

*Can every action needed to deliver and operate the software be performed without a human?*

Actions include build, package, deploy, configure, start, stop, upgrade, rollback and test.

- Share of lifecycle actions available through CLI, API or files (target: 100%).
- Manual steps per release (target: 0).
- Unattended operation: no interactive prompts, meaningful exit codes, machine-readable output.

### 2. Configurability

*Can all configuration be set by automation, or does it need the product's user interface?*

- Share of configuration items settable via files, environment variables or API.
- Number of settings reachable only through a UI (target: 0).
- The same artifact can be promoted across environments, with only configuration differing (yes/no).
- Environment-specific identifiers (GUIDs, record IDs, connection IDs) embedded in the artifact (target: 0). See [Environment-specific identifiers inside the code](#environment-specific-identifiers-inside-the-code).
- Secrets are injected at deploy time and not baked into the artifact (yes/no).

### 3. Source-based assets

*Are the assets text, versionable in git, diffable and mergeable?*

- Share of assets stored as text. A binary or opaque asset blocks diff and merge.
- Direction of flow: assets are authored in git and flow to the system, not authored in the system and exported to git (yes/no). See [Git as a mirror of a SaaS tenant](#git-as-a-mirror-of-a-saas-tenant).
- Each asset can be authored and reviewed on its own, without regenerating a bundle (yes/no).
- Reproducibility: the same commit yields an equivalent artifact.
- Merge-conflict rate on shared assets.

### 4. Repeatability and idempotency

*Can an environment be provisioned, and a deployment re-run, to the same resulting state?*

- Idempotent deployment (yes/no).
- Environment provisioning time, from nothing to running.
- Drift detection: desired and actual state can be compared by automation (yes/no).

### 5. Testability

*Can tests run automatically in the pipeline, in isolation, in ephemeral environments?*

- Share of tests (unit, service, contract, integration) that run in the pipeline.
- Test duration and flakiness rate (fraction of runs whose outcome changes without a code change).
- Availability of mocks or test doubles for dependencies.
- Existence of a headless, ephemeral way to run the software for tests.
- Coverage may be recorded, but on its own it says little about the quality of the tests.

### 6. Deployability

*Can a change be released with low risk, little downtime and a quick way back?*

- Deployment time and downtime per deployment (target: zero downtime, or a small known window).
- Support for rolling, blue/green or canary releases. This requires that old and new versions can run side by side, with backward-compatible schemas, APIs and messages.
- Rollback or roll-forward time.
- Deploy decoupled from release, for example with feature toggles.
- State handling: stateless runtime, or state migrated by automated, versioned scripts.

### 7. Modularity

*Can parts be changed and deployed independently?*

- Number of independently deployable units.
- Deployment unit size and blast radius of a change.
- Number of components that must change or deploy together.

### 8. Observability

*Can the pipeline verify automatically that a release is healthy?*

- Health and readiness endpoints.
- Structured logs, metrics and traces that automation can query.
- Time to detect a failed release.

### 9. Pipeline friendliness

*Does the software fit fast feedback loops and ephemeral infrastructure?*

- Build time and commit-to-feedback time (reference: under 10 minutes for the CI feedback).
- Image size and start-up time.
- Licensing or activation constraints that block parallel or ephemeral environments.

### 10. Security and compliance

*Can security checks and audit evidence be produced automatically?*

- Automated vulnerability scanning and time to patch.
- Traceability of every change to a commit and an approval.
- Age of base images and packages.
- Availability of a software bill of materials (SBOM) and artifact provenance.

### 11. Branchability and mergeability

*Can parallel lines of work be created, integrated and reconciled cheaply and safely, so that short-lived branches and trunk-based development are practical?*

- Merge-conflict rate and conflict resolution time.
- Branch lifetime (target: hours to a few days).
- Share of conflicts resolvable by a text merge, without the product's UI or tooling.
- Branch-specific names, ports, identifiers or paths hard-coded in assets (target: 0).
- A branch's build and tests can run without touching shared resources.

### 12. Ephemeral agents and environments

*Can a CI run create its own agent and environment from code, run the tests, publish the outcome and discard everything?*

- Environment creation time, from nothing to ready for tests.
- Teardown completeness: resources left after a run (target: 0).
- Parallel environments are possible without collisions on license, port, hostname, storage or database.
- Cost of one CI run in compute, licenses and time.
- A run is reproducible from the commit id alone.
- The outcome (logs, reports, test results) is retrievable after the environment is gone.

Aspects 11 and 12 are discussed further in the sections below.

## Rating scale

When a subject is assessed against an aspect, it is rated:

- **Full**: supported without manual intervention.
- **Partial**: supported with limitations. A short note records the limitation and the workaround, for example "configurable via API but requires a restart" or "assets are binary, so no meaningful diff".
- **None**: manual intervention is required.

A rating should cite what was observed, such as a command, an API or a documented behavior, so that it can be checked and re-checked when the technology evolves.

## Branching strategies

The aspects determine which branching strategy a technology can sustain. The strategy in turn determines how much of aspects 11 and 12 is required.

| Strategy | Typical branch lifetime | What it demands from the technology |
|---|---|---|
| Trunk-based development | Hours to a day, or direct commits to trunk | Text-mergeable assets, fast pipeline, feature toggles to decouple deploy from release, very reliable automated tests |
| Short-lived feature branches with pull/merge requests | Days | A full CI run per branch in an ephemeral environment, no shared mutable resources between branches |
| Release branches (maintenance of several versions) | Weeks to years | Fixes can be merged or cherry-picked across branches; builds reproducible per branch; environments per supported version |
| Long-lived environment branches (dev, test, prod) | Permanent | Generally a poor fit: they promote different content per environment and invite drift. Prefer one artifact promoted through environments with configuration kept separate |

Questions that reveal the branching readiness of a technology:

- Can two branches be built, deployed and tested at the same time without interfering? What is shared?
- Can changes made on two branches be merged as text, or does the product's UI or a proprietary tool have to reconcile them?

## Ephemeral agents and environments

For each push or pull request the pipeline should be able to do the following, with nothing manual:

1. Create an agent (container or VM) from a versioned definition.
2. Create the environment: runtime, dependencies (database, messaging, test doubles) and configuration, all from code.
3. Build and deploy the branch's artifact into it.
4. Run the tests and collect the outcome.
5. Publish the outcome (test reports, logs, artifacts) to a place that outlives the environment.
6. Destroy the agent and the environment, whatever the outcome.

Properties that make a technology suitable:

- Environment-specific values come from outside the artifact (aspect 2), so one artifact runs in any environment.
- Start-up is fast and unattended, with no interactive installers or activation steps (aspects 1, 9).
- Licensing allows many short-lived, parallel instances, for instance by a license injected at start-up and released at teardown.
- Nothing is shared and mutable between runs: no common database schema, queue, file share or fixed port. Each run owns its resources or uses unique names derived from the branch or run id.
- External dependencies can be replaced by test doubles or spun up alongside, so that a run does not depend on other teams' systems.
- State is fully provisioned from code (schemas, seed data, configuration), so every run starts from the same known state (aspect 4).
- Teardown is reliable, including after failed or cancelled runs, to avoid orphaned resources and cost.

Additional questions:

- How long does it take from `git push` to a ready environment, and to a discarded one?
- What does one ephemeral run cost in compute, licenses and time?

## Permanent environments and their distance

Ephemeral environments are the preferred default, but some setups keep permanent ones (dev, test, staging, production). Where they exist, three things are worth recording:

1. **Permanent environments: yes / no.** If no, the remaining points do not apply. If yes, list them and say why they cannot be ephemeral (data volume, hardware, licensing, external connections, organizational gates).
2. **Distance between environments at any moment in time.** The lower and the more visible, the lower the risk of a release.
   - Commits ahead/behind between what is deployed in environment A and in environment B (for example `git rev-list --left-right --count A...B`).
   - Number of files and lines that differ between the two versions (for example `git diff --stat A B`).
   - Promotion lag: age of the oldest change that is in the upstream environment but not yet in the downstream one.
   - Number of distinct artifact versions deployed across environments (target: one version moving through, not one version per environment).
   - Configuration distance, measured separately from code distance: number of settings that differ between environments, and how many of those differences are intended (endpoints, credentials, sizing) versus accidental.
   - Whether the distance can be determined by automation, or only by inspecting each environment by hand.
3. **How the distance is managed.**

   | Mechanism | How the environment's content is defined | Typical consequence |
   |---|---|---|
   | Branch per environment | Each environment has its own long-lived branch; promotion is a merge | Distance is visible in git, but content diverges, merges conflict and fixes get lost; high drift risk |
   | Tag or pointer in the trunk history | One trunk; each environment points to a tag, commit or version; promotion moves the pointer | Distance is a clear, queryable number of commits; favors trunk-based development |
   | Artifact version promotion | Build once, deploy the same versioned artifact with environment-specific configuration | Code distance reduces to a version difference; configuration is handled separately |
   | Folder or overlay per environment in the trunk | Environment-specific configuration lives in directories next to shared code (GitOps style) | Configuration distance visible as a diff of folders; code identical |
   | Manual or undocumented | Changes applied directly in each environment | Distance unknown, drift expected |

   Also record who or what moves the pointer (pipeline, approval gate, person) and whether hotfixes made in one environment are guaranteed to flow back to trunk.

## Outcome metrics

The aspects above are properties of the technology. Whether continuous delivery works in practice is measured by the DORA metrics:

- **Deployment frequency**
- **Lead time for changes** (commit to production)
- **Change failure rate**
- **Time to restore service**

Rough correspondence between aspects and outcomes:

| Outcome | Aspects that influence it most |
|---|---|
| Deployment frequency | 1 Automatability, 9 Pipeline friendliness, 11 Branchability, 12 Ephemeral environments |
| Lead time for changes | 1, 5 Testability, 9, 11, 12 |
| Change failure rate | 4 Repeatability, 5 Testability, 6 Deployability, 7 Modularity |
| Time to restore service | 6 Deployability, 8 Observability, 4 Repeatability |

The correspondence is a guide for where to look when an outcome is poor, not a formula.

## Anti-patterns

Some setups look ready for continuous delivery because they use the right words or tools, but fail the aspects above. They are listed here as negative examples, so that they can be recognized when assessing a subject.

### Git as a mirror of a SaaS tenant

**Description.** The product's documentation or the project's governance says "git is the source of truth". In reality, the code is authored in an opaque SaaS tenant (or a UI-only product). Periodically, or on demand, the content is exported, as a zip file or as a set of text files that cannot be authored separately, and pushed to git.

**Why it is an anti-pattern.**

- **The real source of truth is the tenant.** Git holds a copy made after the fact. Anything that depends on the tenant being authoritative (running software, current behavior) is not described by git, only approximated by it.
- **The exported content is an artifact, not a source.** A source is what a human or tool authors and from which artifacts are derived. Here the bundle is derived from the tenant's internal state and cannot be authored in git. It belongs in an artifact repository, with the properties that fit artifacts: immutability, versioning, retention policy, checksums and provenance.
- **Git adds little and misleads.** Diff and merge are the reasons to use git. A zip file has neither. A set of text files that are generated and interdependent (internal identifiers, ordering, timestamps, cross-references) gives noisy diffs, reviews of generated output instead of intent, and unsafe merges. History shows who ran the export, not who made the change or why.
- **It violates the framework constraint** that git contains only text, authored as text ("code is text"). A zip file is a binary. Generated text that cannot be authored is a binary in disguise.
- **It blocks the other readiness aspects.**
  - Branching (aspect 11): a branch in git does not create a branch of the tenant. Parallel work means copies of tenants.
  - Ephemeral environments (aspect 12): without a way to author outside the tenant, there is nothing to deploy into a throwaway environment except the exported bundle.
  - Drift (aspect 4): changes in the tenant and the contents of git diverge silently, and nothing detects it.
  - Rollback (aspect 6): checking out an old commit does not restore the tenant. Importing it may be partial, non-idempotent, or unable to remove what was added later.

**Diagnostic test: the round trip.** Take a fresh, empty tenant (or the nearest equivalent). Deploy from git into it, using only automation. Then ask:

1. Is the result functionally the same as the original?
2. Could a change have been made in git alone (a text edit, a pull request) and then deployed, without ever opening the tenant's UI?
3. If the tenant is deleted tomorrow, can it be rebuilt from git alone?

If the answer to any of these is no, git is a backup or audit copy and not the source of truth. Say so honestly.

**What is acceptable.**

- Keeping the exports as **versioned, immutable artifacts in an artifact repository**, with metadata (tenant, exported at, exported by, tool version, checksum), and promoting those artifacts between tenants through automation. This is the "automated promotion with opaque artifacts" hybrid in [Software Delivery Strategies](./SoftwareDeliveryStrategies.md). It is a recognized transitional strategy, provided it is called by its name.
- Keeping an export in git **explicitly as an audit or backup copy**, when the content is text, the volume is small, and nobody pretends it is the authority. Even so, the preferred place for it remains the artifact repository.
- Products where **git synchronization is bidirectional and git wins**: the product reads its content from git, authoring in git is supported, and the tenant can be recreated from the repository. Here the aspect is rated on its merits, and the round trip test is expected to pass.

**Rating.** For aspect 3, a subject that fails the round trip test is rated **None** or at best **Partial**, with the note "export mirrored into git; tenant is the source of truth".

### Environment-specific identifiers inside the code

**Description.** The platform assigns a generated identifier (a GUID, a record ID, a connection ID) to each object when it is created in an environment. The same logical object (a connection, a queue, a user group, a lookup value, a referenced component) therefore has a different identifier in dev, test and production. Code or configuration that refers to the object contains the identifier, so the content of the artifact depends on the environment it was exported from or is going to.

**Why it hurts continuous delivery.**

- **Build once, deploy many is impossible.** The artifact tested in one environment is not the one that can run in the next. Either someone edits the identifiers on the way (the code "shifts" during deployment), or the artifact is rebuilt or re-exported per environment. In both cases what is deployed was never tested as such.
- **Promotion becomes a transformation step.** A mapping table, a script or a manual find/replace must translate identifiers per target. This step is itself untested code, easy to get wrong, and it often fails silently by pointing at the wrong object or at a missing one.
- **Diffs and merges become noisy.** Between environments, and between exports, identifiers change while intent does not. Reviewers cannot tell a real change from identifier churn, and the "distance" between environments (see [Permanent environments and their distance](#permanent-environments-and-their-distance)) cannot be measured meaningfully.
- **Ephemeral environments are weakened.** A new environment creates new objects with new identifiers, so every throwaway environment needs the mapping applied, or the code fails to find its dependencies.
- **Configuration and code are mixed.** The identifier is environment configuration (aspect 2) living inside the code, so the separation the framework relies on is lost.
- **Rollback and drift detection suffer** (aspects 4 and 6): restoring an old version may point at identifiers that no longer exist, and a comparison of environments reports differences everywhere.

**Diagnostic tests.**

1. Take the artifact from one environment and diff it against the same logical version in another. Count the differences. Every one that is not intended configuration is an environment-specific identifier.
2. Can the artifact be deployed into a freshly created, empty environment without any text transformation, supplying environment values only as external configuration?
3. Does a documented mapping exist between logical names and identifiers, and is it applied by automation?

**Mitigations, from best to worst.**

1. **Refer to objects by stable logical names**, resolved at deploy or run time against the environment (for example named connections, aliases, environment variables, a lookup by a business key). The platform supports it, or an indirection layer is added.
2. **Choose identifiers deterministically**: the identifier is derived from the logical name or set by the pipeline when the object is created, so that it is the same in every environment.
3. **Keep identifiers outside the artifact**: the artifact contains placeholders and the pipeline injects the values from versioned, per-environment configuration.
4. **Transform at deployment** through an automated, versioned and tested mapping. This is a workaround. It must be rated Partial and the mapping treated as code.
5. **Edit by hand or rebuild per environment.** Not acceptable for continuous delivery.

**Rating.** A subject whose artifact embeds environment-specific identifiers with no automatable mitigation is rated **None** for the "same artifact promoted" indicator of aspect 2. With mitigation 3 or 4 in place it is **Partial**, with the mapping noted as the workaround. With mitigation 1 or 2 it can be **Full**.

### Other negative examples

| Anti-pattern | What it looks like | Aspects it undermines |
|---|---|---|
| Binaries or compiled output in git | `.jar`, `.class`, zip files, installers committed to the repository | 3 Source-based assets, 10 Security and compliance (no provenance), 11 Branchability |
| Branch per environment | `dev`, `test`, `prod` long-lived branches, promotion by merge | 11 Branchability, 4 Repeatability (drift), 2 Configurability (content differs per environment) |
| Rebuild per environment | Each environment builds its own artifact from the same commit | 2 Configurability, 4 Repeatability, 6 Deployability (what was tested is not what is deployed) |
| Click-ops configuration | Settings changed in the product's UI, documented with screenshots or a checklist | 1 Automatability, 2 Configurability, 4 Repeatability |
| Shared mutable test environment | One long-lived environment used by all branches and teams | 5 Testability, 11 Branchability, 12 Ephemeral environments |
| Secrets in the repository or in the artifact | Passwords or keys committed, or inside an exported configuration | 2 Configurability, 10 Security and compliance |
| Manual gate disguised as automation | Pipeline waits for a person to execute a step in a UI, then to click "continue" | 1 Automatability, 9 Pipeline friendliness |
| Identifiers that differ per environment inside the artifact | GUIDs or IDs of connections, groups or referenced objects embedded in the code, rewritten on each deployment | 2 Configurability, 4 Repeatability, 12 Ephemeral environments |
| Opaque promotion by copy | Content copied from one environment to the next through the product's own export/import feature, without an artifact in between | 3 Source-based assets, 4 Repeatability, 10 Security and compliance |

## Applying the aspects

Aspects are meant to be picked per subject. A recommended way of working:

1. Select the aspects relevant to the subject and record why the others are not (for example, "modularity: not applicable, a single configuration setting").
2. For each selected aspect, record the rating (Full, Partial, None), the observation that supports it, and the workaround if Partial.
3. State the branching strategy and the environment model (ephemeral only, or permanent environments with their distance management) the rating assumes.
4. Revisit ratings when the technology version changes. A rating without a version is incomplete.

The [Patterns dictionary](../../1c-patterns/README.md) applies this approach to webMethods technology patterns.

**Note**: This article has been written with assistance from [Bob](https://www.ibm.com/products/bob).
