# Spike Report: E2E Testing Framework for Cluster Lifecyle Manager (CLM) System

---

**JIRA Story:** HYPERFLEET-403  
**Date:** Jan 6, 2026  
**Target System:** API Service + Sentinel + Multi-Cloud Adapter Tasks (GCP, AWS, Azure)

---

## Executive Summary

This spike report evaluates E2E testing frameworks for the CLM system—a distributed microservice architecture consisting of an OpenAPI-based API Service, a Sentinel orchestrator, and multiple cloud adapter tasks deployed across provider-specific Kubernetes clusters (GCP, AWS, Azure, etc.).

After evaluating Ginkgo, Godog, and Testify, the report presents **two viable options for team vote**: 
- **Option A (Ginkgo + Markdown)** offers a pure Go testing approach with separate documentation—prioritizing simpler stack, superior debugging, and faster development velocity. 
- **Option B (Hybrid Ginkgo + Godog)** uses executable Gherkin scenarios to guarantee documentation-code alignment—eliminating drift risk and enabling stakeholder collaboration. Both are production-ready; the choice depends on team priorities around development velocity vs. guaranteed documentation synchronization.

The proposed framework emphasizes a **user-centric testing philosophy**: E2E tests validate the system exclusively through the API from the user's perspective, not internal component version combinations. This drives a **branch-based version strategy** where repository branches map to CLM release versions (e.g., release-1.2.x), eliminating version tracking in test code. Backward compatibility is validated by running older branch tests against newer CLM deployments.

The architecture abstracts **multi-provider deployment differences** through a provider interface pattern. Each cloud provider operates a dedicated K8s cluster with consistent core services (API Service, Sentinel) but provider-specific adapter sets. The framework dynamically discovers adapters and handles provider-specific timeouts, dependencies, and resource provisioning.

---

## 1. Background & Problem Statement

### 1.1 System Architecture

The target system comprises Cluster Lifecyle Manager (CLM) components deployed on Kubernetes clusters:

```text
┌──────────────────────────────────────────────────────────────┐
│                    User / API Consumer                       │
└───────────────────────┬──────────────────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────────────────┐
│              Kubernetes Cluster (CLM Deployment)             │
│                                                              │
│  ┌─────────────────┐                                         │
│  │   API Service   │ (OpenAPI, User-facing, Golang)          │
│  └────────┬────────┘                                         │
│           │                                                  │
│           ▼                                                  │
│  ┌─────────────────┐                                         │
│  │    Sentinel     │ (Orchestrator/Controller, Golang)       │
│  └────────┬────────┘                                         │
│           │                                                  │
│           │                                                  │
│           │ Step 1: Foundation Adapter                       │
│           ▼                                                  │
│     ┌──────────────────┐                                     │
│     │  Landing Zone    │ ◄── Base Adapter (Required First)   │
│     │    Adapter       │     Create required resources       │
│     └────────┬─────────┘     (e.g., k8s namespace)           │
│              │                                               │
│              │ Step 2: Dependent Adapters (parallel)         │
│              ├───────────┬─────────────┬─────────────┐       │
│              ▼           ▼             ▼             ▼       │
│         ┌──────────┐ ┌──────────┐ ┌──────────┐               │
│         │Validation│ │   DNS    │ │  Pull    │               │
│         │ Adapter  │ │Placement │ │ Secret   │     ...       │
│         │          │ │ Adapter  │ │ Adapter  │               │
│         └────┬─────┘ └────┬─────┘ └────┬─────┘               │
│              │            │            │                     │
│              └────────────┴────────────┴────────────┘        │
│                           │                                  │
│                           │ Step 3: Orchestration Adapter    │
│                           ▼                                  │
│                    ┌──────────────┐                          │
│                    │  Hypershift  │                          │
│                    │   Adapter    │                          │
│                    └──────┬───────┘                          │
│                           │                                  │
└───────────────────────────┴──────────────────────────────────┘
                            │
                            ▼
                   ┌────────────────┐
                   │ Cloud Provider │
                   │  (GCP/AWS/     │
                   │   Azure/...)   │
                   └────────────────┘
```


