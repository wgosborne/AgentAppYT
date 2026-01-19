# Claude Agents: Project Workflow Demonstration

This repository demonstrates how Claude AI agents work together in a structured handoff pattern to take a project from conception to completion. It's designed as a learning resource for understanding how AI agents can collaboratively handle software engineering workflows.

## What's Inside

This project showcases **8 specialized agents** that each have a specific role in the development lifecycle:

- 🔍 **Researcher** - Explores requirements and asks clarifying questions
- 🏗️ **Architect** - Designs system architecture and technical decisions
- 💻 **Implementer** - Builds the actual code based on architecture
- 🧪 **Tester** - Designs and runs comprehensive test suites
- 🎨 **Designer** - Refines UI/UX and styling
- 📋 **Code Reviewer** - Audits code quality and best practices
- 🚀 **Deployer** - Handles deployment and release management
- 🎯 **Project Manager** - Synthesizes all handoffs into a project snapshot

## The Agent Workflow

Each agent specializes in one phase and hands off structured documentation to the next agent:

```
Researcher (01-requirements.md)
    ↓
Architect (02-architecture.md)
    ↓
Implementer (03-implementer.md)
    ↓
Tester (05-test.md)
    ↓
Designer (04-designer.md)
    ↓
Code Reviewer (06-review.md)
    ↓
Deployer (deployment notes)
    ↓
Project Manager (CLAUDE.md - final snapshot)
```

The `/Handoffs/` folder contains all the structured documentation that passes between agents, making the entire process transparent and replicable.

## How to Use These Agents

### Prerequisites

- **Claude Code CLI** - Latest version installed
- **Access to Claude** - Claude Opus or Sonnet model recommended for best results
- **Your Project** - A folder with your project code and this README

### Getting Started with a New Project

#### Step 1: Researcher - Explore Requirements

```bash
cd your-project-folder
```

In Claude Code, invoke:
```
/researcher
```

The Researcher agent will:
1. Ask you clarifying questions about your project vision
2. Explore the problem space and requirements
3. Generate `/Handoffs/01-requirements.md` with:
   - Problem statement
   - Functional requirements
   - Non-functional requirements (performance, scale, security)
   - Edge cases and risks
   - Open questions
   - Constraints

This is where you define **WHAT** you're building.

---

#### Step 2: Architect - Design the System

Once the Researcher finishes, invoke:
```
/architect
```

The Architect agent will:
1. Read the requirements from `01-requirements.md`
2. Discuss technology stack choices and trade-offs
3. Design the API, data model, and system components
4. Generate `/Handoffs/02-architecture.md` with:
   - Technology stack decisions (with rationale)
   - Component structure
   - API endpoint design
   - Data model and schema
   - Error handling strategy
   - Validation rules
   - Architecture decision records

This is where you define **HOW** you're building it.

---

#### Step 3: Implementer - Build the Code

Once the Architect finishes, invoke:
```
/implementer
```

The Implementer agent will:
1. Read both requirements and architecture
2. Break down the work into implementation phases
3. Build the actual code and project structure
4. Generate `/Handoffs/03-implementer.md` with:
   - Implementation phases and checklist
   - Progress tracking
   - Code organization
   - Build instructions

This is where the code gets **WRITTEN**.

---

#### Step 4: Tester - Validate Quality

Once the Implementer finishes, invoke:
```
/tester
```

The Tester agent will:
1. Read requirements, architecture, and implementation
2. Design comprehensive test cases (unit, integration, e2e)
3. Run tests and report results
4. Generate `/Handoffs/05-test.md` with:
   - Test plan and strategy
   - All test cases (organized by type)
   - Test execution results
   - Coverage analysis
   - Known issues and blockers

This is where code **QUALITY** is verified.

---

#### Step 5: Designer - Polish UI/UX

Once the Tester finishes, invoke:
```
/designer
```

The Designer agent will:
1. Run the development server
2. Review current UI/UX
3. Refine styling and components
4. Generate `/Handoffs/04-designer.md` with:
   - Design decisions and rationale
   - Component updates
   - Styling changes
   - Accessibility improvements
   - Mobile responsiveness notes

This is where the interface gets **REFINED**.

---

#### Step 6: Code Reviewer - Audit the Codebase

Once the Designer finishes, invoke:
```
/reviewer
```

The Code Reviewer agent will:
1. Read all requirements and code
2. Audit code quality, patterns, and best practices
3. Check for security vulnerabilities
4. Generate `/Handoffs/06-review.md` with:
   - Critical issues found (must fix)
   - Minor issues (nice to fix)
   - Future improvement suggestions
   - Best practice recommendations
   - Security assessment

This is where code **CORRECTNESS** is reviewed.

---

#### Step 7: Deployer - Release the Project

Once code review is complete, invoke:
```
/deployer
```

The Deployer agent will:
1. Prepare deployment artifacts
2. Create release notes
3. Execute deployment procedures
4. Generate deployment documentation with:
   - Deployment checklist
   - Environment configuration
   - Release notes
   - Rollback procedures
   - Health check instructions

This is where the project gets **DEPLOYED**.

---

#### Step 8: Project Manager - Synthesize Everything

At any time (or after all agents complete), invoke:
```
/projectmanager
```

The Project Manager agent will:
1. Read all handoff documents and code
2. Create `CLAUDE.md` - a concise project snapshot (<300 lines) containing:
   - Project overview
   - Tech stack summary
   - Architecture highlights
   - Key entities and workflows
   - API endpoints
   - Key decisions and rationale
   - Current status
   - How to run the project
   - Deployment notes

