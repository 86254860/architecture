# HyperFleet Release Process - Spike Report

**Document Status:** Draft
**Date:** 2026-01-29
**Last Updated:** 2026-03-19

---

## Executive Summary

This spike report defines a comprehensive release process for HyperFleet (API service, Sentinel, and Adapter Framework). The proposed process balances agility with stability, leveraging existing Prow infrastructure while establishing clear gates, workflows, and artifacts for production releases.

**Key Recommendations:**
- **Hybrid release cadence:** Regular 3-week (1-sprint) releases for quality + ad-hoc releases for urgent requirements
- Git branching strategy with release branches and cherry-pick workflow
- **Independent component versioning:** Each component (API Service, Sentinel, Adapter Framework) maintains its own semantic version
- Validated HyperFleet releases defined by compatibility-tested component version combinations
- Multi-gate release readiness criteria including automated and manual validation
- Structured bug triage workflow post-code freeze
- Comprehensive release artifacts including container images, Helm charts, and documentation
- **Start with Prow + manual release for MVP**, plan Konflux migration (with Enterprise Contract Policy support) for Post-MVP

---

## 1. Release Entry Criteria

These criteria determine when the development team can initiate the release process and enter code freeze.

### 1.1 Feature Completeness Criteria

**Feature Freeze Gate:**
- ✓ All planned features for the release milestone are code-complete
- ✓ Feature toggles are in place for any experimental or incomplete features if applicable
- ✓ Feature documentation is drafted (can be finalized during code freeze)

**Technical Debt Assessment:**
- ✓ No known security vulnerabilities rated HIGH or CRITICAL remain unaddressed (once konflux job is supported) 
- ✓ Technical debt has been reviewed and acceptable items are explicitly deferred to next release
- ✓ All deprecated APIs have migration paths documented

### 1.2 Testing & Quality Gates

- **CI/CD Pipeline Health:** Prow CI pipeline is green for all components on the main branch, validating:
  - Unit tests: Passing consistently (coverage ensured by pre-submit jobs, not part of release gating)
  - Integration tests: Passing consistently
  - E2E tests: Critical user journeys validated
  - Performance regression tests: No degradation >10% vs. previous release (once performance testing is supported)

- **Build**:
  - Container images: Build successfully for all target architectures
  - Helm charts: Package without errors

### 1.3 Cross-Component Dependencies

**Version Compatibility:**
- ✓ Each component uses independent semantic versioning (see Section 2.5)
- ✓ HyperFleet Release X.Y defines a validated, compatibility-tested set of component versions
- ✓ Compatibility matrix documented showing which component versions work together
- ✓ Breaking changes (if any) are documented with migration guides and version requirements
- ✓ Backward compatibility: Each component supports N-1 version upgrade paths independently
- ✓ Cross-component API contracts validated during integration testing

### 1.4 Documentation Readiness

- ✓ Release notes draft exists with major features listed
- ✓ Known issues and limitations are documented
- ✓ Upgrade/migration documentation is drafted (if applicable)
- ✓ API documentation is up-to-date

### 1.5 Organizational Readiness

- ✓ Release Owner identified and assigned
- ✓ Stakeholder communication plan is in place

**Decision Point:** When all criteria above are met, the Release Owner can call for Feature Freeze and transition to code stabilization phase.

---

## 2. Code Freeze and Branching Strategy

### 2.1 Branching Model

HyperFleet follows a **release branch workflow** based on Kubernetes and OpenShift best practices:

```text
main (development branch)
  │
  │ Active Development Phase
  │
  ├─── release-X.Y (branch created at Feature Freeze)
  │      │
  │      │ Stabilization Phase (bug fixes only)
  │      │
  │      │ Code Freeze (critical fixes only)
  │      │
  │      ├─── vX.Y.0-rc.1 (tag - Release Candidate 1)
  │      ├─── vX.Y.0-rc.2 (tag - Release Candidate 2, if needed)
  │      └─── vX.Y.0 (tag - GA Release)
  │
  │ (main branch continues with X.Y+1 development)
  │
  └─── (next release cycle)

After GA:
release-X.Y (branch maintained post-release)
  │
  ├─── vX.Y.1 (tag - Z-stream/patch release, NO RC)
  ├─── vX.Y.2 (tag - Z-stream/patch release, NO RC)
  └─── ... (support window: 6 months)
```

**Note on Z-Stream (Patch) Releases:**
- Z-stream releases (vX.Y.1, vX.Y.2, etc.) **do not require Release Candidate (RC) tags**
- These releases go directly from testing to GA tag
- Scope limited to bug fixes, security patches, and minor improvements (no new features or breaking changes)
- Faster turnaround for low-risk fixes while maintaining quality gates

### 2.2 Timeline and Freeze Process

**Sprint-Based Release Cycle (3 weeks / 1 sprint):**

**Timeline Breakdown:**
- **Weeks 1-2 (Development Phase):** Active feature development, normal PR process on `main` branch
  - Continuous testing on all PRs (unit, integration, linting)
  - Documentation written alongside code
  - Features merged as they're completed
- **Week 3 (Stabilization & Release Phase):**
  - Feature Freeze - create `release-X.Y` branch, cut RC.1
  - `main` branch reopens immediately for next release development
  - Bug fixes cherry-picked from `main`
  - Full E2E test suite execution (automated + manual)
  - Documentation finalization
  - Code Freeze - only critical/blocker fixes with Release Owner approval
  - Final validation and sign-off
  - GA release tagged and published with all artifacts

**Note:** This 3-week cadence is designed for HyperFleet as a new product requiring rapid feedback cycles with pillar teams. Since CI/CD automation is currently being built, the Stabilization & Release Phase (Week 3) may take longer in the initial releases. As the product matures and automation capabilities improve, the team should continuously refine and optimize the release cadence based on actual data and lessons learned from each release cycle.

### 2.3 Code Freeze Mechanics

**Feature Freeze:**
```bash
# Release Owner creates release branch from main (per component repo)
git checkout main
git pull origin main
git checkout -b release-1.5
git push origin release-1.5

# Create first release candidate (per component version)
git tag -a v1.5.0-rc.1 -m "API Service RC1 for v1.5.0"
git push origin v1.5.0-rc.1
```

**After Feature Freeze:**
- `main` branch reopens immediately for next release (v1.6) development
- All changes to `release-1.5` branch must be cherry-picked from `main` or created as hotfix PRs
- Release Owner reviews and approves all PRs to release branch

**Code Freeze:**
- Only Critical or above bug fixes allowed into release branch (Blocker, Critical)
- Each fix requires:
  - Bug severity: Critical or above (Blocker, Critical)
  - Release Owner approval
  - Successful test run in Prow
  - Risk assessment documented

**Cherry-Pick Process:**

**Standard Fix (Preferred):**
1. Create PR to fix bug in `main` branch → Merge
2. Cherry-pick the fix to release branch via new PR → Release Owner approves