**Note:**
- Each K8s cluster is dedicated to a single cloud provider, ensuring strong isolation and independent lifecycle management
- **Consistent Components:** API Service and Sentinel are deployed identically across all provider clusters
- **Provider-Specific Components:** Adapters may differ between provider clusters based on provider requirements

### 1.2 Testing Challenges

1. **User-Centric Scenarios:** Tests must validate end-to-end flows from the user's perspective (e.g., "Create Cluster → Poll Status → Validate Adapters → Verify Cluster Ready")
2. **Multi-Service Orchestration:** Each scenario involves coordinating behavior across multiple adapters (Landing Zone, Validation, DNS Placement, Pull Secret, and potentially other provider-specific adapters, plus Hypershift) deployed in Kubernetes
3. **Multi-Cluster Testing:** E2E framework must connect to different K8s clusters to validate CLM works across providers (one cluster per provider)
4. **Adapter Composition:** Multiple adapters per provider (Landing Zone, Validation, DNS Placement, Pull Secret, ..., Hypershift) require coordinated validation, with adapter sets potentially varying per provider
5. **Cloud Provider Variance:** Adapters exhibit provider-specific timing, quotas, and failure modes
6. **Version Skew:** Rolling deployments require compatibility validation across service version combinations
7. **Kubernetes-Native Operations:** Tests must interact with K8s APIs for service discovery, health checks, and log collection
8. **Asynchronous Operations:** Cluster provisioning involves eventual consistency, requiring intelligent polling

### 1.3 Key Design Principles

**User-Centric Testing Philosophy:**

E2E tests must validate the system from the **user's perspective**. Users interact exclusively through the **API** and have no visibility into internal component versions or architecture.

**Critical Implications:**

1. ✅ **Test API Behavior, Not Component Combinations**
    - Focus on **CLM release versions** (e.g., v1.2.0), not individual service versions
    - Internal component compatibility (Sentinel, Adapters) is a deployment concern
    - E2E tests validate end-to-end user workflows through the API

2. ✅ **Branch-Based Testing Strategy**
    - Use repository branches to test different CLM releases
    - Test code aligns with the API version it validates

3. ✅ **Adapter Awareness Without Version Tracking**
    - Dynamically discover deployed adapters (varies by provider)
    - Validate adapter functionality through API operations
    - Don't track individual adapter versions separately

---

## 2. Framework Evaluation

### 2.1 Framework Comparison

#### 2.1.1 **Ginkgo v2** (Go BDD Testing Framework)

**Strengths:**
- **Mature Go-Native BDD:** First-class BDD syntax (`Describe`, `Context`, `It`) with excellent Go integration
- **Parallel Execution:** Built-in support for parallel specs at multiple levels (suite, describe, it)
- **Flexible Organization:** Nested contexts, shared behaviors, and spec filtering
- **Rich Assertions:** Integrates with Gomega for expressive, failure-message-rich assertions
- **Polling & Async:** `Eventually()` and `Consistently()` for eventual consistency testing
- **CLI Tooling:** Robust CLI for running, filtering, and debugging tests

**Weaknesses:**
- **Not Plain English:** While BDD-style, still code-first (not Gherkin)
- **Learning Curve:** Developers unfamiliar with BDD may need onboarding

**Use Case Fit:** ★★★★★  
Excellent for Go-native teams requiring robust parallel execution and async handling.

#### 2.1.2 **Godog** (Cucumber/Gherkin for Go)

**Strengths:**
- **Gherkin Syntax:** Enables plain-English scenario definitions (Given/When/Then)
- **Documentation-as-Code:** Feature files serve as both documentation and test definitions
- **Business Alignment:** Non-technical stakeholders can read and validate scenarios
- **Step Reusability:** Step definitions promote DRY principles

**Weaknesses:**
- **Less Mature:** Smaller community compared to Ginkgo
- **Limited Parallel Support:** Parallel execution less sophisticated than Ginkgo
- **Verbosity:** Gherkin can become unwieldy for complex technical scenarios
- **Debugging:** Stack traces less intuitive than pure Go tests

**Use Case Fit:** ★★★★☆  
Ideal when documentation-code alignment is paramount, especially for compliance-driven environments.

#### 2.1.3 **Testify** (Assertion & Mocking Library)

