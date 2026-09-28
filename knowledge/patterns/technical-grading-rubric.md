# Technical Training Capstone Project: Facilitator Grading Framework

This grading framework is designed to evaluate student competence, hands-on execution, and knowledge retention following a technical training program. Because programming languages, platforms, and technical stacks vary across cohorts, this framework focuses on agnostic technical principles, stability, and problem-solving skills rather than language-specific syntax.

Used by `/create-rubric` as the **Weighted** grading style — recommended
for the capstone, while regular hands-on modules keep the command's
**Binary** Checkpoint-gate style by default. See
`.claude/commands/create-rubric.md`.

---

## 📊 1. The 4-Tier Grading Blueprint

**Tier 1 ("Requirements & Architecture") is conditional, not always
included.** Before scoring a target, check its own documentation for an
existing "Requirements / acceptance criteria" section and an existing
"Project architecture" section:

- **If both already exist** — the target's course/spec already defines
  scope, stack, and architecture for the trainee; planning wasn't the
  trainee's own work to grade. **Exclude Tier 1** and use the
  *renormalized* weights below. This is the case for this course's
  current capstone — `capstone/capstone-end-to-end-pipeline-cohort.md`
  already has both a "Requirements / acceptance criteria" section and a
  "Project architecture" section.
- **If neither exists** (a hypothetical future target with no
  predefined spec) — the trainee had to plan the solution themselves.
  **Include Tier 1** and use the *original* weights below.

**Weights — Tier 1 excluded (today's capstone case):**

| Phase | What You Are Grading | Why It Matters | Weight |
| :--- | :--- | :--- | :--- |
| **Technical Execution** | Quality of implementation, configuration standards, security practices, and tool usage taught in training. | Direct evidence that the student absorbed the technical skills taught during the program. | **45%** |
| **Functional Demonstration** | A live, working walkthrough of the project showing all core features in action. | Proves the solution actually works end-to-end and isn't just theoretical. | **35%** |
| **Presentation & Defense** | Ability to explain technical choices, defend architecture, and handle live troubleshooting. | Simulates real-world engineering environments and technical communication. | **20%** |

**Weights — Tier 1 included (future target with no predefined spec):**

| Phase | What You Are Grading | Why It Matters | Weight |
| :--- | :--- | :--- | :--- |
| **Requirements & Architecture** | Project scope, stack selection, and system design or architecture diagrams. | Proves the student can plan a technical solution before writing code or configuring infrastructure. | **15%** |
| **Technical Execution** | Quality of implementation, configuration standards, security practices, and tool usage taught in training. | Direct evidence that the student absorbed the technical skills taught during the program. | **40%** |
| **Functional Demonstration** | A live, working walkthrough of the project showing all core features in action. | Proves the solution actually works end-to-end and isn't just theoretical. | **30%** |
| **Presentation & Defense** | Ability to explain technical choices, defend architecture, and handle live troubleshooting. | Simulates real-world engineering environments and technical communication. | **15%** |

Either way, the **Technical Documentation Gate** (§2 below) applies on
top of the weighted score — it is pass/fail, not a percentage, and
failing it fails the target overall regardless of the weighted score.

---

## ✅ 2. Technical Documentation Gate (required, pass/fail)

Applies on top of either weight table in §1 above, and applies to
**either** grading style in `/create-rubric` — Binary or Weighted, not
just the weighted model. Mirrors how negative-path testing is already a
required gate in the binary Checkpoint model, not merely one more scored
row.

- **Source**: the target's own existing `## Documentation` section
  (today, only the capstone has one — see
  `capstone/capstone-end-to-end-pipeline-cohort.md` /
  `capstone/capstone-end-to-end-pipeline-instructor.md`).
- **What it grades**: whether the trainee actually produced and
  accumulated real evidence as they built — real `pipeline.log` output
  from both the happy-path and failure-path runs, the real crontab
  entry and its cron-triggered log line, real `git status` output, and
  a real explanation of idempotency in the trainee's own words. Not
  whether the section exists with placeholder text.
- **Why it's a gate, not a score**: documentation is a practice being
  taught, not a bonus — it supplements the trainee's own understanding
  and forces them to document as they go, so it can't be averaged away
  by a strong score elsewhere.
- **Result**: Met / Not Met. If a target has no `## Documentation`
  section to grade against, this gate doesn't apply to that target —
  never invent one to fill the gap.

---

## 🛠️ 3. Platform-Agnostic Technical Rubric
*This rubric evaluates the Technical Execution & Functional Demonstration phases from §1.*

| Evaluation Criteria | Below Expectations (1) | Developing (2) | Proficient (3) | Exemplary (4) |
| :--- | :--- | :--- | :--- | :--- |
| **Application of Trained Skills** | Project fails to use the core technologies, tools, or frameworks taught in the training. | Uses the correct tools but implementation is shallow, incomplete, or fundamentally flawed. | **Successfully implements the core concepts, languages, or platform tools taught during the training program.** | Goes beyond the baseline training material, independently implementing advanced features or optimizations. |
| **Functionality & Stability** | The system/application does not run, breaks completely, or crashes immediately during the live test. | Major features are broken, but the basic skeleton or secondary components of the project function. | **All core requirements and features run successfully with minimal to no errors during the live demo.** | Flawless performance. System handles edge cases gracefully and includes robust error logging or validation. |
| **Technical Best Practices** | Poor structure. Code/configurations are messy, un-commented, or systems lack basic security controls. | Functional, but ignores industry standards (e.g., hardcoded secrets, poor naming, zero modularity). | **Follows clean syntax/system standards taught in class. Well-organized, secure, and documented.** | Production-ready quality. Exceptionally clean, optimized, highly secure, modular, and scalable design. |
| **Troubleshooting & Defense** | Cannot explain how the system works or how to fix basic errors when prompted. | Struggles to explain technical hitches; relies heavily on pre-configured scripts without understanding them. | **Confidently explains technical choices and can pinpoint where a bug is located if something breaks.** | Demonstrates deep diagnostic skills. Easily explains architectural trade-offs between different technical solutions. |

---

## 💡 4. Step-by-Step Facilitator Grading Process

### 🔍 Step 1: Pre-Demo Code/System Review
Do not try to grade technical execution entirely during the live presentation. 
* Review their repository, codebase, or cloud infrastructure configurations **before** the live session.
* Check the commit history to ensure incremental work and verify authenticity.
* Look for hardcoded credentials, code duplication, and documentation completeness.

### ⚡ Step 2: The Live "Destructive" Test
During the live demonstration, do not just let the student run a perfectly rehearsed script.
* Ask them to change a variable, input unexpected data, or trigger an intentional error state.
* Watching how their system handles a live anomaly tells you instantly if they genuinely built it or simply copied a template.

### 🗣️ Step 3: The "Why" Q&A Focus
Dedicate the final 5 minutes of their presentation to architectural defense questions. Use prompts like:
* *"Why did you choose this specific library/tool over the alternatives available?"*
* *"If this system had to scale up by a factor of 100 tomorrow, where is your bottleneck and what would break first?"*
* *"What was the most difficult bug you encountered, and how did you diagnose and resolve it?"*