**Release-Specific Fix (Only if bug doesn't exist in main):**
1. Create PR directly to `release-X.Y` branch → Release Owner approves

### 2.4 Release Branch Maintenance

**Support Policy:** 6 months with lifecycle stages

Every release receives 6 months of support from its GA date, divided into two phases:

#### Phase 1: Full Support (first 3 months)
- All bug fixes and enhancements
- Patch releases for any severity (Major+)
- Active maintenance

#### Phase 2: Security Maintenance (months 3-6)
- CRITICAL and HIGH security vulnerabilities only
- Blocker bugs affecting production stability
- Limited patch releases

#### After 6 months: End of Life (EOL)
- No further updates
- Users must upgrade to supported version

#### Backport Decision Tree:
- Version < 3 months old: Backport all Major+ severity bugs
- Version 3-6 months old: Backport only CRITICAL/HIGH CVEs and Blockers
- Version > 6 months old: EOL, no backports

### 2.5 Versioning Strategy

**HyperFleet uses independent component versioning** with validated release combinations.

Each core component (API Service, Sentinel, Adapter Framework) maintains its own semantic version number, evolving at its own pace based on changes and feature development.

```text
HyperFleet Release 1.5 (validated combination):
├─ api-service: v1.5.0
├─ sentinel: v1.4.2
└─ adapter-framework: v2.0.0
```

**Rationale:**
- **Flexibility:** Components can evolve independently without forcing artificial version bumps
- **Semantic accuracy:** Version numbers reflect actual changes (v1.4.2 → v1.4.3 for bug fix, not v1.4.2 → v1.5.0)
- **Efficient releases:** Components without changes don't require new versions or rebuilds
- **Clear change tracking:** Each component's version history accurately reflects its evolution
- **Industry standard:** Aligns with microservices best practices (Kubernetes components, cloud services)

**HyperFleet Release Number:**
- Defines a validated, compatibility-tested set of component versions
- Format: "HyperFleet Release X.Y" (e.g., Release 1.5, Release 1.6)
- Documented in release notes with full compatibility matrix
- Simplifies user experience: "Install HyperFleet Release 1.5" with clear component version mapping

#### 2.5.1 Branching and Tagging Rules

**Key Principles:**
1. **Independent branching:** Each component creates release branches based on its own version (e.g., `release-1.5`, `release-2.0`)
2. **Independent tagging:** Each component tags releases according to its semantic version (e.g., `v1.5.0`, `v1.4.2`, `v2.0.0`)
3. **Selective releases:** Only components with changes create new release branches and tags

**Why?** Components evolve at their own pace, and version numbers should accurately reflect actual changes.

**Component-Specific Releases:**

Between HyperFleet releases, individual components can issue releases independently:

```bash
# Example: Critical bug found in Sentinel after HyperFleet Release 1.5 GA
# Current: HyperFleet Release 1.5 (API v1.5.0, Sentinel v1.4.2, Adapter v2.0.0)

# Sentinel creates patch release v1.4.3
cd openshift-hyperfleet/sentinel
git checkout release-1.4
# Apply fix, test
git tag -a v1.4.3 -m "Sentinel v1.4.3 - Hotfix for metrics bug"
git push origin v1.4.3

# Result: Users can upgrade just Sentinel v1.4.2 → v1.4.3 without full release
# No new HyperFleet Release number needed for single-component release
```

**When to Create Component Release vs HyperFleet Release:**

**Component Release:**

Create a component release only for isolated, fully backward-compatible fixes within a single component that do not impact APIs, schemas, cross-component behavior, or coordinated platform upgrades.

**HyperFleet Release:**

Create a HyperFleet release whenever a change affects supported platform users, introduces cross-component compatibility or contract implications, delivers security or critical stability fixes, or requires coordinated, platform-wide, validated upgrades.

**Supporting Repository Branching for HyperFleet Releases:**

When creating a HyperFleet release, the following supporting repositories also participate in the release process:

**For Major/Minor Releases (e.g., HyperFleet Release 1.5):**

1. **hyperfleet-e2e** - E2E test suites
   - Create `release-1.5` branch at Feature Freeze
   - Contains E2E tests validating the specific component combinations for this release

2. **hyperfleet-infra** - Infrastructure configurations
   - Create `release-1.5` branch at Feature Freeze (if infrastructure changes needed)
   - Contains deployment scripts, cluster configs aligned with this release

3. **hyperfleet-release** - Release coordination and documentation
   - Create `release-1.5` branch at Feature Freeze
   - Contains release notes, compatibility matrices, installation guides
   - Tag the release: `release-1.5` at GA

**For Patch Releases (e.g., HyperFleet Release 1.5.1, 1.5.2):**

Supporting repositories **do not create new branches** for patch releases:

1. **hyperfleet-e2e, hyperfleet-infra**
   - Stay on existing `release-1.5` branch
   - Commit updates/fixes to the same branch as needed

2. **hyperfleet-release**
   - Stay on existing `release-1.5` branch
   - Update release notes and compatibility matrix
   - **Create new tag** for each patch: `release-1.5.1`, `release-1.5.2`

   ```bash
   # Example: HyperFleet Release 1.5.1 (patch)
   cd openshift-hyperfleet/hyperfleet-release
   git checkout release-1.5

   # Update release notes with patch changes
   # (e.g., API Service v1.5.0 → v1.5.1)

   # Tag the patch release
   git tag -a release-1.5.1 -m "HyperFleet Release 1.5.1 (API v1.5.1, Sentinel v1.4.2, Adapter v2.0.0)"
   git push origin release-1.5.1
   ```

**Rationale:** Patch releases are incremental updates on the same base release. Supporting repos use the same branch infrastructure with updated content and new tags to mark each patch version.

#### 2.5.2 Practical Example: HyperFleet Release 1.5

**Scenario:**
- API Service: **Major feature** (new GitOps integration) → Version bump to v1.5.0
- Sentinel: **Bug fixes only** → Patch version bump to v1.4.2
- Adapter Framework: **Breaking changes** → Major version bump to v2.0.0

**At Feature Freeze - Create Component-Specific Release Branches:**

```bash
# API Service - MINOR version bump (new features)
cd openshift-hyperfleet/api-service
git checkout -b release-1.5 && git push origin release-1.5

# Sentinel - PATCH version bump (bug fixes only)
cd openshift-hyperfleet/sentinel
git checkout -b release-1.4 && git push origin release-1.4
# (or cherry-pick to existing release-1.4 if it exists)

# Adapter Framework - MAJOR version bump (breaking changes)
cd openshift-hyperfleet/adapter-framework
git checkout -b release-2.0 && git push origin release-2.0
```

**At GA Release - Tag Each Component with Its Own Version:**

```bash
# API Service - Tag v1.5.0 (new minor version)
cd openshift-hyperfleet/api-service
git checkout release-1.5
git tag -a v1.5.0 -m "API Service v1.5.0 - GitOps integration"
git push origin v1.5.0

# Sentinel - Tag v1.4.2 (patch release)
cd openshift-hyperfleet/sentinel
git checkout release-1.4
git tag -a v1.4.2 -m "Sentinel v1.4.2 - Memory leak fix"
git push origin v1.4.2

# Adapter Framework - Tag v2.0.0 (major version)
cd openshift-hyperfleet/adapter-framework
git checkout release-2.0
git tag -a v2.0.0 -m "Adapter Framework v2.0.0 - Plugin API v2"
git push origin v2.0.0
```

**Result:**
```text
HyperFleet Release 1.5 (validated combination):

Component Release Branches:
- api-service: release-1.5
- sentinel: release-1.4
- adapter-framework: release-2.0

Component Version Tags:
- api-service: v1.5.0
- sentinel: v1.4.2
- adapter-framework: v2.0.0

Container Images (for HyperFleet Release 1.5):
- registry.ci.openshift.org/hyperfleet/api-service:v1.5.0
- registry.ci.openshift.org/hyperfleet/sentinel:v1.4.2
- registry.ci.openshift.org/hyperfleet/adapter-framework:v2.0.0

Compatibility:
- API Service v1.5.0 requires Adapter Framework ≥ v2.0.0
- Sentinel v1.4.2 is compatible with Adapter Framework v1.x and v2.x
- Full compatibility matrix documented in release notes
```

---

## 3. Release Readiness Criteria

Before declaring a release as "GA-Ready", all the following criteria must be satisfied:

### 3.0 Prow Job Configuration for Release Branches

After creating release branches for components and supporting repositories, Prow jobs must be configured to support release branch builds and testing.

**Required Prow Job Configuration:**

**1. Copy Build Jobs for Release Images**

After cutting component release branches (e.g., `release-1.5` for API Service, `release-1.4` for Sentinel):

- **Action:** Copy existing Prow jobs that build container images from `main` branch
- **Update:** Configure copied jobs to trigger on release branch commits
- **Purpose:** Enable automated image builds from release branches when fixes are merged
- **Result:** Release images automatically built and pushed to registry when release branch is updated

**2. Copy E2E Prow Jobs for Nightly Release Testing**

After cutting `hyperfleet-e2e` release branch (e.g., `release-1.5`):

- **Action:** Copy E2E test Prow jobs that run nightly on `main` branch
- **Update:** Configure copied jobs to:
  - Run against the release branch E2E test suite
  - Test the validated component version combination for this release
  - Trigger nightly to detect regressions in release branch
- **Purpose:** Continuous validation of release stability throughout support lifecycle
- **Result:** Early detection of issues in release branches before they reach users

**Best Practices:**
- Copy jobs at Feature Freeze when release branches are created
- Maintain consistent naming convention: `{job-name}-release-X.Y`
- Monitor nightly release test results for regressions
- Disable nightly jobs when release reaches EOL (after 6 months)

### 3.1 Testing & Validation (Mandatory)

**Unit & Integration Testing:**
- ✓ All unit tests passing across all components
- ✓ Integration test suite passing

**E2E Testing:**
- ✓ Critical user workflows validated (E2E test suite)
- ✓ Backward compatibility testing with N-1 version
- ✓ Installation/upgrade path tested

**Performance & Load Testing:**
- ✓ Performance benchmarks show no regression > 10% vs. previous release
- ✓ Load testing validates system handles expected production load
- ✓ Resource utilization (CPU, memory) within acceptable bounds

**Note:** Automated testing is preferred for all scenarios. If a test scenario is not yet automated, it must be executed manually before release approval.

### 3.2 Bug Severity Gates (Mandatory)

- ✓ No open bugs with severity **Normal** or above (Blocker, Critical, Major, Normal)
    - Note: `Normal` bugs do not gate **MVP releases**
- ✓ Minor bugs:  No gate, tracked for future releases

### 3.3 Documentation Completeness (Mandatory)

**Release Documentation:**
- ✓ Release notes finalized (what's new, bug fixes, breaking changes)
- ✓ Known issues and limitations documented
- ✓ Deprecation notices published (if applicable)

**Operational Documentation:**
- ✓ Installation guide updated
- ✓ Upgrade instructions complete (N-1 → N)
- ✓ Deployment runbook created and reviewed

**Technical Documentation:**
- ✓ API documentation updated (if APIs changed)
- ✓ Configuration changes documented

### 3.4 Cross-Team Coordination

- ⚠ Integration validation with dependent offerings (TBD)

### 3.5 Security & Compliance

**Mandatory (All Phases):**
- ✓ Vulnerability scanning: No CRITICAL/HIGH CVEs in container images

**Post-MVP (Deferred to Konflux Migration):**
- ⚠ Supply chain security: SLSA Level 3 provenance generated
- ⚠ Software Bill of Materials (SBOM) generated
- ⚠ Container images signed with Sigstore/Cosign
- ⚠ Enterprise Contract Policy enforcement

**Note:** For MVP, focus on vulnerability scanning. Full supply chain security (SBOM, signing, provenance) will be implemented during Post-MVP Konflux migration.

### 3.6 Release Artifacts Verification (Mandatory)

- ✓ All container images built and pushed to registry
- ✓ Helm charts packaged and tested
- ✓ Git tags created with correct version
- ✓ Release artifacts checksums/signatures verified

**Gate Decision:** Only when ALL mandatory criteria are met can the release be declared GA-ready and published.

---

## 4. Bug Handling Workflow After Code Freeze

### 4.1 Bug Triage Process

When a bug is discovered after code freeze (during RC testing or late in release cycle):

```text
            Bug Reported
                 │
                 ↓
      ┌─────────────────────────┐
      │ Initial Assessment      │
      │ - Severity assignment   │
      │ - Reproducibility       │
      │ - Impact analysis       │
      └───────────┬─────────────┘
                  │
                  ↓
┌────────────────────────────────────────────┐
│          Severity-Based Routing            │
├──────────────┬──────────────┬──────────────┤
│ Blocker/     │    Major     │ Normal/Minor │
│ Critical     │              │              │
└──────┬───────┴──────┬───────┴──────┬───────┘
       │              │              │
       ↓              ↓              ↓
  [FIX NOW]    [TRIAGE MEETING]  [DEFER]
```

### 4.2 Decision Framework

**For Blocker/Critical bugs:**
1. **Immediate Action:** Developer assigned within 2 hours
2. **Fix & Test:** Root cause analysis, fix implementation, automated tests added
3. **Cherry-Pick:** Fix merged to `main` first, then cherry-picked to release branch
4. **New RC:** Cut new release candidate (e.g., v1.5.0-rc.3)
5. **Regression Testing:** Full test suite re-run
6. **Time Box:** If fix takes > 24 hours, consider release delay or degrading severity

**For Major bugs:**
1. **Before Code Freeze:** Major severity bugs must be fixed before GA release
   - **Release Owner Assessment:** Evaluate impact, risk, complexity, and timeline
   - **Fix & Include:** Implement fix and cherry-pick to release branch
   - **If Not Fixable in Timeline:**
     - Consider release delay to allow fix completion
     - OR downgrade severity if impact assessment justifies (requires stakeholder approval and documented rationale)
2. **During Code Freeze:** Major bugs must be either:
   - **Escalated to Critical:** With documented justification showing blocker-level impact to be included in the release
   - **Deferred to Next Patch:** Scheduled for the next patch release (e.g., v1.5.1) if impact does not warrant Critical escalation

**For Normal/Minor bugs:**
- Default: Defer to next patch release or next minor release
- Track in backlog for future releases

### 4.3 Post-Code Freeze PR Approval Process

All PRs to release branch after code freeze require:

1. **Justification:** PR description must include:
   - Bug severity and impact
   - Why fix cannot wait for patch release
   - Risk assessment (What could break?)
   - Test coverage added/modified

2. **Approval Chain:**
   ```text
   Developer → Code Review → Release Owner → Automated Tests → Merge
   ```
   - Minimum 2 approvals (1 technical reviewer + Release Owner)
   - Prow tests must be green
   - No approval bypasses allowed

3. **Communication:**
   - Post to release Slack channel for visibility
   - Update release tracking issue
   - Notify stakeholders if fix delays GA timeline

### 4.4 Hotfix Workflow (Post-GA)

For bugs discovered after GA release:

```bash
# Example assumes Sentinel component (v1.4.x)
# Create hotfix branch from component release tag
git checkout -b hotfix-1.4.3 v1.4.2

# Make fix, test, commit
git commit -m "Fix critical bug in Sentinel component"

# Merge to release branch
git checkout release-1.4
git merge --no-ff hotfix-1.4.3

# Tag component patch release
git tag -a v1.4.3 -m "Patch release v1.4.3"
git push origin release-1.4 --tags

# Cherry-pick to main if applicable
git checkout main
git cherry-pick <commit-sha>
```

**Hotfix Release Timeline:**
- Blocker/Critical severity: Patch release within 48 hours
- Major severity: Patch release within 1 week

### 4.5 Release Recovery Strategy

**HyperFleet uses a roll-forward recovery strategy for MVP releases.**

#### 4.5.1 Roll-Forward (Primary Strategy)

When issues are discovered in a GA release, the default recovery path is to **fix forward** via a patch release:

**Process:**
1. Identify and fix the issue in `main` branch
2. Cherry-pick fix to affected release branch
3. Cut patch release (e.g., v1.5.0 → v1.5.1)
4. Deploy patch following standard deployment procedures

**Timeline:**
- Blocker/Critical: Patch release within 48 hours
- Major: Patch release within 1 week

**Advantages:**
- Simpler testing scope (only test the fix)
- No database migration reversal complexity
- Maintains forward version progression
- Faster response for critical issues

#### 4.5.2 Rollback Support (Post-MVP)

**Status:** Deferred to Post-Q1 (separate epic required)

**Reference:** See [versioning trade-offs documentation](https://github.com/openshift-hyperfleet/architecture/blob/main/hyperfleet/docs/versioning-trade-offs.md#4-database-migration-and-rollback-procedures-post-mvp) for detailed rollback considerations.

**Decision Point:** Evaluate rollback support necessity after MVP based on:
- Incident frequency and severity
- Customer requirements for rollback capabilities
- Database schema stability
- Testing infrastructure maturity

---

## 5. Release Cadence

**Regular Releases:** Every 3 weeks (1 sprint) - ~17 releases per year
- Weeks 1-2: Active development
- Week 3: Feature freeze, stabilization, testing, GA release
  - Note: Duration may vary as CI/CD automation is being built

**Ad-Hoc Releases:** As needed for urgent requirements (3-5 days)
- Triggers: Critical bugs, security vulnerabilities, urgent business needs
- Reduced testing scope (unit, integration, E2E for affected components only)
- Release Owner approval required

**Cadence Refinement:**
As HyperFleet matures and based on retrospective data, the team may adjust the release cadence:
- If automation is strong and quality metrics support it, consider faster cycles
- If team coordination or integration complexity requires it, consider extending to 4 weeks
- Revisit cadence after every several releases during retrospectives

---

## 6. Release Artifacts and Deliverables

### 6.0 Release Repository

**Create: `openshift-hyperfleet/releases` repository**

A dedicated release repository serves as the single source of truth for all HyperFleet releases.

**Purpose:**
- Centralized release notes, installation guides, and upgrade documentation
- Defines validated component version combinations for each HyperFleet Release
- Release tracking issues and automation scripts
- User-facing documentation via GitHub Pages (optional)

**What goes in the release repository:**
- HyperFleet Release tags: `release-1.5`, `release-1.6` (marking validated component combinations)
- Release notes for each HyperFleet Release (`releases/release-1.5/release-notes.md`)
- Compatibility matrix for each release (which component versions work together)
- Installation and upgrade guides
- Links to component container images and artifacts
- Aggregated changelog from all components
- Release automation scripts

**What stays in component repositories:**
- Source code
- Component-specific git tags (e.g., `v1.5.0`, `v1.4.2`, `v2.0.0`)
- Component-specific Helm charts
- Component-specific CHANGELOGs
- Development documentation

**Benefits:** Single location for offering team to find all release information, clear component version combinations for each release, cleaner separation between component development and integrated release artifacts.

### 6.1 Primary Release Artifacts

**Container Images:**
- Built automatically by Prow on release tag creation
- Published to `registry.ci.openshift.org/hyperfleet/*`
- **Image naming:** Each component uses its own independent semantic version
  ```text
  registry.ci.openshift.org/hyperfleet/{component}:v{component-version}
  ```
- **Example for HyperFleet Release 1.5:**
  - `registry.ci.openshift.org/hyperfleet/api-service:v1.5.0`
  - `registry.ci.openshift.org/hyperfleet/sentinel:v1.4.2`
  - `registry.ci.openshift.org/hyperfleet/adapter-framework:v2.0.0`

  (Each component has its own version reflecting its actual changes per Section 2.5)

**Helm Charts:**
- Each component has its own Helm chart in component repositories
  - Note: Adapter Framework base chart is designed for overlay usage by business adapters, not standalone deployment
- Note: Umbrella chart strategy (hyperfleet-chart repo) is under discussion

**Git Tags:**
- Git tags created independently in component repositories: `vX.Y.Z` (reflecting each component's version)
- HyperFleet Release tag created in `openshift-hyperfleet/releases` repository: `release-X.Y` (single source of truth for the validated component combination - see Section 6.0)
- GitHub Releases created from tags with release notes and compatibility matrix

### 6.2 Documentation Deliverables

#### 6.2.1 Release Notes

**Required Sections:**
```markdown
# HyperFleet Release 1.5

## Overview
Brief description of release theme and major highlights

## What's New
### New Features
- **API Service v1.5.0:** GitOps integration for ROSA deployments
- **Adapter Framework v2.0.0:** New Plugin API v2 with enhanced extensibility

### Enhancements
- **API Service v1.5.0:** OAuth 2.1 support, improved authentication flow
- **Sentinel v1.4.2:** Performance improvements in metrics collection

## Breaking Changes
- **Adapter Framework v2.0.0:** Plugin API v2 (migration guide: docs/plugin-migration-v2.md)
  - Requires business adapters to update to Plugin SDK v2.0+
- **API Service v1.5.0:** Deprecated `/v1/legacy-auth` endpoint removed

## Bug Fixes
- **Sentinel v1.4.2:** Fixed memory leak in metrics collector (#234)
- **Sentinel v1.4.2:** Fixed race condition in concurrent monitoring (#256)
- **API Service v1.5.0:** Resolved authentication token refresh issue (#198)

## Known Issues
- **Sentinel:** Metrics dashboard may show delay > 5 seconds under heavy load (workaround: increase polling interval)
- **API Service:** GitOps integration requires Kubernetes 1.28+ for full functionality

## Upgrade Instructions
See [Upgrade Guide](docs/upgrade-to-release-1.5.md) for detailed instructions.

## Compatibility Matrix

**HyperFleet Release 1.5** (validated component combination)

| Component | Version | Changes | Notes |
|-----------|---------|---------|-------|
| API Service | **v1.5.0** | MINOR | New GitOps integration, OAuth 2.1, breaking change (legacy auth removed) |
| Sentinel | **v1.4.2** | PATCH | Memory leak fix, performance improvements |
| Adapter Framework | **v2.0.0** | MAJOR | Plugin API v2 (breaking), enhanced extensibility |

**Component Compatibility:**
- API Service v1.5.0 requires Adapter Framework ≥ v2.0.0
- Sentinel v1.4.2 is compatible with Adapter Framework v1.x and v2.x
- Business adapters must upgrade to Plugin SDK v2.0+ to work with Adapter Framework v2.0.0

**Platform Compatibility:**

| Platform | Supported Versions |
|----------|-------------------|
| Kubernetes | 1.26 - 1.30 |
| Helm | 3.14+ |

## Security
- **All components:** Updated dependencies to address CVE-2026-1234, CVE-2026-5678
- **API Service v1.5.0:** Enhanced OAuth 2.1 security features
- **Sentinel v1.4.2:** No security-specific changes
```

**Checklist:**
- [ ] Release notes drafted during development
- [ ] Release notes finalized before GA
- [ ] Breaking changes clearly documented
- [ ] Known issues listed with workarounds
- [ ] Upgrade instructions validated
- [ ] Published to docs site and GitHub Release

#### 6.2.2 Upgrade/Installation Guide

**Content:**
- Prerequisites (Kubernetes version, permissions, dependencies)
- Fresh installation steps
- Upgrade path from N-1 version
- Adapter Framework: Deployment via business adapter overlay (not standalone)
- Post-installation validation steps
- Troubleshooting common issues

**Checklist:**
- [ ] Installation guide updated
- [ ] Upgrade path tested (N-1 → N)
- [ ] Screenshots/examples updated
- [ ] Published to documentation site

#### 6.2.3 API Documentation

**For API Service:**
- OpenAPI/Swagger specification updated
- API reference documentation published
- Code examples for new endpoints
- Deprecation notices for old APIs

**Checklist:**
- [ ] OpenAPI spec generated from code
- [ ] API docs published (e.g., via Swagger UI)
- [ ] Examples tested and validated
- [ ] Breaking changes highlighted

#### 6.2.4 Change Log

**Format:** Keep a Changelog standard (per component)

**API Service - CHANGELOG.md:**
```markdown
# Changelog - API Service

## [1.5.0] - 2026-05-12

### Added
- New GitOps integration for ROSA deployments
- OAuth 2.1 authentication support

### Changed
- Updated authentication flow to use OAuth 2.1
- Improved API response caching mechanism

### Removed
- `/v1/legacy-auth` endpoint (deprecated since v1.3.0)

### Fixed
- Authentication token refresh issue
- Race condition in concurrent API requests

### Security
- Updated dependencies to address CVE-2026-1234
```

**Sentinel - CHANGELOG.md:**
```markdown
# Changelog - Sentinel

## [1.4.2] - 2026-05-12

### Fixed
- Memory leak in metrics collector
- Race condition in concurrent monitoring

### Security
- Updated dependencies to address CVE-2026-5678
```

**Adapter Framework - CHANGELOG.md:**
```markdown
# Changelog - Adapter Framework

## [2.0.0] - 2026-05-12

### Added
- Plugin API v2 with enhanced extensibility
- Support for async plugin initialization

### Changed
- **BREAKING:** Plugin API v2 replaces v1 (migration guide: docs/plugin-migration-v2.md)
- Improved plugin loading performance by 40%

### Removed
- **BREAKING:** Plugin API v1 support

### Security
- Updated dependencies to address CVE-2026-1234
```

**Checklist:**
- [ ] CHANGELOG.md updated in repository
- [ ] All significant changes categorized
- [ ] Links to PRs/issues included
- [ ] Security fixes clearly marked

### 6.3 Compliance and Security Artifacts

**MVP:**
- Vulnerability scanning of container images
- No CRITICAL/HIGH CVEs in release

**Post-MVP (with Konflux):**
- Enterprise Contract Policy enforcement
- SBOM generation
- SLSA Level 3 provenance
- Image signing with Sigstore/Cosign

---

## 7. Konflux vs. Prow Comparison

### 7.1 Current State: Prow

**What Works:**
- Automated CI/CD pipeline (testing, image builds)
- GitHub integration
- Team familiarity

**What's Missing:**
- SLSA provenance, SBOM generation, image signing
- Requires additional tooling for supply chain security

### 7.2 Konflux Benefits

**Supply Chain Security:**
- SLSA Level 3 provenance, SBOM, Sigstore signing (built-in)
- **Enterprise Contract Policy enforcement**
- Integrated vulnerability scanning

**Release Automation:**
- Unified build-test-release workflow
- OCI artifact management
- Policy-as-code compliance gates

### 7.3 Recommendation

**MVP Approach:**
- **Use Prow + manual release process**
- Manual steps: branching, tagging, Helm packaging, GitHub Releases
- Defer security tooling to Post-MVP
- Focus: Establish process first, automate later

**Post-MVP Migration:**
- Migrate to Konflux for automated releases
- Add Enterprise Contract Policy enforcement
- Implement SBOM, provenance, signing automation

---

## 8. Next Steps

### 8.1 MVP Tickets (First Release Preparation)

**[TICKET-1] Create HyperFleet Releases Repository**
- **Objective:** Set up `openshift-hyperfleet/releases` as single source of truth
- **Tasks:**
  - Initialize repository structure (release notes, docs, charts)
  - Set up issue templates for release tracking and ad-hoc requests
  - Document repository purpose and usage in README
- **Reference:** Section 6

**[TICKET-2] Establish Release Cadence and Calendar**
- **Objective:** Define release schedule to meet first release
- **Tasks:**
  - Decide first release date and version number
  - Publish release calendar for next 3 months (including first release)
  - Document ad-hoc release criteria and process
  - Communicate calendar to team and stakeholders
- **Reference:** Section 5

**[TICKET-3] Prepare for First Release**
- **Objective:** Document procedures and assign ownership for first release
- **Tasks:**
  - Write runbook: Release branching, tagging, image promotion
  - Write runbook: Helm chart packaging and GitHub Release creation
  - Finalize release checklist
  - Assign Release Owner for first release
  - Document Release Owner responsibilities (gatekeeper, approver)
  - Plan rotation strategy for future releases
- **Reference:** Sections 2, 3, 6

**[TICKET-4] Execute First Release**
- **Objective:** Run first release following documented process
- **Tasks:**
  - Create `release-v{X.Y}` branch at feature freeze
  - Cut Release Candidate (RC.1)
  - Execute testing per Section 3.1
  - Cherry-pick bug fixes if needed (follow Section 2 process)
  - Cut final GA release
  - Publish release artifacts and documentation
  - Conduct post-release retrospective
- **Reference:** Sections 2, 3, 6

### 8.2 Post-MVP Improvements

#### 8.2.1 Conduct Retrospectives and Identify Improvements

After completing the first few releases with manual processes, conduct retrospectives to:
- Identify workflow pain points and bottlenecks
- Determine which manual steps should be automated
- Evaluate release process effectiveness (timing, quality gates, coordination)
- Gather feedback from Release Owners, developers, and stakeholders
- Update release procedures based on lessons learned
- Prioritize automation opportunities (Helm packaging, release notes generation, GitHub Releases)

#### 8.2.2 Migrate to Konflux for Official Releases

Transition from manual Prow-based releases to Konflux for production-grade, compliant releases:

##### Why Konflux

- **Enterprise Contract Policy** enforcement for compliance and security gates
- Makes releases more official with built-in governance
- SLSA Level 3 compliance (provenance, SBOM, signing)
- Unified build-test-release pipeline with policy-as-code

##### Migration Approach

- Evaluate Konflux with pilot project (test environment, parallel builds with Prow)
- Implement Enterprise Contract Policy framework and define policies
- Migrate all components to Konflux pipelines
- Automate SBOM generation, image signing, and provenance
- Full cutover after validation

#### 8.2.3 Additional Process Improvements

Based on retrospective findings and Konflux capabilities:
- Establish automated E2E test gate as mandatory release criteria
- Create release health monitoring dashboards
- Define SLI/SLO framework for release quality metrics
- Optimize release cadence based on data (6-month review)
- Consider LTS release designation (e.g., every 4th release)

### 8.3 Success Metrics

**Track and review quarterly:**

**Release Metrics:**
- HyperFleet Release frequency (target: ~17 releases/year with 3-week cadence)
- Component patch release frequency (individual component updates between HyperFleet Releases)
- Code freeze duration (target: < 1 week, ideally 3-4 days)
- On-time delivery (target: > 80% of HyperFleet Releases on schedule)
- Stabilization phase variance (track actual vs. planned to inform process improvements)

**Quality Metrics:**
- Bug escape rate (bugs found post-GA per component and per HyperFleet Release)
- Hotfix frequency per component (target: < 2 patch releases per component between HyperFleet Releases)
- Mean time to patch critical vulnerabilities (target: < 48 hours for any component)
- Cross-component compatibility issues found in production (target: 0)

---

## 9. Release Owner Checklist

This section provides a comprehensive, phase-by-phase checklist for the Release Owner to manage the entire release cycle. Use this checklist to track progress, verify criteria, and ensure all gates are met.

**How to use this checklist:**
1. Copy the relevant phase checklist as the release progresses
2. For each item, verify the criteria using the specified approach
3. Coordinate with check owners to complete mandatory items
4. Use Slack templates (Appendix D) for team communication
5. Update the Release Tracking Issue (Appendix C) with status

---

### Phase 1: Pre-Release Preparation (Weeks 1-2 - Development Phase)

**Timing:** Sprint start through development phase
**Goal:** Ensure team is ready for upcoming release and entry criteria are being tracked

| # | Check Item | Criteria | Approach | Owner | M/O | Comm |
|---|-----------|----------|----------|-------|-----|------|
| 1.1 | Create Release Tracking Issue | Issue created in `openshift-hyperfleet/releases` repo using Appendix C template | Use GitHub issue template, fill in timeline and component versions | Release Owner | M | [Template E.1](#appendix-d-slack-communication-templates) |
| 1.2 | Announce Release Owner and Timeline | Team aware of Release Owner, Feature Freeze date, Code Freeze date, GA target | Post to #hyperfleet-releases Slack channel | Release Owner | M | [Template E.1](#appendix-d-slack-communication-templates) |
| 1.3 | Verify Planned Features Status | All milestone features are on track, component owners confirm readiness | Review component roadmaps, sync with component tech leads | Component Tech Leads | M | Daily standups |
| 1.4 | Monitor CI/CD Pipeline Health | Prow CI green on `main` branch for all components | Check Prow dashboard daily | Dev Team | M | Alert in Slack if broken > 4 hours |
| 1.5 | Track High/Critical CVEs | No CRITICAL/HIGH security vulnerabilities unaddressed | Review vulnerability scan reports weekly | Security/Dev Team | M | Escalate blockers immediately |
| 1.6 | Verify Documentation Drafts Exist | Release notes, API docs, upgrade guides in draft state | Check docs repo for draft content | Tech Writer/Dev Team | M | N/A |
| 1.7 | Review Technical Debt Backlog | Known issues reviewed, deferred items explicitly documented | Tech debt triage meeting (mid-sprint) | Tech Leads | O | Document decisions in tracking issue |

**Decision Point:** By end of Week 2, Release Owner confirms readiness for Feature Freeze (go/no-go decision)

---

### Phase 2: Feature Freeze (Start of Week 3)

**Timing:** Start of Week 3 (exact date per release calendar)
**Goal:** Lock down feature scope, create release branches, cut RC.1

| # | Check Item | Criteria | Approach | Owner | M/O | Comm |
|---|-----------|----------|----------|-------|-----|------|
| 2.1 | **Go/No-Go Decision for Feature Freeze** | All Section 1 entry criteria met (features complete, CI green, docs drafted) | Review Release Tracking Issue checklist, sync with tech leads | Release Owner + Tech Leads | M | [Template E.2](#appendix-d-slack-communication-templates) |
| 2.2 | Create Component Release Branches | Release branches created for each component with changes (e.g., `release-1.5`, `release-2.0`) per Section 2.5.1 | Execute git commands per Section 2.3 for each component | Release Owner | M | Announce branch creation in Slack |
| 2.3 | Create Supporting Repo Branches | Release branches created in `hyperfleet-e2e`, `hyperfleet-infra`, `hyperfleet-release` | Execute git commands per Section 2.5 | Release Owner | M | N/A |
| 2.4 | Configure Prow Jobs for Release Branches | Build jobs and nightly E2E jobs copied and configured per Section 3.0 | Update Prow config, submit PR to openshift/release repo | Release Owner + CI Team | M | Verify jobs run successfully |
| 2.5 | Tag Release Candidates (RC.1) | RC.1 tags created for each component (e.g., `v1.5.0-rc.1`) | Execute git tag commands per Section 2.3 | Release Owner | M | [Template E.3](#appendix-d-slack-communication-templates) |
| 2.6 | Verify RC.1 Images Built | Container images built and pushed to registry for all components | Check registry: `registry.ci.openshift.org/hyperfleet/*:vX.Y.Z-rc.1` | Release Owner | M | Alert CI team if build fails |
| 2.7 | Lock Main Branch? (Optional) | `main` branch remains open for next release development (v1.6) - no lock needed | Communicate to team that main is open for next release | Release Owner | O | [Template E.3](#appendix-d-slack-communication-templates) |
| 2.8 | Update Release Tracking Issue | Status updated to "Feature Freeze", RC.1 tagged | Update GitHub issue | Release Owner | M | N/A |

**Communication:** Post Feature Freeze announcement with RC.1 details and branch protection rules (Slack Template E.3)

---

### Phase 3: Stabilization (Week 3, Post-Feature Freeze)

**Timing:** After Feature Freeze through Code Freeze (2-4 days)
**Goal:** Execute full test suite, fix bugs, finalize documentation

| # | Check Item | Criteria | Approach | Owner | M/O | Comm |
|---|-----------|----------|----------|-------|-----|------|
| 3.1 | Execute E2E Test Suite | All critical user workflows validated, E2E tests passing | Run `hyperfleet-e2e` test suite against RC.1 component combination | QA Team | M | Report test results daily |
| 3.2 | Execute Backward Compatibility Tests | N-1 version upgrade path validated | Test upgrade from previous release to current RC | QA Team | M | Document upgrade issues |
| 3.3 | Execute Performance Regression Tests | No performance degradation > 10% vs. previous release | Run performance benchmarks (if automated) | QA Team | O (MVP) | Report regressions immediately |
| 3.4 | Bug Triage and Severity Assignment | All bugs discovered have severity assigned (Blocker/Critical/Major/Normal/Minor) | Daily bug triage meetings using Section 4.1 process | Release Owner + Dev Leads | M | [Template E.4](#appendix-d-slack-communication-templates) |
| 3.5 | Fix Blocker/Critical Bugs | All Blocker/Critical bugs resolved or downgraded | Implement fixes in `main`, cherry-pick to release branch per Section 2.3 | Dev Team | M | Track in tracking issue |
| 3.6 | Fix Major Bugs | All Major bugs resolved or explicitly deferred (with justification) | Fix in `main`, cherry-pick to release branch OR defer with documented rationale | Dev Team | M | Document deferrals in tracking issue |
| 3.7 | Cherry-Pick Process Validation | All cherry-picks follow standard process (PR from main → release branch) | Review all release branch PRs for approval and testing | Release Owner | M | Approve each cherry-pick PR |
| 3.8 | Finalize Release Notes | Release notes complete with all sections per Section 6.2.1 | Review draft, add final features/bugs, validate compatibility matrix | Tech Writer + Release Owner | M | Review with tech leads |
| 3.9 | Finalize API Documentation | OpenAPI spec, API reference, breaking changes documented | Generate from code, publish to docs site | Dev Team | M (if API changes) | N/A |
| 3.10 | Finalize Upgrade Guide | Upgrade instructions tested and validated | Manual upgrade test from N-1 version | QA Team + Tech Writer | M | N/A |
| 3.11 | Cross-Component Compatibility Validation | Compatibility matrix tested, all component version combinations validated | Integration testing with validated component versions | QA Team | M | Report compatibility issues immediately |
| 3.12 | Cut Additional RCs (if needed) | RC.2, RC.3 cut after bug fixes, each RC triggers full regression test | Tag new RC per Section 2.3 after merging fixes | Release Owner | M (if bugs fixed) | [Template E.5](#appendix-d-slack-communication-templates) |

**Decision Point:** When stabilization is complete and all Major+ bugs resolved → proceed to Code Freeze

---

### Phase 4: Code Freeze (Week 3, Late - 1-2 days before GA)

**Timing:** 1-2 days before GA target
**Goal:** Final lockdown, only critical/blocker fixes allowed, final validation

| # | Check Item | Criteria | Approach | Owner | M/O | Comm |
|---|-----------|----------|----------|-------|-----|------|
| 4.1 | **Code Freeze Announcement** | Team aware that only Blocker/Critical fixes allowed with Release Owner approval | Post Code Freeze announcement to Slack | Release Owner | M | [Template E.6](#appendix-d-slack-communication-templates) |
| 4.2 | Verify Bug Severity Gates (Section 3.2) | No open Blocker/Critical/Major bugs (Normal+ for MVP) | Review Jira/GitHub issues filter by severity and release version | Release Owner | M | N/A |
| 4.3 | Execute Final E2E Test Suite | All E2E tests passing on latest RC | Run full `hyperfleet-e2e` suite | QA Team | M | Report results in Slack |
| 4.4 | Execute Final Integration Tests | All integration tests passing | Run Prow integration test jobs | QA Team | M | Check Prow dashboard |
| 4.5 | Verify Container Image Vulnerability Scan | No CRITICAL/HIGH CVEs in final RC images | Run vulnerability scanner (e.g., Trivy, Clair) on RC images | Security Team | M | Block release if CRITICAL/HIGH found |
| 4.6 | Validate Helm Charts | Helm charts package without errors and deploy successfully | Test Helm install in clean cluster | QA Team | M | Document packaging issues |
| 4.7 | Verify All Release Artifacts | Container images, Helm charts, git tags, checksums available | Check registry, chart repo, GitHub tags per Section 6.1 | Release Owner | M | N/A |
| 4.8 | Documentation Final Review | All docs (release notes, upgrade guide, API docs) reviewed and published | Final review by Tech Writer and Tech Leads | Tech Writer + Tech Leads | M | Approve docs PR |
| 4.9 | Stakeholder Sign-Off | Pillar teams and stakeholders approve release | Email/Slack confirmation from key stakeholders | Release Owner | M | [Template E.7](#appendix-d-slack-communication-templates) |

**Decision Point:** All gates passed → proceed to GA Release

---

### Phase 5: GA Release Day

**Timing:** Release day (end of Week 3 per release calendar)
**Goal:** Tag GA release, publish artifacts, announce to users

| # | Check Item | Criteria | Approach | Owner | M/O | Comm |
|---|-----------|----------|----------|-------|-----|------|
| 5.1 | Final Go/No-Go Decision | All Phase 4 checks complete, no blockers | Final sync meeting with tech leads | Release Owner + Tech Leads | M | [Template E.8](#appendix-d-slack-communication-templates) |
| 5.2 | Tag GA Release (Component Tags) | Component version tags created (e.g., `v1.5.0`, `v1.4.2`, `v2.0.0`) per Section 2.5 | Execute git tag commands for each component | Release Owner | M | N/A |
| 5.3 | Tag GA Release (HyperFleet Release Tag) | HyperFleet Release tag created in `releases` repo (e.g., `release-1.5`) | Execute git tag in `openshift-hyperfleet/releases` repo | Release Owner | M | N/A |
| 5.4 | Verify GA Container Images Built | GA images built and pushed to registry with final version tags | Check registry: `registry.ci.openshift.org/hyperfleet/*:vX.Y.Z` | Release Owner | M | Alert CI team if build fails |
| 5.5 | Create GitHub Releases | GitHub Releases created from tags with release notes and compatibility matrix | Use GitHub UI or `gh` CLI per Section 6.1 | Release Owner | M | N/A |
| 5.6 | Publish Helm Charts | Helm charts published to chart repository (if applicable) | Package and push charts to chart repo | Release Owner | M (if chart repo exists) | N/A |
| 5.7 | Publish Release Notes | Release notes published to docs site and GitHub Releases | Merge docs PR, publish to docs.hyperfleet.io | Tech Writer | M | N/A |
| 5.8 | Update Compatibility Matrix | Compatibility matrix updated in docs and `releases` repo | Update compatibility table per Section 6.2.1 | Tech Writer | M | N/A |
| 5.9 | Release Announcement (Internal) | Internal teams notified of GA release | Post to #hyperfleet-releases, #hyperfleet-general | Release Owner | M | [Template E.9](#appendix-d-slack-communication-templates) |
| 5.10 | Release Announcement (External) | Users/customers notified of GA release (if applicable) | Email announcement, blog post, social media | Product Marketing | O | Coordinate with PM team |
| 5.11 | Update Release Tracking Issue | Tracking issue updated to "Released", close issue | Update GitHub issue, add final notes | Release Owner | M | N/A |

**Communication:** Celebrate with team! Post GA announcement with release notes link and highlights

---

### Phase 6: Post-Release (Week 4+)

**Timing:** After GA release through next sprint
**Goal:** Monitor release health, gather feedback, improve process

| # | Check Item | Criteria | Approach | Owner | M/O | Comm |
|---|-----------|----------|----------|-------|-----|------|
| 6.1 | Monitor Release Health (First 24-48 hours) | No critical issues reported, metrics stable | Monitor dashboards, Slack channels, support tickets | Release Owner + On-call Team | M | Escalate issues immediately |
| 6.2 | Collect Release Metrics | Release metrics captured per Section 8.3 | Extract data: code freeze duration, on-time delivery, bug escape rate | Release Owner | M | Document in retrospective notes |
| 6.3 | Schedule Release Retrospective | Retrospective scheduled within 1 week of GA | Create calendar invite, prepare retrospective agenda | Release Owner | M | Invite all contributors |
| 6.4 | Conduct Release Retrospective | Team feedback gathered, process improvements identified | Run retrospective meeting using standard format | Release Owner | M | [Template E.10](#appendix-d-slack-communication-templates) |
| 6.5 | Document Lessons Learned | Retrospective action items documented and assigned | Update release process doc or create improvement tickets | Release Owner | M | Share with team |
| 6.6 | Update Release Calendar | Next release dates published (if not already scheduled) | Update shared release calendar | Release Owner | M | Announce next release timeline |
| 6.7 | Handoff to Next Release Owner | Next Release Owner identified and briefed | Transition meeting, share lessons learned | Current + Next Release Owner | M | N/A |
| 6.8 | Enable Nightly Release Branch Testing | Prow nightly jobs running on release branch per Section 3.0 | Verify nightly jobs configured and running | CI Team | M | Monitor for regressions |

---

### Phase 7: Ongoing Release Branch Maintenance (Post-GA)

**Timing:** Ongoing for 6 months (support window per Section 2.4)
**Goal:** Maintain release branch with patch releases as needed

| # | Check Item | Criteria | Approach | Owner | M/O | Comm |
|---|-----------|----------|----------|-------|-----|------|
| 7.1 | Monitor Release Branch Health | Nightly E2E tests passing on release branch | Review Prow nightly test results | Release Owner (rotating) | M | Alert team if failures |
| 7.2 | Triage Patch Release Requests | Bugs evaluated for backport per Section 2.4 decision tree | Review bug severity, release age, backport criteria | Component Tech Leads | M | Use Section 4 process |
| 7.3 | Execute Patch Releases (Z-Stream) | Patch releases (e.g., v1.5.1) cut as needed per Section 2.1 (no RC required) | Follow hotfix workflow per Section 4.4 | Release Owner (rotating) | M | [Template E.11](#appendix-d-slack-communication-templates) |
| 7.4 | Update Support Lifecycle Status | Release phase tracked (Full Support → Security Maintenance → EOL) | Update release tracking doc with lifecycle phase | Release Owner | M | Notify users 1 month before EOL |
| 7.5 | Disable Nightly Jobs at EOL | Prow nightly jobs disabled when release reaches 6 months EOL | Disable Prow jobs per Section 3.0 | CI Team | M | N/A |

---

## 10. Appendices

### Appendix A: References and Sources

**Kubernetes Release Process:**
- [Kubernetes Release Cycle](https://kubernetes.io/releases/release/)
- [Kubernetes Release Cadence](https://goteleport.com/blog/kubernetes-release-cycle/)
- [Patch Releases | Kubernetes](https://kubernetes.io/releases/patch-releases/)
- [Kubernetes Branch](https://github.com/kubernetes/kubernetes/branches)

**Konflux CI/CD:**
- [Why Konflux?](https://konflux-ci.dev/docs/)
- [How we use software provenance at Red Hat](https://developers.redhat.com/articles/2025/05/15/how-we-use-software-provenance-red-hat)
- [Konflux Release Data Flow](https://konflux.pages.redhat.com/docs/users/releasing/preparing-for-release.html)

**Release Artifacts:**
- [OCI Artifacts Explained: Beyond Container Images](https://oneuptime.com/blog/post/2025-12-08-oci-artifacts-explained/view)
- [Manage Helm charts | Artifact Registry](https://docs.cloud.google.com/artifact-registry/docs/helm/manage-charts)

**Bug Handling and Code Freeze:**
- [Mastering the Code Freeze Process](https://ones.com/blog/mastering-code-freeze-process-software-stability/)
- [Code Freezes and Feature Flags](https://devcycle.com/blog/code-freezes-and-feature-flags)

**Cloud Readiness:**
- [The Ultimate Cloud Readiness Checklist for 2026](https://www.pulsion.co.uk/blog/cloud-readiness-checklist/)
- [Production Readiness Checklist](https://gruntwork.io/devops-checklist/)

### Appendix B: Glossary

- **Feature Freeze:** Deadline after which no new features accepted for current release
- **Code Freeze:** Period when only critical bug fixes are allowed into release branch
- **GA (General Availability):** Official release available to all users
- **RC (Release Candidate):** Pre-release version for final testing
- **SLSA:** Supply-chain Levels for Software Artifacts - security framework
- **SBOM:** Software Bill of Materials - list of all components in software
- **Cherry-Pick:** Applying specific commits from one branch to another
- **Hotfix:** Urgent fix applied to released version outside normal release cycle
- **LTS:** Long-Term Support - release with extended maintenance period
- **N-1 Compatibility:** Supporting one version back (e.g., v1.5 compatible with v1.4)

### Appendix C: Template - Release Tracking Issue

```markdown
# HyperFleet Release 1.5 Tracking Issue

## Timeline (3-week sprint cycle)
- Sprint Start: YYYY-MM-DD
- Feature Freeze: YYYY-MM-DD
- Code Freeze: YYYY-MM-DD
- GA Target: YYYY-MM-DD

**Note:** Dates may shift based on stabilization phase needs as automation is being built.

## Release Owner
@username

## Component Versions for Release 1.5

| Component | Target Version | Status | Notes |
|-----------|---------------|--------|-------|
| API Service | v1.5.0 | 🟡 In Progress | GitOps integration, OAuth 2.1 |
| Sentinel | v1.4.2 | 🟢 Ready | Bug fixes only |
| Adapter Framework | v2.0.0 | 🟡 In Progress | Breaking: Plugin API v2 |

## Release Criteria Status
- [ ] All planned features complete
- [ ] E2E tests passing with component combination
- [ ] No Blocker/Critical/Major bugs in any component
- [ ] Cross-component compatibility validated
- [ ] Documentation complete (including compatibility matrix)
- [ ] Pillar team sign-off

## Release Candidates
- [ ] v1.5.0-rc.1 (April 15, at Feature Freeze)
- [ ] v1.5.0-rc.2 (April 16-17, if needed)
- [ ] v1.5.0-rc.3 (April 18, if needed)

## Blockers
- None currently

## Compatibility Validation
- [ ] API Service v1.5.0 + Adapter Framework v2.0.0 integration tested
- [ ] Sentinel v1.4.2 compatibility with all components verified
- [ ] Backward compatibility with Release 1.4 validated
- [ ] Breaking changes documented with migration guides

## Communication
- [ ] Release notes drafted (including compatibility matrix)
- [ ] Breaking changes highlighted
- [ ] Stakeholders notified (T-1 week)
- [ ] Release announcement prepared

## Post-Release
- [ ] Retrospective scheduled
- [ ] Metrics collected
- [ ] Stabilization phase variance documented
- [ ] Component version tracking updated
```

### Appendix D: Slack Communication Templates

This appendix provides Slack message templates for Release Owner communication throughout the release cycle. Copy and customize these templates as needed.

---

#### Template E.1: Release Kickoff Announcement (Phase 1)

**Channel:** `#hyperfleet-releases`
**When:** Sprint start (Week 1)

```
:rocket: **HyperFleet Release 1.5 - Release Kickoff**

Hello team! I'm the Release Owner for **HyperFleet Release 1.5**.

**Timeline:**
• Sprint Start: [YYYY-MM-DD]
• Feature Freeze: [YYYY-MM-DD] (Start of Week 3)
• Code Freeze: [YYYY-MM-DD] (End of Week 3)
• GA Target: [YYYY-MM-DD]

**Component Versions:**
• API Service: v1.5.0 (new GitOps integration)
• Sentinel: v1.4.2 (bug fixes)
• Adapter Framework: v2.0.0 (breaking: Plugin API v2)

**Release Tracking Issue:** [Link to GitHub issue]

**Action Items:**
• All features for this release must be merged to `main` by Feature Freeze date
• Documentation drafts (release notes, API docs, upgrade guides) should be in progress
• Please flag any potential blockers ASAP

Questions? Tag me or comment on the tracking issue.
```

---

#### Template E.2: Feature Freeze Go/No-Go Decision (Phase 2)

**Channel:** `#hyperfleet-releases`
**When:** End of Week 2 (before Feature Freeze)

```
:traffic_light: **HyperFleet Release 1.5 - Feature Freeze Go/No-Go Decision**

Team, we're approaching Feature Freeze for Release 1.5 (scheduled for [DATE]). Here's our readiness status:

**Entry Criteria Status:**
✅ All planned features code-complete
✅ CI/CD pipeline green on main
✅ No CRITICAL/HIGH CVEs
⚠️ Documentation drafts in progress (finalizing this week)
✅ Feature toggles in place for experimental features

**Component Readiness:**
• API Service (v1.5.0): ✅ Ready
• Sentinel (v1.4.2): ✅ Ready
• Adapter Framework (v2.0.0): ✅ Ready

**Decision: GO for Feature Freeze on [DATE]**

**Next Steps:**
• Release branches will be created on [DATE]
• RC.1 will be tagged on [DATE]
• `main` branch remains open for v1.6 development after Feature Freeze

Any concerns or blockers? Please raise them NOW.
```

---

#### Template E.3: Feature Freeze Announcement (Phase 2)

**Channel:** `#hyperfleet-releases`, `#hyperfleet-dev`
**When:** Feature Freeze day (Start of Week 3)

```
:snowflake: **Feature Freeze - HyperFleet Release 1.5**

Feature Freeze is now in effect for Release 1.5!

**Release Branches Created:**
• `openshift-hyperfleet/api-service` → release-1.5
• `openshift-hyperfleet/sentinel` → release-1.4
• `openshift-hyperfleet/adapter-framework` → release-2.0
• `openshift-hyperfleet/hyperfleet-e2e` → release-1.5
• `openshift-hyperfleet/hyperfleet-release` → release-1.5

**Release Candidate:**
• RC.1 tagged: API Service v1.5.0-rc.1, Sentinel v1.4.2-rc.1, Adapter Framework v2.0.0-rc.1
• Container images: registry.ci.openshift.org/hyperfleet/*:vX.Y.Z-rc.1

**Branch Policy (Starting Now):**
• `main` branch: **OPEN** for v1.6 development - business as usual
• `release-X.Y` branches: **RESTRICTED** - only bug fixes allowed (see rules below)

**How to get bug fixes into Release 1.5:**
1. Create PR to fix bug in `main` branch → Merge
2. Cherry-pick the fix to release branch via new PR → I will review and approve
3. All release branch PRs require Release Owner approval + Prow green

**Stabilization Phase:**
• E2E testing in progress
• Bug triage meetings: Daily at [TIME] ([Zoom link])
• Target Code Freeze: [DATE]
• Target GA: [DATE]

Questions? Tag me!
```

---

#### Template E.4: Bug Triage Announcement (Phase 3)

**Channel:** `#hyperfleet-releases`
**When:** Daily during Stabilization Phase

```
:bug: **Daily Bug Triage - HyperFleet Release 1.5 - [DATE]**

**Current Bug Status:**
• Blocker: 0
• Critical: 1 (API Service #234 - auth token issue) - Fix in progress by @dev1
• Major: 2 (Sentinel #256 - metrics lag, Adapter #298 - plugin load error)
• Normal: 5 (tracked for v1.5.1)

**Decisions from Today's Triage:**
• #234 (Critical): Fix merged to main, cherry-picking to release-1.5 today
• #256 (Major): Downgraded to Normal - workaround documented, will fix in v1.5.1
• #298 (Major): Fix targeted for EOD today, will cut RC.2 tomorrow if fixed

**Action Required:**
• @dev1: Complete cherry-pick PR for #234 by EOD
• @dev2: Complete fix for #298 by EOD, add test coverage

**Next Triage:** Tomorrow [TIME] ([Zoom link])

See tracking issue for full bug list: [Link]
```

---

#### Template E.5: New Release Candidate Announcement (Phase 3)

**Channel:** `#hyperfleet-releases`, `#hyperfleet-dev`
**When:** After cutting RC.2, RC.3, etc.

```
:package: **Release Candidate RC.2 - HyperFleet Release 1.5**

RC.2 has been tagged with bug fixes from Stabilization Phase.

**What's in RC.2:**
• Fixed: API Service #234 - Authentication token refresh issue (Critical)
• Fixed: Adapter Framework #298 - Plugin loading error (Major)
• Regression test suite passing ✅

**Tags:**
• API Service: v1.5.0-rc.2
• Sentinel: v1.4.2-rc.1 (no changes from RC.1)
• Adapter Framework: v2.0.0-rc.2

**Container Images:**
• registry.ci.openshift.org/hyperfleet/api-service:v1.5.0-rc.2
• registry.ci.openshift.org/hyperfleet/sentinel:v1.4.2-rc.1
• registry.ci.openshift.org/hyperfleet/adapter-framework:v2.0.0-rc.2

**Testing in Progress:**
• Full E2E test suite running now (ETA: 2 hours)
• Backward compatibility testing (N-1 upgrade)

**Code Freeze:** Target [DATE] if testing passes
```

---

#### Template E.6: Code Freeze Announcement (Phase 4)

**Channel:** `#hyperfleet-releases`, `#hyperfleet-dev`
**When:** Code Freeze day (1-2 days before GA)

```
:ice_cube: **Code Freeze - HyperFleet Release 1.5**

Code Freeze is now in effect. We're in the final stretch!

**Branch Policy (Effective Immediately):**
• `release-X.Y` branches: **LOCKED** - Only Blocker/Critical fixes with Release Owner approval
• All PRs to release branches must include:
  - Bug severity: Blocker or Critical
  - Justification why it cannot wait for patch release
  - Risk assessment
  - Test coverage

**Current Status:**
• Latest RC: RC.2 (or RC.3 if applicable)
• E2E tests: ✅ Passing
• Integration tests: ✅ Passing
• Vulnerability scan: ✅ No CRITICAL/HIGH CVEs
• Documentation: ✅ Finalized
• Open bugs: 0 Blocker, 0 Critical, 0 Major

**GA Target:** [DATE] (2 days from now)

**Final Validation:**
• Stakeholder sign-off in progress
• Final artifact verification today
• Go/No-Go decision: [DATE TIME]

We're almost there! :rocket:
```

---

#### Template E.7: Stakeholder Sign-Off Request (Phase 4)

**Channel:** Direct message or `#hyperfleet-stakeholders`
**When:** 1 day before GA

```
:white_check_mark: **HyperFleet Release 1.5 - Stakeholder Sign-Off Request**

Hello! We're ready for GA release and need your final approval.

**Release Summary:**
• HyperFleet Release 1.5
• Component versions: API Service v1.5.0, Sentinel v1.4.2, Adapter Framework v2.0.0
• GA Target: [DATE]

**Quality Gates:**
✅ All E2E tests passing
✅ All integration tests passing
✅ No CRITICAL/HIGH CVEs
✅ Backward compatibility validated (N-1 upgrade tested)
✅ Documentation complete (release notes, upgrade guide, API docs)
✅ No open Blocker/Critical/Major bugs

**Release Artifacts Ready:**
• Container images published to registry
• Helm charts packaged and tested
• Release notes: [Link to preview]
• Upgrade guide: [Link to preview]

**Request:** Please review and provide sign-off by [DATE TIME].
Reply with ✅ to approve or raise concerns.

Questions? Let me know!
```

---

#### Template E.8: GA Release Go/No-Go Decision (Phase 5)

**Channel:** `#hyperfleet-releases`
**When:** GA Release Day (morning)

```
:rocket: **HyperFleet Release 1.5 - Final Go/No-Go Decision**

Team, it's GA day! Here's our final readiness check:

**All Gates Passed:**
✅ All testing complete (E2E, integration, backward compatibility)
✅ Vulnerability scan clean (no CRITICAL/HIGH CVEs)
✅ Documentation finalized and published
✅ Stakeholder sign-off received
✅ Release artifacts verified (images, Helm charts, tags)
✅ No open Blocker/Critical/Major bugs

**Decision: GO for GA Release**

**GA Release Timeline Today:**
• 10:00 AM - Tag GA release (component tags + HyperFleet Release tag)
• 10:30 AM - Verify GA images built
• 11:00 AM - Create GitHub Releases
• 11:30 AM - Publish release notes to docs site
• 12:00 PM - **GA Announcement** :tada:

I'll post updates as we progress. Stand by!
```

---

#### Template E.9: GA Release Announcement (Phase 5)

**Channel:** `#hyperfleet-releases`, `#hyperfleet-general`, `#hyperfleet-users`
**When:** GA Release Day (after all artifacts published)

```
:tada: **HyperFleet Release 1.5 is now Generally Available!** :tada:

We're excited to announce HyperFleet Release 1.5 is officially released!

**What's New:**
• **API Service v1.5.0:** GitOps integration for ROSA deployments, OAuth 2.1 support
• **Adapter Framework v2.0.0:** New Plugin API v2 with enhanced extensibility
• **Sentinel v1.4.2:** Memory leak fix, performance improvements

**Release Artifacts:**
• Container Images: registry.ci.openshift.org/hyperfleet/*:vX.Y.Z
• Release Notes: [Link to docs]
• Upgrade Guide: [Link to docs]
• GitHub Release: [Link to GitHub]

**Compatibility:**
• Kubernetes: 1.26 - 1.30
• Helm: 3.14+
• Backward compatible with Release 1.4 (N-1 upgrade tested)

**Breaking Changes:**
• Adapter Framework: Plugin API v2 (see migration guide: [Link])
• API Service: Removed deprecated `/v1/legacy-auth` endpoint

**Installation:**
```bash
# Helm installation example
helm repo update
helm install hyperfleet hyperfleet/hyperfleet --version 1.5.0
```

**Upgrade from Release 1.4:**
See upgrade guide: [Link]

**Known Issues:**
See release notes for known issues and workarounds: [Link]

**Thank you** to everyone who contributed to this release! :clap:

**Feedback:** Please report issues in [GitHub Issues] or #hyperfleet-support

**Next Release:** HyperFleet Release 1.6 scheduled for [DATE]
```

---

#### Template E.10: Release Retrospective Invitation (Phase 6)

**Channel:** `#hyperfleet-releases`
**When:** 1-3 days after GA

```
:hourglass: **HyperFleet Release 1.5 - Retrospective Meeting**

Great job on Release 1.5, team! Let's gather feedback and identify improvements.

**Retrospective Meeting:**
• **Date:** [DATE]
• **Time:** [TIME]
• **Duration:** 60 minutes
• **Zoom:** [Link]
• **Agenda Doc:** [Link to shared doc]

**Please come prepared to discuss:**
• What went well? (Keep doing)
• What didn't go well? (Stop doing)
• What can we improve? (Start doing)
• Specific pain points during this release cycle

**Pre-Retrospective Input:**
If you can't attend, please add your feedback to the agenda doc before the meeting.

**Release Metrics to Review:**
• Code freeze duration: X days (target: < 1 week)
• On-time delivery: [Yes/No]
• Bug escape rate: X bugs post-GA
• Component patch releases needed: X (target: < 2)

See you there!
```

---

#### Template E.11: Patch Release Announcement (Phase 7)

**Channel:** `#hyperfleet-releases`, `#hyperfleet-general`
**When:** Patch release (v1.5.1, v1.5.2, etc.)

```
:package: **HyperFleet Release 1.5.1 (Patch Release) Available**

We've released **HyperFleet Release 1.5.1**, a patch release with critical bug fixes.

**What's Fixed:**
• **API Service v1.5.1:** Fixed critical bug in GitOps integration (#345)
• **Sentinel v1.4.2:** No changes
• **Adapter Framework v2.0.0:** No changes

**Release Type:** Patch release (bug fixes only, no new features)

**Upgrade:**
This is a **drop-in replacement** for Release 1.5.0. No migration required.

```bash
# Helm upgrade
helm upgrade hyperfleet hyperfleet/hyperfleet --version 1.5.1
```

**Container Images:**
• registry.ci.openshift.org/hyperfleet/api-service:v1.5.1
• registry.ci.openshift.org/hyperfleet/sentinel:v1.4.2 (unchanged)
• registry.ci.openshift.org/hyperfleet/adapter-framework:v2.0.0 (unchanged)

**Release Notes:** [Link to patch release notes]

**Compatibility:** Same as Release 1.5.0 (see compatibility matrix)

**Recommendation:** All users on Release 1.5.0 should upgrade to 1.5.1 to benefit from these fixes.
```

---

### Appendix E: Template - Ad-Hoc Release Request

```markdown
# Ad-Hoc Release Request: HyperFleet Release 1.5.1 (or Component-Specific Patch)

## Release Type
- [ ] **Full HyperFleet Release** (multiple components, validated combination)
- [ ] **Single Component Patch** (e.g., Sentinel v1.4.3 only, no new HyperFleet Release number)

## Requestor Information
- **Requested by:** @username
- **Request date:** YYYY-MM-DD
- **Urgency:** Critical / High / Medium
- **Target release date:** YYYY-MM-DD

## Justification
Why can't this wait for the next regular release (HyperFleet Release X.X on DATE)?

[Explain business justification, customer impact, or urgency]

## Scope
What will be included in this ad-hoc release?

### Component Version Changes

| Component | Current Version (Release 1.5) | New Version | Change Type | Reason |
|-----------|------------------------------|-------------|-------------|--------|
| API Service | v1.5.0 | v1.5.1 | PATCH | Critical security fix |
| Sentinel | v1.4.2 | v1.4.2 | No change | - |
| Adapter Framework | v2.0.0 | v2.0.0 | No change | - |

### Changes Included
- [ ] Feature/fix #1: Brief description
- [ ] Feature/fix #2: Brief description
- [ ] Bug fix #3: Brief description

### Changes Explicitly Excluded
List what is NOT included to keep scope tight:
- Other pending PRs
- Unrelated bug fixes
- Nice-to-have features

## Impact Assessment

### Components Affected
- [ ] API Service - [changes description]
- [ ] Sentinel - [changes description]
- [ ] Adapter Framework - [changes description]

### Risk Level
- [ ] Low - Minor change, well-tested, quick patch if needed
- [ ] Medium - Moderate change, some risk
- [ ] High - Significant change, complex fix-forward required

### Blast Radius
- Number of users/environments affected: [estimate]
- Customer impact if issue occurs: [description]

## Testing Plan

### Automated Testing (Mandatory)
- [ ] Unit tests added/updated
- [ ] Integration tests passing
- [ ] E2E tests for affected components passing
- [ ] CI pipeline green

### Manual Testing (Mandatory)
- [ ] Smoke test plan defined
- [ ] Critical user paths validated
- [ ] Regression testing for affected areas

### Testing Deferred
What testing will be deferred to next regular release?
- [ ] Full exploratory testing
- [ ] Performance regression testing
- [ ] Other: [specify]

## Recovery Plan
**Primary strategy: Roll-forward via patch release (MVP approach)**

### If Issues Discovered
- [ ] Hotfix patch release plan documented
- [ ] Fix timeline estimated (target: < 48 hours for critical)
- [ ] Workaround available for users (if applicable)

### Database Migration Considerations
- [ ] Schema changes included? [yes/no]

## Stakeholder Coordination

### Offering Team Notification
- [ ] Offering team notified (minimum 48 hours advance)
- [ ] Integration testing with GCP completed
- [ ] Deployment coordination confirmed

### Communication Plan
- [ ] Release notes drafted
- [ ] Stakeholders notified
- [ ] Customer communication prepared (if external)

## Release Owner Approval

**Decision:**
- [ ] **Approved** - Proceed with ad-hoc release
- [ ] **Rejected** - Defer to next regular release (DATE)
- [ ] **Needs More Info** - [specify what's needed]

**Approval by:** @Technical Lead and @Manager
**Date:** YYYY-MM-DD
**Conditions:** [any special conditions or requirements]

## Release Timeline

| Day | Date | Activities | Owner |
|-----|------|-----------|-------|
| 1 | YYYY-MM-DD | Development + unit tests | @dev-team |
| 2 | YYYY-MM-DD | Code review + CI tests | @reviewers |
| 3 | YYYY-MM-DD | E2E testing + RC build | @qa-team |
| 4 | YYYY-MM-DD | Manual testing + stakeholder review | @qa + @stakeholders |
| 5 | YYYY-MM-DD | GA release + deployment | @Technical Lead and @Manager |

## Post-Release Monitoring
- [ ] Metrics dashboard monitored for 24 hours
- [ ] No error rate increase
- [ ] No performance degradation
- [ ] Post-release review completed
```

---