**Strengths:**
- **Lightweight:** Minimal abstraction, feels like idiomatic Go
- **Suite Support:** `suite` package provides setup/teardown hooks
- **Familiar Syntax:** Assertions resemble JUnit/xUnit patterns

**Weaknesses:**
- **No BDD Structure:** Lacks hierarchical organization (no nested contexts)
- **No Parallel Primitives:** Must implement parallelization manually
- **Limited Async Support:** No built-in polling/eventual consistency helpers

**Use Case Fit:** ★★★☆☆  
Suitable for unit/integration tests but lacks E2E orchestration features.

### 2.2 Recommendation Options (For Team Vote)

Both approaches are viable for CLM E2E testing. The choice depends on team priorities and organizational context. **Please review both options and vote on your preference.**

---

#### **Option A: Ginkgo + Markdown Documentation**

**Approach:**  
Use **Ginkgo v2** for all test implementation with **separate Markdown documents** for scenario documentation. Tests reference doc sections via comments/tags.

**Key Benefits:**
1. ✅ **Simpler Stack:** Pure Go testing, no additional DSL (Gherkin) - single language for entire team
2. ✅ **Superior Debugging:** Stack traces map directly to Go code, no step mapping indirection
3. ✅ **Full Go Power:** No Gherkin constraints on test structure - use full language features
4. ✅ **Flexible Documentation:** Markdown supports diagrams, code blocks, tables, links, images
5. ✅ **Lower Learning Curve:** Most developers already know Markdown and Go
6. ✅ **Faster Development:** Write tests directly without defining step definitions
7. ✅ **Easier Maintenance:** Single codebase, no step definition sync overhead
8. ✅ **Native Performance:** No interpretation layer between tests and implementation
9. ✅ **Team Familiarity:** Go developers can be productive immediately

**Trade-offs:**
- ⚠️ Drift Risk: Docs and code can diverge without automated validation (mitigated by process and code review)
- ⚠️ Manual Sync: Requires discipline to keep docs updated (can be enforced via PR templates and checklists)

**Best For:**
- Engineering-focused teams comfortable with Go
- Projects prioritizing development velocity and debugging efficiency
- Teams wanting maximum flexibility in test implementation
- Organizations with strong code review practices
- Technical audiences who value code readability over natural language

**Implementation Example:**  
```markdown
<!-- scenarios/cluster_lifecycle.md -->
## Scenario: Create Cluster with All Adapters
**Test ID:** E2E-CLM-001  
**Priority:** Critical

### Prerequisites
- CLM cluster (gcp-prod) is accessible via kubeconfig
- All adapters deployed and healthy

### Steps
1. Connect to GCP CLM cluster via K8s API
2. Verify all 5 adapters are in Running state
3. Submit CREATE_CLUSTER request via API service
4. Poll cluster status every 30 seconds
5. Validate cluster reaches READY within 15 minutes
6. Verify all adapters report SUCCESS status

### Expected Results
- API returns 202 Accepted
- Cluster ID is valid UUID
- All adapters complete successfully
- Cluster state is READY
```

