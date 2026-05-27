# Isagawa QA Platform (Selenium)

![License: Proprietary](https://img.shields.io/badge/License-Proprietary-red)
![Python](https://img.shields.io/badge/Python-3.10%2B-blue)
![Selenium](https://img.shields.io/badge/Selenium-4.15%2B-green)

AI-powered Selenium test automation with a 5-layer architecture and runtime enforcement. Describe a requirement in plain English, and the AI agent discovers page elements, generates Page Objects, Tasks, Roles, and Tests following strict separation of concerns. Every action is gated by the [Isagawa Kernel](https://github.com/isagawa-co/isagawa-kernel), so the AI can only produce code that matches your architecture.

Built on the [Isagawa Kernel](https://github.com/isagawa-co/isagawa-kernel).

---

## The Problem

AI can generate Selenium tests in seconds. But without enforcement:

- Tests break existing architecture patterns
- Page Objects get skipped or mixed with business logic
- The same mistakes repeat across every session
- You spend more time fixing AI output than writing tests yourself

The cycle: generate, breaks something, fix, generate, breaks it differently, start over.

---

## The Solution

The Isagawa QA Platform combines a **5-layer test architecture** with the **Isagawa Kernel**, a self-building enforcement system that runs inside the AI agent.

The kernel does not monitor the AI from outside. It manages the AI from within. The agent reads your reference implementations before writing anything, follows your patterns exactly, and gets permanently smarter after every failure.

The result: tests that follow your architecture on the first pass, every time.

---

## How It Works

When you invoke `/qa-workflow`, the agent runs a 5-step pipeline with self-enforcing gates at every step.

```
/qa-workflow
    |
Step 1: USER INPUT
    Receive requirement, URL, credentials
    Gate: requirement has persona, action, URL
    |
Step 2: PRE-FLIGHT
    Verify URLs accessible, check environment config
    Gate: all URLs respond, config file valid
    |
Step 3: AI PROCESSING
    Open browser via Playwright MCP, discover page elements
    Gate: element map captured for each URL
    |
Step 4: CONSTRUCTION
    Read reference files, generate Page Objects, Tasks, Roles, Test
    Gate: all files follow 5-layer architecture, locators only in POMs
    |
Step 5: EXECUTION
    Run pytest, capture results
    Gate: test passes or failure triaged with user
    |
COMPLETE
```

**Example output:**

```
Input:
  "As an employee manager, I want to create an employee
   and assign them a task"
  URL: https://myapp.com/employees, https://myapp.com/tasks

  |
  +-- Discover: Playwright MCP opens browser, maps all elements
  +-- Generate: LoginPage, EmployeesPage, TasksPage (POMs)
  |             EmployeeManagementTasks, TaskManagementTasks (Tasks)
  |             EmployeeManager, TaskManager (Roles)
  |             TestE2ECreateEmployeeAndAssignTask (Test)
  +-- Execute: pytest runs, 1 passed in 12.31s
  +-- Learn: patterns recorded, next test is better
```

---

## Architecture

### 5-Layer Separation of Concerns

| Layer | Responsibility | Example |
|-------|---------------|---------|
| **BrowserInterface** | Selenium WebDriver wrapper. Waits, clicks, typing, screenshots, logging. | `BrowserInterface.click()`, `.enter_text()` |
| **Page Object** | Locators as class constants. Atomic methods. Return self. State-checks. | `LoginPage.enter_email("user@test.com")` |
| **Task** | One domain operation. Composes Page Objects. `@autologger("Task")`. | `EmployeeManagementTasks.create_employee()` |
| **Role** | User persona workflow. Composes Tasks. `@autologger("Role")`. | `EmployeeManager.create_employee()` |
| **Test** | AAA pattern. Pytest fixtures. Assert via POM state-checks. | `test_e2e_create_employee_and_assign_task()` |

```
Test (Arrange / Act / Assert)
  +-- Role (multi-task workflow, user persona)
       +-- Task (single domain operation)
            +-- Page Object (one page, atomic actions, fluent API)
                 +-- BrowserInterface (Selenium wrapper, waits, logging)
```

### Reference Implementations

The framework ships with canonical implementations in `framework/_reference/`. The agent reads these before generating any code.

| File | Layer | Purpose |
|------|-------|---------|
| `pages/login_page.py` | Page Object | Login form locators and actions |
| `pages/employees_page.py` | Page Object | Employee list and creation modal |
| `pages/tasks_page.py` | Page Object | Task list and assignment modal |
| `tasks/employee_management_tasks.py` | Task | Login + employee CRUD operations |
| `tasks/task_management_tasks.py` | Task | Task creation and assignment |
| `roles/employee_manager.py` | Role | Employee management workflows |
| `roles/task_manager.py` | Role | Task management workflows |
| `tests/test_e2e_create_employee_and_assign_task.py` | Test | Full integration: create employee, assign task |

---

## Kernel Enforcement

The [Isagawa Kernel](https://github.com/isagawa-co/isagawa-kernel) gates every action at runtime. It is not a linter or post-hoc checker.

- **Session gating.** The agent cannot write code until `/kernel/session-start` initializes the session and loads protocol state.
- **Anchor cycling.** Every 10 actions, the hook forces the agent to re-read its protocol via `/kernel/anchor`, preventing drift from architecture patterns.
- **Failure capture.** When a test fails, the hook sets `needs_learn: true` and blocks further writes until `/kernel/learn` records the lesson permanently.
- **Reference-first construction.** Before generating any file, the agent reads the corresponding reference implementation in `framework/_reference/`. No reference, no generation.
- **Human-in-the-loop triage.** On any failure, the agent stops and presents four options (fix, investigate, skip, abort). It never loops through autonomous fixes.

---

## Quick Start

### Prerequisites

- Python 3.10+
- Node.js 18+ (for Playwright MCP)
- Chrome or Brave browser
- [Claude Code](https://claude.ai/claude-code)

### Install

```bash
git clone https://github.com/isagawa-qa/platform-selenium.git
cd platform-selenium
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
```

### Configure

Edit `framework/resources/config/environment_config.json` to add your application URL:

```json
{
  "myapp": {
    "url": "https://staging.your-app.com"
  }
}
```

### Run

```bash
claude                    # Start Claude Code in the project directory
> start                   # Agent runs domain-setup automatically
                          # --> "Restart Claude Code to activate hooks"
claude                    # Restart
> continue                # Agent anchors and is ready
> /qa-workflow            # Generate your first test
> /pr                     # Review generated code against architecture
```

### Tests

```bash
# Default (Chrome, headed)
pytest tests/

# Headless Chrome
pytest tests/ --headless

# Brave browser
pytest tests/ --browser brave

# Specific environment
pytest tests/ --env staging

# With HTML report
pytest tests/ --html=report.html --self-contained-html
```

---

## Project Structure

```
platform-selenium/
+-- .claude/
|   +-- commands/                    # Kernel + QA workflow commands
|   |   +-- qa-workflow.md           # /qa-workflow (5-step test generation)
|   |   +-- qa-workflow-dev.md       # /qa-workflow-dev (dev mode)
|   |   +-- pr.md                    # /pr (code review against architecture)
|   +-- hooks/                       # Gate enforcer + test failure detector
|   |   +-- universal-gate-enforcer.py
|   |   +-- test-failure-detector.py
|   +-- skills/
|   |   +-- kernel-domain-setup/     # Self-building kernel setup
|   |   +-- qa-management-layer/     # 5-step QA workflow skill
|   |       +-- SKILL.md             # Entry point, rules, reading order
|   |       +-- workflow.md          # 5-step index with data flow
|   |       +-- gate-contract.md     # Validation contract (6 responsibilities)
|   |       +-- steps/               # Step-specific criteria
|   +-- settings.json                # Hook configuration
+-- framework/
|   +-- _reference/                  # Canonical code patterns (read-before-write)
|   |   +-- pages/                   # POM reference implementations
|   |   +-- tasks/                   # Task reference implementations
|   |   +-- roles/                   # Role reference implementations
|   |   +-- tests/                   # Test reference implementations
|   +-- interfaces/
|   |   +-- browser_interface.py     # BrowserInterface (Selenium wrapper)
|   +-- resources/
|       +-- chromedriver/            # Driver factory
|       +-- config/                  # Environment configuration
|       +-- utilities/               # Autologger decorator
+-- tests/
|   +-- data/                        # Test data (credentials, fixtures)
|   +-- conftest.py                  # Pytest fixtures and configuration
+-- docs/
|   +-- architecture.md
|   +-- getting-started.md
+-- .mcp.json                        # Playwright MCP server config
+-- CLAUDE.md                        # Kernel instructions
+-- CONTRIBUTING.md                  # Architecture rules and PR process
+-- requirements.txt
+-- LICENSE
```

---

## Other Platforms

| Platform | Domain | Repo |
|----------|--------|------|
| **platform-selenium** | Selenium test automation | (this repo) |
| **platform-ssh** | SSH compliance automation | [isagawa-qa/platform-ssh](https://github.com/isagawa-qa/platform-ssh) |

---

## Contact

**Email:** [alain@isagawa.co](mailto:alain@isagawa.co)
**Web:** [isagawa.co](https://isagawa.co)

---

## License

[Proprietary](LICENSE). Copyright (c) 2025 Isagawa. All rights reserved. Evaluation use only. Contact [alain@isagawa.co](mailto:alain@isagawa.co) for commercial licensing.