This becomes your **PROJECT BIBLE** - a single source of truth for anyone joining the project.

---

## The Handoffs Folder Structure

As you run agents, expect this folder to grow:

```
/Handoffs/
├── 01-requirements.md      ← Created by Researcher
├── 02-architecture.md      ← Created by Architect
├── 03-implementer.md       ← Created by Implementer
├── 04-designer.md          ← Created by Designer
├── 05-test.md              ← Created by Tester
└── 06-review.md            ← Created by Code Reviewer
```

Plus at the project root:
```
CLAUDE.md                    ← Created by Project Manager (final synthesis)
```

Each file is designed to be:
- **Self-contained** - Contains all info needed for the next agent
- **Structured** - Consistent format across projects
- **Actionable** - Contains concrete decisions and outputs
- **Auditable** - Shows reasoning and trade-offs

---

## Example: The Beer Mile Betting App

This repository already contains a completed example: a lightweight web app for betting on beer mile performance.

**Handoffs created:**
- ✅ `/Handoffs/01-requirements.md` - User roles, prop types, betting workflow
- ✅ `/Handoffs/02-architecture.md` - Next.js, PostgreSQL, SSE for live updates
- ✅ `/Handoffs/03-implementer.md` - 7 implementation phases
- ✅ `/Handoffs/05-test.md` - 120+ comprehensive test cases
- ✅ `/CLAUDE.md` - Complete project snapshot

You can review these files to understand the handoff format and see how each agent builds on the previous one's work.

---

## Key Features of This Workflow

### 1. **Structured Handoffs**
Each agent produces a specific document that the next agent reads, ensuring continuity and clarity.

### 2. **Specialized Agents**
Each agent is optimized for its specific domain (requirements, architecture, testing, etc.) rather than trying to do everything.

### 3. **Clear Separation of Concerns**
- Researcher focuses on **WHAT**, not HOW
- Architect focuses on **HOW**, not WHAT
- Implementer focuses on **BUILDING**, not DESIGNING
- This prevents scope creep and improves quality

### 4. **Transparent Decision-Making**
Every major decision is documented with rationale and trade-offs, making it easy to understand project history.

### 5. **Replicable Process**
The entire workflow is documented and repeatable. New team members can understand the project by reading the handoffs.

---

## Tips for Best Results

### When Invoking Agents

1. **Be clear about your project** - The Researcher will ask questions, but starting with a clear vision helps
2. **Answer questions honestly** - Agents ask for a reason; vague answers lead to vague designs
3. **Review agent outputs** - Check each handoff document before proceeding to the next agent
4. **Ask follow-up questions** - Agents are conversational; clarify if anything isn't right
5. **Use `/resume`** - If interrupted, you can resume an agent with its ID

### When Reviewing Handoffs

- **01-requirements.md**: Does it match your vision?
- **02-architecture.md**: Do the technology choices make sense?
- **03-implementer.md**: Is the implementation plan realistic?
- **05-test.md**: Does test coverage match your quality standards?
- **04-designer.md**: Does the UI match your brand?
- **06-review.md**: Are the findings reasonable?

### Project Manager Tips

- Run `/projectmanager` after major milestones
- It auto-generates from existing handoffs, so it updates as you progress
- Use `CLAUDE.md` as your "project at a glance" for quick reference

---

## Advanced Usage

### Optional Agents

Some agents are optional depending on your project:

- **Fun-Guy** (before Researcher) - If you want creative brainstorming before diving into requirements
- **Project Manager** - Can be invoked anytime to get a project snapshot

### Resuming Work

If an agent gets interrupted, find its agent ID and resume:

```
/task resume=<agent-id>
```

The agent will continue from where it left off with full context preserved.

### Multiple Iterations

This workflow doesn't have to be linear:

- If the Architect questions something fundamental, go back to the Researcher
- If the Implementer discovers a design issue, the Architect can iterate
- This is normal and healthy

---

## File Navigation

- **Agent Prompts** - The instructions each agent follows
- **Handoffs** - All structured documentation (read these to understand the project)
- **docs** - Additional documentation
- **CLAUDE.md** - Project summary (auto-generated by Project Manager)

---

## Example Agent Invocations

Here's what a typical session might look like:

```
You: /researcher
Researcher: Hi! What are we building?
You: A betting app for beer mile races
Researcher: [asks 5-10 clarifying questions]
Researcher: [writes 01-requirements.md and calls /architect]
---

Architect: I've read the requirements. Here's my tech stack thinking...
Architect: [discusses architecture decisions]
Architect: [writes 02-architecture.md and calls /implementer]
---

Implementer: I've reviewed the architecture. Here's my implementation plan...
Implementer: [builds code and writes 03-implementer.md]
---

[Continue through Tester, Designer, Code Reviewer, Deployer, Project Manager]
```

---

## Learning Resources

This repository demonstrates:
- **Agent collaboration** - How to design systems where agents hand off work
- **Structured documentation** - Creating reusable templates for complex workflows
- **Software engineering practices** - Requirements, design, implementation, testing, review, deployment

Watch the YouTube video accompanying this repo to see these agents in action!

---

## Contributing

This is a demonstration project. If you find issues or have suggestions for the agent prompts, contributions are welcome.

---

## Questions?

- **About Claude Code?** Run `/help` in the CLI
- **About this project?** Check the CLAUDE.md file for a complete project snapshot
- **About the agents?** Review the prompts in `/Agent Prompts/` folder

Happy building! 🚀