```go
// tests/cluster_lifecycle_test.go
// Implements: scenarios/cluster_lifecycle.md
var _ = Describe("Cluster Lifecycle", Label("E2E-CLM-001", "multi-provider", "critical"), func() {
    // Multi-provider test using DescribeTable
    DescribeTable("Create cluster successfully",
        func(providerName string, providerConfig map[string]string) {
            var (
                ctx       context.Context
                provider  providers.CloudProvider
                k8sClient *clients.K8sClient
                apiClient *clients.APIClient
                clusterID string
            )

            ctx = context.Background()
            // Build complete config with common and provider-specific settings
            config := map[string]string{
                "kubeconfig":   fmt.Sprintf("/path/to/%s-kubeconfig", providerName),
                "cluster_name": fmt.Sprintf("%s-prod", providerName),
                "namespace":    "clm-system",
            }
            // Merge provider-specific config
            for k, v := range providerConfig {
                config[k] = v
            }

            var err error
            provider, err = providers.NewProvider(providerName, config)
            Expect(err).NotTo(HaveOccurred())

            // Submit cluster creation request
            By(fmt.Sprintf("Creating %s cluster via API", providerName))
            clusterSpec := &clients.ClusterSpec{
                Name:      fmt.Sprintf("test-%s-%d", providerName, time.Now().Unix()),
                Provider:  providerName,
                Region:    provider.Region(),
                NodeCount: 3,
            }

            cluster, err := apiClient.CreateCluster(ctx, clusterSpec)
            Expect(err).NotTo(HaveOccurred())
            Expect(cluster).NotTo(BeNil())

            // Verify all adapters completed successfully (with provider-specific timeouts)
            By("Verifying all adapters complete successfully")
            for _, adapterName := range requiredAdapters {
                timeout := provider.GetAdapterTimeout(adapterName)
                Eventually(func() string {
                    status, err := apiClient.GetAdapterStatus(ctx, clusterID, adapterName)
                    if err != nil {
                        GinkgoWriter.Printf("Error getting adapter status: %v\n", err)
                        return "ERROR"
                    }
                    return status.State
                }, timeout, 30*time.Second).Should(Equal("SUCCESS"),
                    fmt.Sprintf("Adapter %s should reach SUCCESS state", adapterName))
            }

            // Verify cluster is ready
            By("Polling for cluster READY state")
            clusterTimeout := provider.GetClusterTimeout(providerName)
            Eventually(func() string {
                status, err := apiClient.GetClusterStatus(ctx, clusterID)
                if err != nil {
                    GinkgoWriter.Printf("Error getting cluster status: %v\n", err)
                    return "ERROR"
                }
                GinkgoWriter.Printf("Cluster state: %s\n", status.State)
                return status.State
            }, clusterTimeout, 30*time.Second).Should(Equal("READY"),
                "Cluster should reach READY state within 15 minutes")

            // Cleanup
            _ = helpers.DeleteCluster(ctx, apiClient, clusterID, 5*time.Minute)
        },

        // Test entries with provider-specific configuration and labels
        Entry("on GCP", "gcp", map[string]string{
            "region":     "us-central1",
            "project_id": "test-project",
        }, Label("gcp", "provider:gcp")),

        Entry("on AWS", "aws", map[string]string{
            "region":     "us-east-1",
            "account_id": "123456789012",
        }, Label("aws", "provider:aws")),
    )
})
```

**For new feature:**
- **Create Documentation:** Add or update the Markdown document describing the scenario with prerequisites, steps, and expected results
- **Implement Tests:** Write corresponding Go test using Ginkgo via documentation comments/labels (e.g., `// Implements: docs/scenarios/feature.md`)
- **Code Review:** Ensure documentation and test implementation are reviewed together to maintain alignment

---

#### **Option B: Hybrid Ginkgo + Godog Architecture** 

**Approach:**  
Use **Ginkgo v2** as the test runner and orchestration layer, with **Godog (Gherkin)** for scenario definition in critical user flows.

**Key Benefits:**
1. ✅ **Executable Documentation:** Gherkin `.feature` files auto-validate against implementation
2. ✅ **Zero Drift Risk:** Scenarios cannot diverge from code (tests fail if out of sync)
3. ✅ **Stakeholder Collaboration:** Product, QA, Compliance can read and validate scenarios
4. ✅ **Living Documentation:** Scenarios serve as both specs and tests
5. ✅ **Industry Standard:** Gherkin is widely adopted (Cucumber, SpecFlow, Behave)

**Trade-offs:**
- ⚠️ Learning Curve: Team needs to learn Gherkin syntax and step definitions
- ⚠️ Debugging Complexity: Requires mapping from Gherkin steps to Go code
- ⚠️ Constraints: Test logic must fit Given/When/Then structure
- ⚠️ Additional Abstraction Layer: Step definitions add indirection
- ⚠️ Slower Development: Must define steps before writing tests
- ⚠️ Maintenance Overhead: Keep step definitions in sync with test code

**Best For:**
- Organizations with strong compliance/audit requirements requiring traceable documentation
- Cross-functional teams with significant non-developer involvement (Product, QA, Legal)
- Projects where business stakeholders need to validate test scenarios
- Teams with existing Gherkin/Cucumber experience

