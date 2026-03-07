# The Validate Phase: How I Catch AI Code Issues Before They Reach My Tests

> **Series context:** This is a deep-dive follow-up to [How I Automate Parts of My SDLC with AI Agents](#). If you haven't read that post, the short version: I built an agentic dev workflow (ADW) that automates my full development cycle: Plan → Build → **Validate** → Test → Review → Document. This post focuses on the Validate phase.

---

## Why Validation Is the Most Underrated Phase

- The elephant in the room: AI-generated code is fast but imperfect
- Linters and static analysis exist for human written code why would AI-written code get a free pass?
- Without a validate phase, imperfections land directly in your test agent (or worse, in review)
- The validate phase is the quality gate that makes the rest of the pipeline trustworthy
- Quick recap of where it sits in the pipeline:

```plaintext
Plan → Build → [Validate ×3] → [Test ×3] → Review → Document
                    ↑
              You are here
```

---

## What Validation Is NOT (Scope Clarity)

- Not running unit tests that is the Test Agent's job (separate agent, separate concerns)
- Not running the application
- No external service calls or DB connections
- **Purely static analysis** we only analyze the code itself, nothing needs to execute

> This separation is intentional. Each agent does one thing well (SRP). Keeping validation static means it is fast enough to retry 3 times without killing your pipeline's momentum and not burning a hole in your wallet.

---

## The Tool Stack

### JavaScript / TypeScript (the original)

- ESLint with custom architectural rules
- Custom rules enforce things like: no direct fetch in components, no model imports in routes
- One command, JSON output, done
- I create custom claude commands that encapsulate the exact flow each agent needs, so a simplified version of the validation agent's command looks like this:

```bash
cd backend && npm run validate:architecture:json
cd frontend && npm run validate:architecture:json
```

### Java / Spring Boot (the new addition)

The same philosophy, different tools. Here is the parallel:

| Concern        | JS Tool             | Java Tool   |
| -------------- | ------------------- | ----------- |
| Architecture   | ESLint custom rules | ArchUnit    |
| Code style     | ESLint              | Checkstyle  |
| Code smells    | ESLint plugins      | PMD         |
| Bug patterns   | —                   | SpotBugs    |
| Fast fail gate | implicit            | mvn compile |

**Execution order matters fastest checks first:**

```bash
# 1. Fast fail stop here if this breaks, no point running anything else
mvn compile -q

# 2. Style + formatting
mvn checkstyle:check

# 3. Code smells and complexity
mvn pmd:check

# 4. Bytecode-level bug patterns
mvn spotbugs:check

# 5. Architecture rules only isolated by JUnit tag
mvn test -Dgroups=architecture -Dsurefire.failIfNoSpecifiedTests=false
```

> **Why compile first?** Most Java static analysis tools require compiled bytecode. A compile failure is also the cheapest signal no point running ArchUnit on code that does not compile.

---

## Architecture Rules with ArchUnit

- Brief intro: ArchUnit lets you write your architecture decisions as executable tests
- These are not regular unit tests they validate structure, not logic
- Tag them separately so the Validation Agent and Test Agent have zero overlap

```java
@Tag("architecture")
@AnalyzeClasses(packages = "com.yourapp")
class ArchitectureRules {

    // Controllers must not call repositories directly
    @ArchTest
    static final ArchRule no_direct_repo_in_controllers =
        noClasses()
            .that().resideInAPackage("..controller..")
            .should().dependOnClassesThat()
            .resideInAPackage("..repository..");

    // Services must not import Spring MVC annotations
    @ArchTest
    static final ArchRule services_must_not_use_mvc =
        noClasses()
            .that().resideInAPackage("..service..")
            .should().dependOnClassesThat()
            .resideInAPackage("org.springframework.web.bind.annotation..");

    // Naming conventions enforced
    @ArchTest
    static final ArchRule controllers_named_correctly =
        classes()
            .that().resideInAPackage("..controller..")
            .should().haveSimpleNameEndingWith("Controller");

    // @Transactional only allowed on service layer
    @ArchTest
    static final ArchRule transactional_only_on_services =
        noClasses()
            .that().resideOutsideOfPackage("..service..")
            .should().beAnnotatedWith(Transactional.class);
}
```

The Test Agent excludes this tag so there is zero overlap between the two agents:

```bash
# Validation Agent runs ONLY architecture tests
mvn test -Dgroups=architecture -Dsurefire.failIfNoSpecifiedTests=false

# Test Agent runs everything EXCEPT architecture tests
mvn test -DexcludedGroups=architecture
```

---

## The Standardized Violation Schema The Secret Sauce

- I find each tool has its own output format it can be noisy and inconsistent across tools
- The agent cannot reliably reason about what to fix if the input format varies per tool
- Solution: normalize everything into one consistent JSON schema before feeding it into the fix loop
  - We can use AI to help with this normalization step write a prompt that takes raw tool output and maps it to the schema this can be done by giving it a few examples of the input and output format that are required.
- This is the same schema used in the JS version the contract does not change, only the tools that populate it do
  - Tools change but the schema is stable and consistent across languages this is the key to making the rest of the pipeline tool-agnostic.

```json
[
  {
    "rule": "ArchUnit/no-direct-repo-in-controllers",
    "file": "src/main/java/com/app/controller/UserController.java",
    "line": 34,
    "column": null,
    "severity": "error",
    "message": "Controllers should not import repositories directly. Use a service.",
    "fix_suggestion": "Replace UserRepository injection with UserService. Controllers should only depend on the service layer."
  },
  {
    "rule": "checkstyle/MethodLength",
    "file": "src/main/java/com/app/service/OrderService.java",
    "line": 87,
    "column": 1,
    "severity": "warning",
    "message": "Method length is 72 lines (max 50).",
    "fix_suggestion": "Extract the validation logic into a private helper method to reduce method length."
  }
]
```

**Two severity levels, two behaviors:**

- `severity: "error"` → fails validation, triggers the auto-fix retry loop
- `severity: "warning"` → logged and visible but does not fail the phase

**A note on fix_suggestion for Java:** ESLint can generate suggestions natively. Java tools cannot. Instead, maintain a small rule registry a lookup map of rule name → suggestion string that the normalizer uses when building the output. Upfront effort, but it pays off every retry cycle.

---

## The Auto-Fix Retry Loop

- When violations are found the agent does not stop it feeds the structured violations back to the LLM to fix, then re-validates
- Hard cap at 3 retries before escalating to human intervention
- Two failure modes to guard against:

```plaintext
Build Output
     ↓
Run Validation Tools
     ↓
Normalize all output → JSON violations array
     ↓
violations.length > 0?
  YES → Feed violations to fix agent → Re-validate (max 3 attempts)
  NO  → Phase complete ✅
     ↓
Still failing after 3 retries? → Halt, surface to human 🛑
```

**Violation diffing between retries** track the count before and after each fix attempt. If the count is not going down, the agent is stuck. Escalate instead of looping.

**Regression detection** if new violations appear that were not present before a fix attempt, the fix introduced a regression. Treat this as a separate signal and re-run from the last clean state rather than continuing forward.

**What this looks like in your pipeline output:**

```plaintext
Phase 3: Validation
======================================================================
SUCCESS
   Critical: 0, Warnings: 2, Attempts: 2

   Found 4 violations on first pass
   Auto-fixed: controller importing repository directly, method too long,
               missing @Override annotation, unused import
   Re-validated: CLEAN
```

---

## What About SonarQube?

- You might already have SonarQube running in CI does it belong here too?
- **Short answer: no, not inside the validation agent**
- `mvn sonar:sonar` is slow and expensive a bad fit for a loop that may run 3 times
- This agent runs **before a push**, so there is no Sonar result to even poll yet
- SonarQube's natural home is post-push in CI, as a final safety net before merge
- Instead, run **SonarLint in connected mode** locally in your IDE same quality profile as your server, zero pipeline cost

> The principle: order checks by cost. Cheap and fast first, expensive later (or delegate to CI entirely). This is why compile runs before Checkstyle, and Checkstyle before ArchUnit.

---

## Before and After: What Your Test Agent Receives

**Without a validation agent** the test agent receives raw AI output that may include:

- A controller calling a repository directly (arch violation)
- A method that is 90 lines long (PMD)
- Unused imports (Checkstyle)
- A missing null check (SpotBugs)
- The test agent now has to fight bad structure AND broken tests simultaneously

**With a validation agent** the test agent receives code that has already passed:

- Compile check
- Style and formatting rules
- Smell and complexity thresholds
- Bug pattern analysis
- Your architectural boundaries

The test agent works with clean, well-structured code every time. That is why your test agent rarely needs all 3 of its own retries.

---

## Key Takeaway

The validate phase is not about distrusting AI. It is about applying the same rigor to AI generated code that you apply to any code the same linters, the same architectural rules, the same standards your team already agreed on. The difference is it runs automatically, fixes itself, and only escalates to you when it genuinely cannot resolve the issue.

> The code that reaches your Test Agent has already been through a compile check, style validation, smell detection, bug pattern analysis, and your architecture rules. You are not reviewing raw AI output. You are reviewing code that has already been through the gauntlet.

---

## What's Next

- I plan on writing a post around the Review Agent next why its needed and how can we apply a but more automation if there are any issues found during that phase.
- The Review Agent did the AI actually build what the spec asked for?

---

_Tags: `#ai` `#claudecode` `#java` `#springboot` `#softwaredevelopment` `#productivity`_
