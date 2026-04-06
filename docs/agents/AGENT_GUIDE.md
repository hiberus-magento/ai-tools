# Agent Development Guide

Guide for understanding, using, and extending Claude Code agents in this repository.

## What Are Agents?

Agents are specialized AI assistants that handle specific types of tasks. Each agent:
- Runs in its own context window
- Has a custom system prompt
- Has specific tool access
- Works independently and returns results

Agents are defined as Markdown files with YAML frontmatter.

## Agent Structure

```markdown
---
name: agent-name
description: Clear description of when Claude should use this agent. Include "Use proactively" for automatic delegation.
tools: Read, Grep, Glob, Bash
model: opus
---

System prompt content here...
Agent instructions, workflow, output format, etc.
```

### Frontmatter Fields

- **name** (required): Unique identifier (lowercase, hyphens)
- **description** (required): When Claude should delegate to this agent
- **tools** (optional): Allowed tools (inherits all if omitted)
- **model** (optional): `opus`, `sonnet`, `haiku`, or `inherit` (default: inherit)
- **permissionMode** (optional): Permission handling mode
- **maxTurns** (optional): Max agentic turns before stopping
- **skills** (optional): Skills to preload
- **mcpServers** (optional): MCP servers available to this agent
- **memory** (optional): Persistent memory scope (`user`, `project`, `local`)
- **background** (optional): Always run in background (true/false)
- **color** (optional): Display color in UI