**Implementation Example:**  
```gherkin
# scenarios/cluster_lifecycle.feature
@cluster-lifecycle @multi-provider
Feature: Cluster Lifecycle Management
  As a platform user
  I want to create, monitor, and delete clusters via the API
  So that I can manage infrastructure programmatically across multiple cloud providers

  @critical @E2E-CLM-001 @multi-provider
  Scenario Outline: Create cluster successfully on multiple providers
    Given I am testing provider "<provider>"

    When I submit a "CREATE_CLUSTER" request with:
      | field      | value        |
      | name       | test-cluster |
      | provider   | <provider>   |
    Then I should receive a "202" response
    And the response should contain a valid cluster ID

    # Verify adapters complete (uses provider-specific timeouts)
    When I check the adapter statuses for the cluster
    Then all required adapters should reach "SUCCESS" state

    # Verify cluster ready
    When I poll the cluster status every "30 seconds"
    Then the cluster should reach "READY" state within "15 minutes"
```

```go
// Firstly, it should define each step
func (s *ClusterSteps) theClusterShouldReachStateWithin(expectedState, timeoutStr string) error {
    ctx := context.Background()
    apiClient := s.commonSteps.GetAPIClient()

    timeout := parseDuration(timeoutStr)
    pollInterval := 30 * time.Second

    fmt.Printf("Waiting for cluster to reach %s state (timeout: %v)...\n", expectedState, timeout)

    startTime := time.Now()
    for {
        status, err := apiClient.GetClusterStatus(ctx, s.CreatedClusterID)
        if err == nil && status.State == expectedState {
            fmt.Printf("✓ Cluster reached %s state\n", expectedState)
            return nil
        }

        if time.Since(startTime) > timeout {
            currentState := "UNKNOWN"
            if status != nil {
                currentState = status.State
            }
            return fmt.Errorf("cluster did not reach %s state within %v (current: %s)",
                expectedState, timeout, currentState)
        }

        time.Sleep(pollInterval)
    }
}

// Secondly, register the step, which mapping the content to the step
func (s *ClusterSteps) RegisterSteps(sc *godog.ScenarioContext) {
    // Also register common steps
    s.commonSteps.RegisterSteps(sc)
    
    // Cluster creation steps
    // Adapter status steps
    // Cluster status steps
    sc.Step(`^I poll the cluster status every "([^"]*)"$`, s.iPollTheClusterStatusEvery)
    sc.Step(`^the cluster should reach "([^"]*)" state within "([^"]*)"$`, s.theClusterShouldReachStateWithin)
    
    // Cleanup steps
    sc.Step(`^a cluster exists with ID "([^"]*)"$`, s.aClusterExistsWithID)
    sc.Step(`^I delete the cluster$`, s.iDeleteTheCluster)
    sc.Step(`^the cluster should be deleted within "([^"]*)"$`, s.theClusterShouldBeDeletedWithin)
}

