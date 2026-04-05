# How I Automate Parts of My Software Development Lifecycle with AI Agents

Every developer knows the drill: You get a feature request. You create a branch. You write a plan (maybe). You implement. You write tests. You review. You document. Rinse and repeat. What if an AI could handle the tedious parts while you focus on the interesting problems? That's exactly what I built, I call it AI Developer Workflows (ADW). In this post, I'll show you how I automated the complete software development lifecycle using AI agents, and how you can do the same, I have templates for typescript, golang, and java, however it can easily be adjusted to other languages you just need to update the prompt commands.

## My day to day development workflow

Most of my day wasn't spent solving interesting problems. It was spent on ceremony. Can I take my workflow and get AI to automate parts or all of it? Here is my day before my AI workflow:

1. Read through all the Jira tasks and find the one I like the most, assign to myself. Hoping the details exist and it's NOT just a one liner that the PM created.
2. **Planning**: Read a lot of code and decide HOW my feature will fit into the code base.
3. **Implementation Plan**: Once I have all relevant files or identified the areas of change, I create a document to keep track of all the changes that are needed.
4. **Implementation**: Start coding!
5. Oh yeah tests: I then write my tests, I know I probably should follow TDD or some framework.
6. **Review**: Now that everything is done lets compare the feature with the actual jira ticket did we build the correct thing? Did We miss anything? hopefully not and also pray for NO scope creep.
7. **Documentation**? Lol

## The Solution: AI Developer Workflow (ADW)

ADW is a framework that orchestrates AI agents through a complete SDLC pipeline. The idea is to have one agent perfect one task extremely well (SRP), vs trying to get a single agent to perform multiple tasks.

```markdown
┌───────────────────────────────────────────────────────────────────────┐
│ ADW Pipeline │
├───────────────────────────────────────────────────────────────────────┤
│ │
│ Prompt ──► Plan ──► Build ──► Validate ──► Test ──► Review ──► Doc │
│ │ │ │ │ │ │ │
│ ▼ ▼ ▼ ▼ ▼ ▼ │
│ Spec Code Quality Fixes Issues Docs │
│ File Changes Enforced Applied Fixed Created │
│ │
└───────────────────────────────────────────────────────────────────────┘
```

### 1. Plan Phase:

For larger code bases this can be broken into two parts: **Research** and **Planning**, the **research agent** does a deep analysis on what are the relevant files before passing this to the **planning agent** just to control a bit more of the context window. Explained in a [Presentation](https://youtu.be/eIoohUmYpGI?si=7JUODxAs0FqnVqmq&t=665): "I shipped code I don't understand and I bet you have too" by Jake Nations.

Next you provide a prompt like "**Add user authentication with JWT tokens.**" The planning agent:

- Researches your codebase
- Identifies relevant files and patterns (Important)
- Creates a detailed implementation spec
- Outputs a structured plan file

### 2. Build Phase

The builder agent reads the spec and:

- Implements the feature following your codebase patterns
- Creates necessary files and modifications
- Follows existing conventions automatically

### 3. Validate Phase (The "AI Writes Bad Code" Killer)

This is the phase that addresses the elephant in the room: "**But AI-generated code is garbage!**"

The validation agent:

- Runs linters, static analysis, and architectural rules
- Catches anti-patterns, code smells, and style violations
- Automatically fixes violations and retries
- Enforces YOUR coding standards, not generic ones

This isn't just `go fmt`. It's running tools like `golangci-lint` with your custom ruleset, checking for security issues, verifying architectural boundaries, and ensuring the AI generated code follows the same standards as us humans. For non-golang'ers there are tools like **ArchUnit** (java) and **ArchUnitTS** (typescript) to Enforce architecture rules.

If the AI writes code that violates your standards? The validation agent fixes it automatically, then re-validates. Up to N retries until it's clean.

### 4. Test Phase

This agent I would say is one of the more important agents, if you get this one done correctly you shouldnt hit any regressions (mostly). What I like to do here is setup both unit tests and integration tests, the ensure that the slash command know how to execute them both. This way if we break anything this agent will find the issues and correctly fix them. NOTE: we also need to explain HOW can AI troubleshoot issues, we need to have good logging (any production app should have great logging) and again tell AI how to search these logs if it does encounter issues. Do this well and you will save A LOT of tokens and time. In my apps I always add centralized logging and explain to AI how to search these logs effectively.

The test agent:

- Runs your test suite
- If tests fail, analyzes the failures
- Attempts to fix issues automatically
- Retries up to N times (configurable)

### 5. Review Phase

The review agent:

- Compares implementation against the original spec
- Identifies gaps, bugs, or missing requirements
- Categorizes issues by severity (blocker, tech debt, skippable)
- Creates a review report (We have an agent to resolve these issues: travis_patch.py if there are blockers)

### 6. Document Phase

The documentation agent:

- Generates user-facing documentation
- Updates relevant README sections
- Creates API documentation if applicable
- We update our conditional_docs.md, this allows us to conditionally load documentation when we are using `/feature`, essentially allowing AI to dynamically load documents if they are required in the new feature.

### What This Looks Like in Practice

Here's the magic. One command:

```markdown
~ uv run travis_sdlc.py "Add rate limiting to the API endpoints"

======================================================================
Travis SDLC Workflow
ADW ID: a1b2c3d4
======================================================================

# Phase 1: Planning

✅ SUCCESS
File: specs/feature-a1b2c3d4-api-rate-limiting.md

# Phase 2: Implementation

✅ SUCCESS

# Phase 3: Validation

✅ SUCCESS
Critical: 0, Warnings: 2, Attempts: 2

↳ Found 3 violations on first pass
↳ Auto-fixed: unused variable, missing error check, import order
↳ Re-validated: CLEAN

# Phase 4: Testing

✅ SUCCESS
Passed: 47, Failed: 0, Attempts: 1

# Phase 5: Review

✅ SUCCESS
Issues: 0

# Phase 6: Documentation

✅ SUCCESS
Path: app_docs/feature-a1b2c3d4-api-rate-limiting.md

======================================================================
✅ WORKFLOW COMPLETED SUCCESSFULLY
======================================================================
```

Notice **Phase 3**: The validation found 3 violations, **automatically fixed** them, and re-validated. Acknowledging that AI does NOT always write the best code, we need to put in checks into our agents that will enforce coding standards. 

From a single prompt to a fully implemented, tested, reviewed, and documented feature.

## Skeptic's Corner: Addressing the Hard Questions

I been working with AI for over a few years now, one of the most common push back is quality and the quantity of code outputted.

### AI-generated code is unmaintainable garbage.

**You are correct!** Unfiltered AI output often has issues: unused variables, missing error handling, duplicate methods or trying to re-build a class we already have, …etc. Here is the thing, human developers write code with issues too. This is why we have linters, code reviews, and CI pipelines. My approach of the ADW applies the same rigor to AI generated code through the **Validate Phase**.

```markdown
┌─────────────────────────────────────────────────────────────────┐
│ Validation Phase Loop │
├─────────────────────────────────────────────────────────────────┤
│ │
│ Build Output ──► Validate ──► Violations? ──► Auto-Fix ──┐ │
│ ▲ │ │
│ └────────────── Re-validate ◄────────┘ │
│ │
│ Max 3 retries, then human intervention required │
│ │
└─────────────────────────────────────────────────────────────────┘
```

The validation phase runs `golangci-lint`, `eslint`, security scanners, and **your custom architectural rules**. If violations are found, the agent fixes them automatically and revalidates. The AI doesn't write perfect code. But the **system catches and corrects mistakes before they reach you.**

### AI just creates tech debt that I'll have to clean up later.

This is a valid concern. AI can take shortcuts, copy-paste patterns inappropriately, or ignore edge cases. The ADW addresses this with the **Review Phase:** I have below a recent log that found a few issues in my application. This phase can be customized to FAIL if a condition is met I currently have it set to only fail on blockers. I get the agent to create a detail plan on how to fix these issues the file is saved here: specs/review_issues/review-4cb749dc.md If i wanted ai to fix these i just run the command `uv run .awd/travis/travis_patch.py 4cb749dc` (the ID number of this job and it will pick everything up on its own)

```markdown
======================================================================
Phase 5: Review
======================================================================

ADW Logger initialized - ID: 4cb749dc
Travis Review starting - ADW ID: 4cb749dc
Reviewing implementation against spec: specs/feature-a1b2c3d4-nuclei-vulnerability-scanning.md

Review Summary:
Status: PASSED
Tests: PASSED
Build: PASSED
Summary: The Nuclei vulnerability scanning feature has been successfully implemented with all core functionality working as specified. The implementation includes proper CLI commands (vuln scan, vuln list), database persistence with migration, Nuclei tool integration, and comprehensive test coverage. All tests pass and the build succeeds. Minor issues exist around missing config validation, incomplete findings integration, and lack of repository tests, but none are blocking the release of this feature.

Issues Found: 7

Issue #1:
Severity: tech_debt
File: internal/cli/vuln_scan.go:541
Description: The --custom flag uses BuildCustomTemplateArgs but the implementation uses -templates flag which differs from Nuclei's actual -t flag for custom templates. This may cause issues when users try to specify custom template paths.
Resolution: Update BuildCustomTemplateArgs in pkg/recon/vulnscan/nuclei.go to use -t flag instead of -templates flag for custom template paths, matching Nuclei's actual CLI interface.

Issue #2:
Severity: skippable
File: configs/default.yaml:181
Description: The config includes 'severity' and 'exclude_templates' fields, but these are not validated or used anywhere in the codebase. The CLI always requires explicit --severity or --templates flags.
Resolution: Either implement support for reading default severity and exclude_templates from config file in the scan command, or remove these unused fields from the config schema to avoid user confusion.

Issue #3:
Severity: tech_debt
File: specs/feature-a1b2c3d4-nuclei-vulnerability-scanning.md:209-212
Description: The spec mentions 'Auto-create findings from critical/high severity results' and 'Link vuln_scans to findings table' as acceptance criteria, but this integration is not implemented. The code only stores in vuln_scans table without creating finding entries.
Resolution: Add integration with the findings table to auto-create findings for critical/high severity vulnerabilities. This can be done by calling the findings repository after saving vuln scans with high severity.

Issue #4:
Severity: tech_debt
File: internal/repository/vuln_scan.go
Description: No tests exist for the VulnScanRepository despite the repository having complex JSON serialization logic for references and extracted_results. This creates risk of bugs in database operations.
Resolution: Add unit tests for VulnScanRepository covering Create, GetByID, GetBySeverity, GetByHost, GetByCVE, Exists, and JSON serialization/deserialization of array fields.

Issue #5:
Severity: skippable
File: .gitignore:9
Description: The bbrecon binary was removed from .gitignore, which will cause the compiled binary to be tracked by git. This is generally not desired for build artifacts.
Resolution: Add 'bbrecon' back to .gitignore to prevent the binary from being committed to the repository.

Issue #6:
Severity: tech_debt
File: pkg/recon/vulnscan/nuclei.go:1762-1768
Description: BuildTemplateArgs uses -tags flag for template categories, but this may not match Nuclei's expected behavior. Nuclei template categories like 'cves' typically use the -t flag with path like '-t cves/' not '-tags cves'.
Resolution: Verify the correct Nuclei flag for template categories and update BuildTemplateArgs to use -t flag with category paths (e.g., '-t cves/') instead of -tags. Add integration test with actual Nuclei to verify.

Issue #7:
Severity: skippable
File: internal/cli/vuln_scan.go:702
Description: The getTargetsFromDB function only builds HTTPS URLs but some services may only be accessible via HTTP. This could miss vulnerabilities on HTTP-only services.
Resolution: Update getTargetsFromDB to check the subdomain's HTTPScheme or URL field if available, or build both HTTP and HTTPS URLs based on the actual probed protocol from the web recon phase.
Review issues written to specs/review_issues/review-4cb749dc.md

Review issues file created: specs/review_issues/review-4cb749dc.md
Review phase completed successfully

Phase 5: Review: ✅ SUCCESS
```

The review agent specifically looks for:

- Spec compliance: Did we actually build what was planned?
- Tech debt indicators: Shortcuts, TODOs, incomplete error handling
- Missing edge cases: What happens when X fails?

Issues are categorized by severity. **Blockers** stop the workflow. **Tech debt / skippable** is documented but doesn't block. **You decide** what bar to set.

### AI doesn't understand MY codebase. It'll write code that doesn't fit.

This is why the **Planning Phase** exists, before writing any code, the planning agent:

1. Reads your `README.md` and `DESIGN.md`
2. Searches for similar coding patterns in your codebase.
3. Identifies the files it needs to modify.
4. Creates a spec that follows YOUR conventions, does NOT make up patterns.

AI analyzes your existing patterns before generating any code. The commands are built to extend what already exists, and only create new components when nothing relevant is found.

I have not tested this but if the **Codebase is large** you can easily break this phase into two parts: 1. **Research agent**, and 2. **Planning agent**, planner utilizes the research agents results. The **Research Agent** main job is to find What is relevant to this feature within this large codebase and pass it to the planner. This way we are NOT wasting the planners context trying to find everything needed to 'X' feature.

### You still have to review everything anyway. What's the point?

Yes, we should always review everything however, here is the difference between the two:

**Without ADW:**

- Review raw AI output
- Find the 12 linting errors
- Notice missing error handling
- Realize it didn't follow your patterns
- Send it back, wait, review again

**With ADW:**

- Validation already caught the linting errors
- Tests already verified basic functionality
- Review already flagged tech debt
- I'm reviewing **polished** code, not first drafts

The code that reaches me has already **passed**:

- Linter (validation phase)
- Static analysis (validation phase)
- Unit tests (test phase)
- Spec compliance (review phase)

My review is the final check, not the first line of defense.

## The Bottom Line

**ADW doesn't trust the AI** output. It verifies, validates, tests, and reviews automatically. AI is fast but imperfect, the system is designed to **catch imperfections programmatically** before they reach you. You still need to review the code, however, it has already passed multiple quality gates not raw AI output. This ADW is a good first start to your feature, it doesn't always one shot the feature but it will get you 80–90% there.

## The Architecture

Now that we address some of the Skepticisms let's look at the architecture: ADW consists of three layers and only one layer is language-specific.

```markdown
┌─────────────────────────────────────────────────────────────────┐
│ ADW Architecture │
├─────────────────────────────────────────────────────────────────┤
│ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Layer 3: Orchestrator (Python) LANGUAGE-AGNOSTIC │ │
│ │ travis_sdlc.py - chains phases, manages state │ │
│ └─────────────────────────────────────────────────────────┘ │
│ │ │
│ ▼ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Layer 2: Slash Commands (Markdown) LANGUAGE-SPECIFIC │ │
│ │ /test, /validate, /review - customize per language │ │
│ └─────────────────────────────────────────────────────────┘ │
│ │ │
│ ▼ │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Layer 1: Agent Module (Python) LANGUAGE-AGNOSTIC │ │
│ │ Claude Code execution, retry logic, state management │ │
│ └─────────────────────────────────────────────────────────┘ │
│ │
└─────────────────────────────────────────────────────────────────┘
```

### Layer 1: Core Agent Module (Language-Agnostic)

The foundation that handles:

- Claude Code CLI execution
- Retry logic for transient failures
- Output parsing (JSONL → JSON)
- State management

This never changes regardless of what language your project uses.

### Layer 2: Slash Commands (Language-Specific)

This is where the magic happens. Slash commands are markdown templates that define **HOW** each phase executes for YOUR language:

```markdown
.claude/commands/
├── test.md # "Run go test ./..." or "npm test" or "mvn test"
├── validate.md # "Run golangci-lint" or "eslint" or "checkstyle"
├── feature.md # Plan format with language-specific patterns
├── implement.md # Implementation instructions
├── review.md # Review criteria
└── document.md # Documentation format
```

To support a new language, you only customize these files. For example, here's how `/test` differs by language, to ensure these work in claude code just open the claude terminal and type `/test` if your test run correctly this is how the agent will execute this command.

```markdown
## Test Execution

- Command: `go test ./... -v -race -coverprofile=coverage.out`
- test_name: "go_test"

**TypeScript:**

## Test Execution

- Command: `npm test -- --coverage --watchAll=false`
- test_name: "jest_test"

**Java:**

## Test Execution

- Command: `mvn test -B`
- test_name: "maven_test"
```

Same orchestrator. Same workflow. Different language-specific commands

### Layer 3: Orchestrator (Language-Agnostic)

The `travis_sdlc.py` script that:

- Chains phases together
- Manages state between phases
- Handles failures and retries
- Provides observability and logging

This is pure Python and doesn't know or care what language your project uses. It just calls the slash commands and processes the results. NOTE: Each of these files can be run independently, they are meant to be isolated function calls to the claude sdk. Learned this from IndyDevDan.

### Why This Matters

This architecture means:

1. One orchestrator to maintain The Python ADW code works for any language
2. Easy to add new languages Just write new slash commands
3. Shareable workflow logic Test/retry/review logic is universal
4. Customizable per project Each repo can have its own command variations

Want to use ADW on a Rust project? Write `/test.md` with `cargo test`, `/validate.md` with `clippy`, and you're done. The orchestrator handles the rest.

## Getting Started

Prerequisites:

- Claude Code CLI installed and authenticated
- Python 3.11+ with `uv` package manager (install with brew)
- Your codebase with a README.md and basic structure

Add a .env file with the following:

```bash
CLAUDE_CODE_PATH=claude # This is the path to claude code default should be this
```

After getting the prereqs you need to EDIT the slash commands to match your repo, big brain move is get claude code to do it maybe?

```markdown
Please REVIEW the slash commands located in @.claude/commands/\*
there are some language specific files like:

- @.claude/commands/test.md
- @.claude/commands/validate.md
  ...

We need to update them to MATCH our system in this repo ensure all commands
are correctly matching our system.
```

## Example Commands

```markdown
# Simple feature

uv run travis_sdlc.py "Add a health check endpoint"

# Bug fix

uv run travis_sdlc.py "Fix the memory leak in the cache module" --plan-type bug

# Chore/refactor

uv run travis_sdlc.py "Refactor the logging to use structured output" --plan-type chore

# Use a more powerful model for complex tasks

uv run travis_sdlc.py "Implement OAuth2" --model opus

# Skip optional phases

uv run travis_sdlc.py "Quick fix" --skip-review --skip-document

# Increase test retry attempts

uv run travis_sdlc.py "Tricky feature" --max-test-retries 5
```

## What's Next

This post covered the "what" and "why" of ADW. In the next posts, I plan on explaining deeper into the following phases.

1. The Planning Phase How to write effective prompts and customize plan templates
2. Validation Enforcing code quality with linters, auto-fixes, and custom rules
3. Test & Review Handling failures, auto-fixes, and quality gates
4. Customizing Slash Commands Adapting ADW for Go, Java, TypeScript, Rust, or any language

The ADW framework is available on GitHub: [https://github.com/travism26/claude_code_agent_templates] I'd love to hear how you're using it. Drop a comment or reach out on Twitter [@travism26].

## Shoutouts

[IndyDevDan](https://www.youtube.com/@indydevdan) I took his course and a lot of the ideas I learned and expanded upon are from his course.