See [Claude Code subagent documentation](https://code.claude.com/docs/en/sub-agents) for complete reference.

## Agent Locations

Agents can be stored in multiple locations with different scopes:

| Location | Scope | Priority | Use Case |
|----------|-------|----------|----------|
| `.claude/agents/` | Project | High | Project-specific agents (check into git) |
| `~/.claude/agents/` | User | Medium | Personal agents for all projects |
| Plugin `agents/` | Where enabled | Low | Distributed via plugins |

**This repository:** Agents in `agents/*.md` are designed to be copied to `.claude/agents/` in your project or `~/.claude/agents/` for personal use.

## Creating Effective Agents

### 1. Write a Clear Description

The description determines when Claude delegates. Be specific:

❌ **Bad:** "Analyzes requirements"  
✅ **Good:** "Expert Adobe Commerce RFP/RFQ analyzer. Use proactively when ingesting any new RFP, tender document, or project brief to extract structured requirements."

Include "Use proactively" to encourage automatic delegation.

### 2. Define a Clear Workflow

Structure your agent prompt with clear steps:

```markdown
# Execution Workflow

When invoked:

1. **Load inputs**
   - Read file X
   - Understand context Y

2. **Process data**
   - Extract information
   - Classify items
   - Validate logic

3. **Generate output**
   - Follow format exactly
   - Maintain traceability
   - No hallucinations
```

### 3. Specify Output Format Exactly

Be prescriptive about output structure:

```markdown
# Output Format

Generate `output_filename.md` with these sections:

## 1. Section Name
- Field 1: [Format]
- Field 2: [Format]

## 2. Another Section
Use this exact structure:
[Show example]
```

### 4. Make Agents Self-Contained

Agents should be **portable** and work in any project:

❌ **Bad:** "See project-context.md for field definitions"  
✅ **Good:** Include all taxonomies, formats, and structures in the agent definition

### 5. Restrict Tools Appropriately

- **Read-only agents** (reviewers, analyzers): `Read, Grep, Glob, Bash`
- **Generators** (writers, estimators): Add `Write, Edit`
- **Avoid over-permissioning**: Only grant tools the agent needs

### 6. Choose the Right Model

- **opus**: Complex reasoning, architecture decisions, detailed analysis
- **sonnet**: Balanced capability/speed for most development tasks
- **haiku**: Fast, simple tasks (not recommended for complex workflows)
- **inherit**: Use main conversation's model (default)

### 7. Include Quality Checks

Add validation sections:

```markdown
# Quality Bar
- No hallucinated features
- No invented systems
- Every item must have traceability
- Must be useful to downstream consumers
```

## Agent Patterns

### Pattern 1: Analyzer (Read-Only)

**Purpose:** Extract information, identify patterns, flag issues

```yaml
tools: Read, Grep, Glob, Bash
model: opus
```

**Examples:** rfp-analyzer, code-reviewer

### Pattern 2: Generator (Read-Write)

**Purpose:** Create new documents, transform data

```yaml
tools: Read, Grep, Glob, Bash, Write, Edit
model: opus
```

**Examples:** story-generator, effort-estimator

### Pattern 3: Pipeline Agent

**Purpose:** Part of a sequential workflow

- Input: Well-defined from previous agent
- Output: Well-defined for next agent
- Traceability: Maintains IDs from source

**Examples:** RFP analysis pipeline agents

### Pattern 4: Domain Specialist

**Purpose:** Deep expertise in specific technology

```yaml
model: sonnet
skills: [domain-specific-skill]
```

**Examples:** hyva-theme-agent, acs-storefront-agent (future)

## Testing Agents

### Manual Testing

```
# Invoke explicitly
Use the [agent-name] agent to [task]

# Or @-mention
@agent-[name] [task]
```

### Validation Checklist

- [ ] Agent produces expected output structure
- [ ] All sections are present and complete
- [ ] No hallucinated information
- [ ] Traceability maintained from inputs
- [ ] Format is machine-readable (for downstream agents)
- [ ] Quality checks pass

### Iterative Refinement

1. Run agent on sample input
2. Review output quality
3. Identify issues (missing sections, wrong format, hallucinations)
4. Update agent prompt
5. Test again

## Debugging Agents

### Agent Not Found

Check agent file location:
```bash
# List project agents
ls .claude/agents/

# List user agents
ls ~/.claude/agents/
```

Restart Claude Code session to load new agents.

### Agent Produces Wrong Output

- Review "Output Format" section in agent definition
- Check if input format changed
- Verify agent has correct tools
- Test with simpler input first

### Agent Hallucinates

Add stronger constraints:
```markdown
# Critical Rules
- Extract ONLY from source documents
- Never invent systems, integrations, or requirements
- If information is missing, flag it as "Unknown" or add to "Open Questions"
- Every statement must have source traceability
```

### Agent Output Not Consumable by Next Agent

Standardize interfaces between agents:
- Define exact field names
- Use consistent IDs (REQ-XXX-NNN)
- Include metadata for validation

## Extending the RFP Pipeline

### Add a New Agent to Pipeline

Example: Add a "risk-analyzer" between requirements and estimation:

1. **Create agent file**
   ```bash
   # .claude/agents/risk-analyzer.md
   ```

2. **Define inputs/outputs clearly**
   - Input: `01_requirements_analysis.md`
   - Output: `01b_risk_analysis.md`

3. **Update pipeline documentation**
   - Add to workflow diagram
   - Update `docs/agents/rfp-analysis-pipeline.md`

4. **Test integration**
   - Run full pipeline with new agent
   - Verify downstream agents still work

### Customize Existing Agents

**Scenario:** Your company uses different domain taxonomy

1. Copy agent to project: `.claude/agents/rfp-analyzer.md`
2. Edit "Domain Classification" section
3. Update downstream agents to use same taxonomy
4. Document changes in project README

## Agent Composition Patterns

### Sequential Pipeline
```
Agent A → output → Agent B → output → Agent C
```
Each agent processes previous output.

### Parallel + Consolidate
```
Agent A → output A ↘
Agent B → output B → Agent D (consolidator)
Agent C → output C ↗
```
Multiple agents work in parallel, one consolidates.

### Iterative Refinement
```
Agent A → output → human review → Agent A (refined) → output
```
Human reviews and refines before next step.

## Best Practices Summary

1. ✅ **Portable agents**: No external dependencies
2. ✅ **Clear descriptions**: Claude knows when to use them
3. ✅ **Structured workflows**: Step-by-step execution
4. ✅ **Exact output formats**: Prescriptive structure
5. ✅ **Minimal tool access**: Only what's needed
6. ✅ **Quality validation**: Built-in checks
7. ✅ **Traceability**: Maintain source references
8. ✅ **Self-documenting**: Explain purpose and usage

## Resources

- [Claude Code Subagent Documentation](https://code.claude.com/docs/en/sub-agents)
- [Example Agents Repository](https://github.com/affaan-m/everything-claude-code)
- [RFP Analysis Pipeline](rfp-analysis-pipeline.md)
- [Output Templates](templates/)

## Contributing

When contributing new agents:

1. Follow naming convention: `[prefix]-[description].md`
2. Include clear description with "Use proactively"
3. Define structured workflow
4. Specify exact output format
5. Make agent self-contained (no external references)
6. Test with real inputs
7. Document in appropriate workflow guide
8. Add to README.md agent table