// Thirdly, load all the scenarios, and dynamically creates Describe blocks for each feature
func registerFeatureTests() {
    scenariosDir := "../scenarios"

    // Discover all feature files
    features, err := discoverFeatures(scenariosDir)

    // Create a Describe block for each feature file with appropriate labels
    for _, feature := range features {
        // Capture feature in closure
        f := feature

        // Convert labels to Label() arguments
        ginkgoLabels := make([]interface{}, len(f.Labels))
        for i, label := range f.Labels {
            ginkgoLabels[i] = label
        }

        // Create Describe block with dynamic labels
        Describe(f.Name, Label(ginkgoLabels...), func() {
            var (
                ctx          context.Context
                clusterSteps *steps.ClusterSteps
                adapterSteps *steps.AdapterSteps
                commonSteps  *steps.CommonSteps
            )

            BeforeEach(func() {
                ctx = context.Background()
                // Initialize step definition handlers
                clusterSteps = steps.NewClusterSteps()
                adapterSteps = steps.NewAdapterSteps()
                commonSteps = steps.NewCommonSteps()
            })

            It("executes feature scenarios from: "+filepath.Base(f.Path), func() {
                GinkgoWriter.Printf("\nExecuting feature: %s\n", f.Path)
                GinkgoWriter.Printf("Labels: %v\n", f.Labels)

                // Create Godog test suite for this specific feature
                suite := godog.TestSuite{
                    ScenarioInitializer: func(sc *godog.ScenarioContext) {
                        // Register all step definitions
                        clusterSteps.RegisterSteps(sc)
                        adapterSteps.RegisterSteps(sc)
                        commonSteps.RegisterSteps(sc)

                        // Scenario hooks
                        sc.Before(func(ctx context.Context, scenario *godog.Scenario) (context.Context, error) {
                            GinkgoWriter.Printf("\n--- Scenario: %s ---\n", scenario.Name)
                            return ctx, nil
                        })

                        sc.After(func(ctx context.Context, scenario *godog.Scenario, err error) (context.Context, error) {
                            // Cleanup after each scenario
                            if clusterSteps.CreatedClusterID != "" {
                                GinkgoWriter.Printf("Cleaning up cluster: %s\n", clusterSteps.CreatedClusterID)
                                _ = clusterSteps.Cleanup(ctx)
                            }
                            return ctx, nil
                        })
                    },
                    Options: &godog.Options{
                        Format:   "pretty",
                        Paths:    []string{f.Path},
                        TestingT: GinkgoT(),
                    },
                }

                // Run Godog scenarios
                status := suite.Run()
                if status != 0 {
                    Fail("Feature scenarios failed")
                }
            })
        })
    }
}
```

**For new feature:**
- **Create Gherkin Scenarios:** Add new test cases in a `.feature` file using Given/When/Then syntax
- **Define Step Functions:** Implement Go functions for any new steps and register them in the scenario context
- **Validation:** Run tests to ensure Gherkin steps map correctly to step definitions (tests will fail if steps are undefined)
- **Maintenance:** When updating scenarios, verify step mappings remain valid and update step definitions as needed

---

## 3. Cloud Provider Abstraction Design

### 3.1 Cloud Provider Interface Pattern

Following the **Common Adapter Framework** blueprint, the E2E framework uses a provider interface to abstract cloud-specific operations while maintaining awareness of the Kubernetes deployment model.

#### 3.1.1 Core Architecture Principles

1. **One K8s Cluster Per Provider**: Each provider (GCP, AWS, Azure) has its own dedicated K8s cluster
2. **Consistent Core Services**: API Service and Sentinel are deployed identically across all provider clusters
3. **Provider-Specific Adapters**: Each cluster contains provider-specific adapter instances (no sharing between providers)
4. **Dynamic Discovery**: Test framework discovers services and adapters from K8s at runtime

#### 3.1.2 Provider Interface Design

```go
// CloudProvider abstracts cloud-specific operations for E2E testing
type CloudProvider interface {
    // Metadata
    Name() string                        // "gcp", "aws", "azure"
    Region() string                      // Default region for this provider

    // Kubernetes Environment
    GetClusterName() string              // K8s cluster name (e.g., "gcp-prod-cluster")
    GetKubeconfig() string               // Path to kubeconfig
    GetNamespace() string                // CLM deployment namespace

    // Adapter Management
    RequiredAdapters() []string          // All adapters for this provider
    GetAdapterTimeout(adapterType string) time.Duration
    ValidateAdapterDependencies(adapterName string) error

    // Service Discovery (from K8s)
    DiscoverAPIEndpoint(ctx context.Context, k8sClient *kubernetes.Clientset) (string, error)
    DiscoverAdapters(ctx context.Context, k8sClient *kubernetes.Clientset) (map[string]*AdapterInfo, error)

    // Test Resource Management
    ProvisionTestResources(ctx context.Context, opts ProvisionOpts) (*TestResources, error)
    CleanupTestResources(ctx context.Context, resources *TestResources) error
}
```

#### 3.1.3 Provider Implementation Example

```go
// GCP Provider
func (p *GCPProvider) RequiredAdapters() []string {
    return []string{
        "landing-zone",    // Foundation adapter (runs first)
        "validation",      // Validation checks
        "dns-placement",   // DNS configuration
        "pull-secret",     // Image pull secrets
        "hypershift",      // Orchestration adapter
    }
}

