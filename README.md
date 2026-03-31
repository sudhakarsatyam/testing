# HULA Platform — Review & Architectural Remediation Strategy
---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [CRITICAL: Anti-Pattern Naming Convention](#2-critical-anti-pattern-naming-convention)
3. [Redundancy & Dead Code (DRY Violations)](#3-redundancy--dead-code-dry-violations)
4. [Over-Fragmentation & Low Readability](#4-over-fragmentation--low-readability)
5. [Tight Coupling & Untestability](#5-tight-coupling--untestability)
6. [Hardcoded Values & Environment Risk](#6-hardcoded-values--environment-risk)
7. [Naming Conventions & Documentation Gaps](#7-naming-conventions--documentation-gaps)
8. [Master Remediation Roadmap](#8-master-remediation-roadmap)

---

## 1. Executive Summary

This document provides a detailed analysis of the technical debt, structural flaws, and security risks identified across the HULA codebase. It categorizes issues into critical themes, provides specific code evidence from the repository, and outlines a phased remediation roadmap.

A total of **six critical concern areas** have been identified, ranging from security-impacting code duplication to architectural anti-patterns that undermine testability, maintainability, and operational stability.

| Area | Severity | Key Concern |
|------|----------|-------------|
| **Anti-Pattern Naming** | 🔴 CRITICAL | Services named after implementation language, not business function |
| **Redundancy & Dead Code** | 🟠 HIGH | Identical JWT logic replicated across 5+ controllers; 30+ pass-through wrappers |
| **Over-Fragmentation** | 🟠 HIGH | 90% identical methods with 45-line permission blocks hiding 1-line operations |
| **Tight Coupling** | 🟠 HIGH | 710-line God class with direct `new RestTemplate()`; zero unit test capability |
| **Hardcoded Values** | 🟡 MEDIUM | Infrastructure IPs and literals embedded in source; fragile across environments |
| **Naming & Documentation** | 🟡 MEDIUM | Mixed snake_case/camelCase; implementation-focused names; no Javadoc |

---

## 2. CRITICAL: Anti-Pattern Naming Convention

> **🔴 SEVERITY: CRITICAL**  
> **Affected Components:** `PythonController`, `PythonApiService`

### The Problem

The HULA codebase contains a fundamental architectural anti-pattern: **naming services and controllers after the implementation technology rather than their business function.**

`PythonController` and `PythonApiService` are named after the language of the downstream system they communicate with — not the domain capability they provide. This is a critical violation of clean architecture principles.

### Why This Is Wrong

A controller or service name should communicate **what business function it serves**, not **how it is implemented** or **what technology stack it calls**. The implementation language of a downstream dependency is an internal detail that should be invisible to the rest of the system.

```java
// ❌ CURRENT: Named after the downstream language — tells you NOTHING about what it does
@RestController
@RequestMapping("/api/python")
public class PythonController {
    
    private final PythonApiService pythonApiService;
    
    // What does this controller do? You have to read every method to find out.
    // If the downstream service is rewritten in Go tomorrow, this class name becomes a lie.
}
```

```java
// ❌ CURRENT: 710-line God class named after an implementation detail
@Service
public class PythonApiService {
    
    // This class handles THREE completely different domains:
    // 1. Report generation
    // 2. Workload management  
    // 3. Test execution
    
    // None of these have anything to do with "Python" — that's just the 
    // language the downstream service happens to be written in.
    
    private final RestTemplate restTemplate = new RestTemplate();
    
    public Object getScenarioReport(...) { /* report logic */ }
    public Object createWorkload(...) { /* workload logic */ }
    public Object executeTest(...) { /* execution logic */ }
    // ... 700+ more lines mixing all three domains
}
```

### Recommended Renaming

```java
// ✅ CORRECT: Named after business function — immediately clear what it does
@RestController
@RequestMapping("/api/reports")
public class ReportController {
    
    private final ReportClient reportClient;
    
    // Anyone reading this knows: this controller handles reports.
    // The downstream implementation is irrelevant.
}

// ✅ CORRECT: Named after business function
@RestController
@RequestMapping("/api/executions")
public class ExecutionController {
    
    private final ExecutionClient executionClient;
}
```

```java
// ✅ CORRECT: God class split into domain-specific clients
@Service
public class ReportClient {
    // Only report generation logic — ~200 lines instead of 710
    private final RestTemplate restTemplate;
    
    public ReportClient(RestTemplate restTemplate) {
        this.restTemplate = restTemplate; // injected, not new'd
    }
}

@Service
public class WorkloadClient {
    // Only workload management logic
}

@Service
public class ExecutionClient {
    // Only test execution logic
}
```

| Current Name | Recommended Name | Rationale |
|---|---|---|
| `PythonController` | `ReportController` / `ExecutionController` | Name after the business function, not the downstream language |
| `PythonApiService` | Split into: `ReportClient`, `WorkloadClient`, `ExecutionClient` | 710-line God class handles 3 domains — split and name each by responsibility |

### The Principle

> *Name every class, service, and controller after **what it does for the business**, never after how it is implemented, what language it calls, or what protocol it uses. If swapping the downstream technology would make the name incorrect, the name is wrong.*

This applies universally: a service that calls a gRPC endpoint should not be named `GrpcService`. A controller that proxies to a Node.js microservice should not be named `NodeController`. The name must reflect the **domain**, not the **plumbing**.

---

## 3. Redundancy & Dead Code (DRY Violations)

> **🟠 SEVERITY: HIGH**  
> **Persistent Areas:** `ProjectController`, `WebsiteController`, `UserController`, `ScenariosController`, `PythonController` *(to be renamed)*

### Findings

- Every controller manually extracts JWT tokens and defines an identical `getAction(String method)` utility — violating DRY across 5+ files
- Token parsing logic is replicated across approximately **50 endpoints**, creating multiple points of failure for security updates
- `PythonController` *(to be renamed)* contains **30+ methods** that act as simple pass-throughs to the downstream service, adding zero business logic

### Impact

- A security fix must be applied in 5+ places simultaneously — miss one and a vulnerability stays open
- 30+ wrapper methods increase maintenance surface with no added value
- New developers waste time understanding repeated boilerplate instead of business logic

### Code Evidence

**Issue 1: Identical `getAction()` utility replicated in 5+ controllers**

```java
// ❌ THIS EXACT METHOD EXISTS IN: ProjectController, WebsiteController, 
//    UserController, ScenariosController, PythonController
//    Any change must be made 5 times or behavior diverges.

private String getAction(String method) {
    return switch (method) {
        case "GET"    -> "view";
        case "POST"   -> "create";
        case "PUT"    -> "update";
        case "DELETE"  -> "delete";
        default       -> "";
    };
}
```

**Issue 2: Token parsing repeated in ~50 endpoints**

```java
// ❌ THIS BLOCK IS COPY-PASTED INTO NEARLY EVERY ENDPOINT
//    A single security bug here means patching 50 methods.

@GetMapping("/some-endpoint")
public ResponseEntity<Object> someEndpoint(
        @RequestHeader("Authorization") String token) {
    
    // --- START OF DUPLICATED BLOCK (appears ~50 times) ---
    String jwtToken = token.startsWith(SecurityConstants.BEARER_PREFIX) 
                      ? token.substring(SecurityConstants.BEARER_PREFIX_LENGTH) 
                      : token;
    
    String userId = jwtUtils.extractUserId(jwtToken);
    if (!accessService.hasAccess(jwtToken, "entity", getAction("GET"))) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body("Access denied");
    }
    // --- END OF DUPLICATED BLOCK ---
    
    // The actual business logic is just this ONE line:
    return ResponseEntity.ok(service.getData());
}
```

**Issue 3: 30+ pass-through wrappers adding zero value**

```java
// ❌ PythonController (to be renamed) — these methods do NOTHING except forward the call
//    Each one is 5-15 lines of boilerplate for a 1-line delegation.

@GetMapping("/reports/scenario/{id}")
public ResponseEntity<Object> getScenarioReport(@PathVariable String id, 
                                                 @RequestHeader("Authorization") String token) {
    // ... 10 lines of duplicated auth/permission checking ...
    return ResponseEntity.ok(pythonApiService.getScenarioReport(id, jwtToken));
}

@GetMapping("/reports/workload/{id}")
public ResponseEntity<Object> getWorkloadReport(@PathVariable String id, 
                                                 @RequestHeader("Authorization") String token) {
    // ... SAME 10 lines of duplicated auth/permission checking ...
    return ResponseEntity.ok(pythonApiService.getWorkloadReport(id, jwtToken));
}

// ... 28 MORE methods with the same pattern ...
```

### Recommended Remediation

```java
// ✅ FIX 1: Centralize security in a Spring HandlerInterceptor or SecurityUtils
@Component
public class SecurityUtils {
    
    public String extractToken(String authHeader) {
        return authHeader.startsWith(SecurityConstants.BEARER_PREFIX)
            ? authHeader.substring(SecurityConstants.BEARER_PREFIX_LENGTH)
            : authHeader;
    }
    
    public String mapHttpMethodToAction(String method) {
        return switch (method) {
            case "GET"    -> "view";
            case "POST"   -> "create";
            case "PUT"    -> "update";
            case "DELETE"  -> "delete";
            default       -> "";
        };
    }
    
    public void validateAccess(String token, String entity, String action) {
        if (!accessService.hasAccess(token, entity, action)) {
            throw new AccessDeniedException("Insufficient permissions");
        }
    }
}
```

```java
// ✅ FIX 2: Consolidate 30+ endpoints into parameterized routes
@RestController
@RequestMapping("/api/reports")
public class ReportController {

    @GetMapping("/{reportType}/{id}")
    public ResponseEntity<Object> getReport(
            @PathVariable String reportType,
            @PathVariable String id,
            @AuthenticatedUser UserContext user) {  // resolved by interceptor
        
        return ResponseEntity.ok(reportClient.getReport(reportType, id));
    }
    // ONE method replaces 30+ pass-throughs
}
```

---

## 4. Over-Fragmentation & Low Readability

> **🟠 SEVERITY: HIGH**  
> **Persistent Areas:** `WebsiteController`, `ScenariosController`

### Findings

- Overloaded methods contain **40–50 lines** of identical permission lookup and validation code, hiding the 1-line business operation
- Two methods in `WebsiteController` are **90% identical** — only the final service call differs
- The actual business logic is buried under layers of repetitive permission and user lookup noise

### Impact

- New developers forced to sift through repetitive noise to find the signal — significantly increases onboarding time
- Any change to the permission pattern requires updating dozens of methods simultaneously
- Code reviews become ineffective when reviewers cannot distinguish business logic from boilerplate

### Code Evidence

```java
// ❌ WebsiteController — TWO methods that are 90% IDENTICAL
//    The only difference is the final service call (paginated vs non-paginated)

@GetMapping("/v1/{project_id}")
public ResponseEntity<Object> getWebsitesByProjectIdV1(
        @PathVariable("project_id") String projectId,
        @RequestParam int page,
        @RequestParam int size,
        @RequestHeader("Authorization") String token) {
    
    // --- 45 LINES OF DUPLICATED PERMISSION & USER LOOKUP ---
    String jwtToken = token.startsWith(SecurityConstants.BEARER_PREFIX) 
        ? token.substring(SecurityConstants.BEARER_PREFIX_LENGTH) : token;
    String userId = jwtUtils.extractUserId(jwtToken);
    String userRole = jwtUtils.extractRole(jwtToken);
    
    if (!accessService.hasAccess(jwtToken, "websites", getAction("GET"))) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body("Access denied");
    }
    
    // Validate project access
    Optional<Project> project = projectRepository.findById(projectId);
    if (project.isEmpty()) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body("Project not found");
    }
    if (!project.get().getMembers().contains(userId) && !userRole.equals("ADMIN")) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body("Not a member");
    }
    // ... more validation ...
    // --- END OF 45-LINE DUPLICATED BLOCK ---
    
    // THE ACTUAL BUSINESS LOGIC — ONE LINE:
    return ResponseEntity.ok(websiteService.getWebsitesByProjectId(projectId, page, size));
}

@GetMapping("/{project_id}")
public ResponseEntity<Object> getWebsitesByProjectId(
        @PathVariable("project_id") String projectId,
        @RequestHeader("Authorization") String token) {
    
    // --- EXACT SAME 45 LINES COPY-PASTED HERE ---
    String jwtToken = token.startsWith(SecurityConstants.BEARER_PREFIX) 
        ? token.substring(SecurityConstants.BEARER_PREFIX_LENGTH) : token;
    String userId = jwtUtils.extractUserId(jwtToken);
    String userRole = jwtUtils.extractRole(jwtToken);
    // ... identical 45 lines ...
    // --- END OF DUPLICATED BLOCK ---
    
    // THE ONLY DIFFERENCE — no pagination:
    return ResponseEntity.ok(websiteService.getWebsitesByProjectId(projectId));
}
```

### Recommended Remediation

```java
// ✅ Extract the 45-line block into a single reusable method
private AccessContext validateProjectAccess(String token, String projectId) {
    String jwtToken = securityUtils.extractToken(token);
    String userId = jwtUtils.extractUserId(jwtToken);
    String userRole = jwtUtils.extractRole(jwtToken);
    
    securityUtils.validateAccess(jwtToken, "websites", "view");
    
    Project project = projectRepository.findById(projectId)
        .orElseThrow(() -> new ResourceNotFoundException("Project not found"));
    
    if (!project.getMembers().contains(userId) && !userRole.equals("ADMIN")) {
        throw new AccessDeniedException("Not a project member");
    }
    
    return new AccessContext(jwtToken, userId, userRole, project);
}

// ✅ Now each endpoint is clean — just mapping + business call
@GetMapping("/v1/{project_id}")
public ResponseEntity<Object> getWebsitesPageable(
        @PathVariable("project_id") String projectId,
        @RequestParam int page, @RequestParam int size,
        @RequestHeader("Authorization") String token) {
    
    validateProjectAccess(token, projectId);
    return ResponseEntity.ok(websiteService.getWebsitesByProjectId(projectId, page, size));
}

@GetMapping("/{project_id}")
public ResponseEntity<Object> getWebsites(
        @PathVariable("project_id") String projectId,
        @RequestHeader("Authorization") String token) {
    
    validateProjectAccess(token, projectId);
    return ResponseEntity.ok(websiteService.getWebsitesByProjectId(projectId));
}
```

---

## 5. Tight Coupling & Untestability

> **🟠 SEVERITY: HIGH**  
> **Persistent Areas:** `PythonApiService` *(to be renamed per Section 2)*

### Findings

- **God Class:** `PythonApiService` spans **710 lines** handling report generation, DB updates, and workload management in a single class
- **Direct instantiation** of `RestTemplate` (`new RestTemplate()`) prevents any form of mocking or unit testing
- **Manual JSON navigation** using `ObjectMapper` with raw `JsonNode` traversal — fragile and lacks type-safety
- **No separation of concerns:** infrastructure code mixed with business logic throughout

### Impact

- **Zero unit test capability** — any change requires full integration environment to verify
- **No regression safety net** — production defects discovered by users, not tests
- Single class failure cascades across reports, workloads, and execution simultaneously
- Refactoring becomes high-risk due to entangled responsibilities

### Code Evidence

```java
// ❌ TIGHT COUPLING: Direct instantiation makes unit testing IMPOSSIBLE
//    You cannot mock this — it will always make real HTTP calls

@Service
public class PythonApiService {  // to be renamed — see Section 2
    
    // This line alone prevents all unit testing of this 710-line class
    private final RestTemplate restTemplate = new RestTemplate();
    
    public Object getScenarioReport(String scenarioId, String token) {
        // Makes a real HTTP call — cannot be mocked in tests
        ResponseEntity<String> response = restTemplate.exchange(
            baseUrl + "/api/reports/scenario/" + scenarioId,
            HttpMethod.GET,
            new HttpEntity<>(createHeaders(token)),
            String.class
        );
        
        // ❌ FRAGILE: Manual JSON navigation — breaks if response structure changes
        //    No compile-time safety, no IDE autocomplete, no documentation
        ObjectMapper mapper = new ObjectMapper();
        JsonNode rootNode = mapper.readTree(response.getBody());
        List<Object> allTestcases = mapper.convertValue(
            rootNode, 
            new TypeReference<List<Object>>() {}
        );
        
        // ❌ SIDE EFFECT: DB update buried inside what looks like a read operation
        //    A method named "get" should NOT modify the database
        scenarioRepository.updateLastAccessTime(scenarioId, LocalDateTime.now());
        
        return allTestcases;
    }
    
    // ... 680+ more lines mixing reports, workloads, execution, and DB operations
}
```

### Recommended Remediation

```java
// ✅ FIX 1: Inject RestTemplate as a Spring Bean — now mockable in tests
@Configuration
public class RestClientConfig {
    
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .connectTimeout(Duration.ofSeconds(5))
            .readTimeout(Duration.ofSeconds(30))
            .build();
    }
}
```

```java
// ✅ FIX 2: Split God class into domain-specific clients with DTOs
@Service
public class ReportClient {
    
    private final RestTemplate restTemplate;  // injected — mockable
    
    public ReportClient(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }
    
    public ScenarioReportDTO getScenarioReport(String scenarioId, String token) {
        // ✅ Type-safe: Spring auto-deserializes into a proper DTO
        return restTemplate.exchange(
            baseUrl + "/api/reports/scenario/" + scenarioId,
            HttpMethod.GET,
            new HttpEntity<>(createHeaders(token)),
            ScenarioReportDTO.class     // ← DTO instead of raw String + manual parsing
        ).getBody();
    }
}
```

```java
// ✅ FIX 3: Define DTOs for type-safe API communication
public record ScenarioReportDTO(
    String scenarioId,
    String status,
    List<TestCaseDTO> testCases,
    LocalDateTime generatedAt
) {}

public record TestCaseDTO(
    String id,
    String name,
    String result,
    long durationMs
) {}
```

```java
// ✅ Now unit tests are possible:
@ExtendWith(MockitoExtension.class)
class ReportClientTest {
    
    @Mock
    private RestTemplate restTemplate;
    
    @InjectMocks
    private ReportClient reportClient;
    
    @Test
    void getScenarioReport_returnsReport() {
        // This test runs in milliseconds with no HTTP calls
        when(restTemplate.exchange(any(), eq(HttpMethod.GET), any(), eq(ScenarioReportDTO.class)))
            .thenReturn(ResponseEntity.ok(testReport));
        
        ScenarioReportDTO result = reportClient.getScenarioReport("sc-123", "token");
        
        assertNotNull(result);
        assertEquals("sc-123", result.scenarioId());
    }
}
```

---

## 6. Hardcoded Values & Environment Risk

> **🟡 SEVERITY: MEDIUM**  
> **Persistent Areas:** `application.yml`, `ScenariosController`

### Findings

- Infrastructure IPs (`192.168.0.147`) hardcoded directly in `application.yml`
- Literal strings (`"scenarios"`, `"projects"`) hardcoded in controller source code for access checks
- No environment-specific configuration strategy — same values used across Dev, Stage, and Prod

### Impact

- Application is fragile across environments — deployment to a new environment requires code changes
- IP changes require a rebuild and redeployment instead of a config update
- Hardcoded literals create hidden dependencies easy to miss during refactoring

### Code Evidence

```yaml
# ❌ application.yml — Hardcoded IP that will break in any other environment
spring:
  datasource:
    url: jdbc:postgresql://192.168.0.147:5432/postgresDB
    # What happens when this moves to AWS RDS? Or a different subnet?
    # A code change + rebuild + redeployment — for an infrastructure config.
```

```java
// ❌ ScenariosController — Hardcoded literal strings for access control
//    If someone renames the "scenarios" permission entity, this silently breaks

@GetMapping("/{id}")
public ResponseEntity<Object> getScenario(@PathVariable String id,
        @RequestHeader("Authorization") String token) {
    
    String jwtToken = /* ... duplicated extraction ... */;
    
    // ❌ "scenarios" is a magic string — no compile-time validation
    //    Typo "scnearios" would silently fail open or closed depending on ACL defaults
    if (!accessService.hasAccess(jwtToken, "scenarios", getAction("GET"))) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body("Access denied");
    }
    
    return ResponseEntity.ok(scenarioService.getScenario(id));
}
```

### Recommended Remediation

```yaml
# ✅ application.yml — Externalized with environment variable + sensible default
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:postgresDB}
    # Dev: uses localhost (default)
    # Stage: DB_HOST=stage-db.internal
    # Prod: DB_HOST=prod-db.internal
    # Zero code changes between environments.
```

```java
// ✅ Central constants class — single source of truth, compile-time safe
public final class SecurityEntities {
    
    public static final String SCENARIOS = "scenarios";
    public static final String PROJECTS  = "projects";
    public static final String WEBSITES  = "websites";
    public static final String USERS     = "users";
    
    private SecurityEntities() {} // prevent instantiation
}

// Usage — now a typo is a compile error, not a silent runtime bug:
if (!accessService.hasAccess(jwtToken, SecurityEntities.SCENARIOS, action)) { ... }
```

---

## 7. Naming Conventions & Documentation Gaps

> **🟡 SEVERITY: MEDIUM**  
> **Persistent Areas:** `WebsiteController`, `PythonApiService` *(to be renamed)*

### Findings

- Mixed `snake_case` (`project_id`) and `camelCase` (`projectId`) within the same codebase
- Implementation-focused method names like `sendGetRequestTo...` describe *how* it works rather than *what* it achieves
- Large methods with complex side-effects (manual DB updates inside API calls) lack any Javadoc
- Class-level naming anti-pattern (`PythonController`) as detailed in Section 2

### Impact

- Inconsistent naming creates confusion during code reviews and cross-team collaboration
- Implementation-focused names must be updated whenever the implementation changes
- Missing documentation forces developers to reverse-engineer behavior from source code

### Code Evidence

```java
// ❌ MIXED NAMING: snake_case in path variable, camelCase in Java — pick one
@GetMapping("/v1/{project_id}")
public ResponseEntity<Object> getWebsitesByProjectId(
        @PathVariable("project_id") String projectId,  // snake_case → camelCase mapping
        @RequestParam int page,
        @RequestParam int size) { ... }
```

```java
// ❌ IMPLEMENTATION-FOCUSED NAME: tells you HOW, not WHAT
//    If the protocol changes from REST to gRPC, this name becomes wrong

public Object sendGetRequestToScenarioReportService(String scenarioId, String token) {
    // What does this method DO for the business?
    // It gets a scenario report.
    // The name should be: getScenarioReport()
    ResponseEntity<String> response = restTemplate.exchange(/* ... */);
    return parseResponse(response);
}
```

```java
// ❌ HIDDEN SIDE-EFFECT: No documentation warns that a "get" method modifies the DB
public Object getScenarioReport(String scenarioId, String token) {
    // ... fetches report from external service ...
    
    // SURPRISE: This "get" method also writes to the database
    // No Javadoc, no method name hint — you only discover this by reading 50 lines
    scenarioRepository.updateLastAccessTime(scenarioId, LocalDateTime.now());
    
    return report;
}
```

### Recommended Remediation

```java
// ✅ STANDARDIZE: Use camelCase in Java, @JsonProperty for external mapping
@GetMapping("/v1/{projectId}")  // camelCase in path
public ResponseEntity<Object> getWebsitesByProjectId(
        @PathVariable String projectId,  // clean — no mapping needed
        @RequestParam int page,
        @RequestParam int size) { ... }

// For external APIs that require snake_case:
public record ProjectDTO(
    @JsonProperty("project_id") String projectId,
    @JsonProperty("created_at") LocalDateTime createdAt
) {}
```

```java
// ✅ BUSINESS-INTENT NAME: tells you WHAT, not HOW
/**
 * Retrieves the scenario report for the given scenario.
 * 
 * <p>Note: Updates the scenario's last-accessed timestamp as a side effect.
 * 
 * @param scenarioId the scenario identifier
 * @param token      JWT authorization token
 * @return the scenario report with all test case results
 * @throws ResourceNotFoundException if the scenario does not exist
 */
public ScenarioReportDTO getScenarioReport(String scenarioId, String token) {
    ScenarioReportDTO report = reportClient.fetchReport(scenarioId, token);
    scenarioRepository.updateLastAccessTime(scenarioId, LocalDateTime.now());
    return report;
}
```

---

## 8. Master Remediation Roadmap

The following phased roadmap addresses the identified issues in order of priority and dependency. Each phase builds on the previous one to minimize disruption while maximizing incremental improvement.

### Phase 1 — Rename & Consolidate *(Immediate)*

| Action | Expected Outcome |
|--------|-----------------|
| Rename `PythonController` → `ReportController` / `ExecutionController` | Class names reflect business function, not downstream technology |
| Centralize `SecurityUtils` component | Removes ~25% of redundant controller code across 5+ files |
| Eliminate all language-based naming across codebase | Future-proof naming that survives technology changes |

### Phase 2 — Decouple & Split *(Week 2–3)*

| Action | Expected Outcome |
|--------|-----------------|
| Decompose `PythonApiService` → `ReportClient`, `WorkloadClient`, `ExecutionClient` | 710-line God class replaced by 3 focused, testable services |
| Inject `RestTemplate` and `ObjectMapper` as Spring-managed Beans | Unit testing becomes possible for all external integrations |
| Implement DTOs for all external API responses | Type-safe communication; eliminates fragile `JsonNode` navigation |

### Phase 3 — Stabilize & Externalize *(Week 3–4)*

| Action | Expected Outcome |
|--------|-----------------|
| Replace all hardcoded IPs with `${PLACEHOLDER:default}` env vars | Zero-code-change deployments across Dev/Stage/Prod |
| Move all string literals to `SecurityConstants` / `SecurityEntities` | Compile-time safety for permission checks; eliminates magic strings |
| Standardize naming: `camelCase`, business-intent method names | Consistent codebase; faster onboarding; cleaner code reviews |

### Phase 4 — Validate & Document *(Week 4–6)*

| Action | Expected Outcome |
|--------|-----------------|
| Build regression safety net: unit tests for all newly decoupled services | Confidence in changes; defects caught before production |
| Add Javadoc for all public API methods | Side-effects documented; contracts explicit |
| Establish coding standards doc + enforce via linting | Prevents regression to old patterns; self-enforcing quality |

---

*Confidential — For leadership and architecture review only*