func (p *GCPProvider) GetAdapterTimeout(adapterType string) time.Duration {
    timeouts := map[string]time.Duration{
        "landing-zone":   8 * time.Minute,
        "validation":     5 * time.Minute,
        "dns-placement":  6 * time.Minute,
        "pull-secret":    3 * time.Minute,
        "hypershift":     15 * time.Minute,
    }
    return timeouts[adapterType]
}
```

---

## 4. Compatibility Testing Strategy

### 4.1 User-Centric Testing Philosophy

**Key Principle:** E2E tests validate end-to-end flows from the **user's perspective**. Users interact exclusively through the **API** and have no visibility into internal component versions (Sentinel, Adapters, etc.).

**Implication:** E2E testing should focus on **API version compatibility**, not internal component version combinations. Internal component compatibility is a **deployment concern**, not an E2E testing concern.

### 4.2 CLM Release-Based Testing Approach

#### 4.2.1 **Branch-Based Testing Model**

**Simple Principle:** Repository branch = CLM version. No version tracking in test code needed.

**Repository Branch Structure:**

```
e2e-tests/
├── release-1.0.x/        # Tests for CLM v1.0.x
├── release-1.1.x/        # Tests for CLM v1.1.x
├── release-1.2.x/        # Tests for CLM v1.2.x (current)
└── main/                 # Tests for next release (development)
```

**Testing Approach:**
- Each branch contains tests written for that specific CLM release
- No version information needed in test code
- Branch name implicitly defines which API version tests expect
- Tests are always compatible with the branch they're on

#### 4.2.2 **Backward Compatibility Testing**

**Key Insight:** Latest CLM components can run tests from previous version branches.

**Testing Workflow:**

```bash
# Test current release with its own tests
git checkout release-1.2
./e2e-runner --env=config/environments/gcp-staging.yaml
# Result: All release-1.2 tests pass on CLM v1.2.0

# Test backward compatibility: old tests on new CLM
git checkout release-1.1  # Previous version tests
./e2e-runner --env=config/environments/gcp-staging.yaml  # Same v1.2.0 environment
# Result: release-1.1 tests still pass, proving backward compatibility
```

---

## 5. Reliability & Resilience Patterns

### 5.1 The Flakiness Problem

E2E tests are inherently flaky due to:
- Network latency variance
- Asynchronous operations
- Resource contention
- External service dependencies

### 5.2 Anti-Flakiness Patterns

#### 5.2.1 **Poll-and-Wait Pattern**

Never use fixed `time.Sleep()`. Always poll with intelligent backoff:

```go
// Good: Polling with timeout
Eventually(func() string {
    status, err := apiClient.GetClusterStatus(clusterID)
    Expect(err).NotTo(HaveOccurred())
    return status.State
}, 10*time.Minute, 30*time.Second).Should(Equal("READY"))
```

**Gomega's `Eventually`:**
- Polls until condition is met or timeout
- Configurable polling interval
- Automatic retry on transient errors

#### 5.2.2 **Idempotent Cleanup**

Cleanup must be idempotent and never fail:

```go
AfterEach(func() {
    defer GinkgoRecover() // Don't let cleanup failures crash suite
    
    // Idempotent cleanup
    if testCluster != nil {
        _ = apiClient.DeleteCluster(testCluster.ID) // Ignore errors
        
        // Wait for deletion to complete (but don't block forever)
        Eventually(func() bool {
            _, err := apiClient.GetCluster(testCluster.ID)
            return err != nil && IsNotFoundError(err)
        }, 5*time.Minute, 10*time.Second).Should(BeTrue())
    }
    
    // Force cleanup if graceful deletion fails
    if testEnv != nil {
        ctx, cancel := context.WithTimeout(context.Background(), 2*time.Minute)
        defer cancel()
        _ = testEnv.ForceCleanup(ctx)
    }
})
```

---

## References
- [Ginkgo](https://github.com/onsi/ginkgo)
- [Gomega](https://github.com/onsi/gomega)
- [Cucumber for golang](https://github.com/cucumber/godog)
- [Gherkin](https://cucumber.io/docs/gherkin/)
- [Testify](https://github.com/stretchr/testify)
- [API testing tools](https://testguild.com/api-testing-tools/)
- [Bruno](https://github.com/usebruno/bruno/tree/main)
- [Markdown style test suite example of ty](https://github.com/astral-sh/ruff/blob/main/crates/ty_python_semantic/resources/mdtest/typed_dict.md)

---
