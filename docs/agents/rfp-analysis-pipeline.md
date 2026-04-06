# Adobe Commerce RFP Analysis Pipeline

Complete automated workflow for analyzing RFPs, generating requirements, creating backlogs, and producing effort estimations for Adobe Commerce projects.

## Overview

This pipeline consists of 4 specialized Claude Code agents that work sequentially to transform an RFP document into a detailed, estimable project plan with human vs AI-assisted effort comparisons.

```
RFP Document(s)
     ↓
[rfp-analyzer] → 01_requirements_analysis.md
     ↓
[story-generator] → 03_technical_backlog.md
     ↓
[effort-item-extractor] → 02_estimable_items.md
     ↓
[effort-estimator] → 04_detailed_estimation.md
     ↓
Final Proposal Package
```

---

## The Agents

### 1. rfp-analyzer
**Purpose:** Extract and structure requirements from RFP documents

**Input:** RFP documents (.pdf, .docx, .md, etc.)

**Output:** `01_requirements_analysis.md`

**What it does:**
- Reads all RFP documents
- Extracts project context (platform, scope, integrations)
- Identifies functional requirements by domain
- Documents integration touchpoints
- Captures non-functional requirements
- Flags ambiguities, risks, and gaps
- Assigns complexity and estimability ratings

**When to use:** First step when you receive a new RFP/RFQ

---

### 2. story-generator
**Purpose:** Convert requirements into development-ready user stories

**Input:** `01_requirements_analysis.md`

**Output:** `03_technical_backlog.md`

**What it does:**
- Filters for items requiring custom development
- Generates INVEST-compliant user stories
- Writes BDD (Given/When/Then) acceptance criteria
- Provides Adobe Commerce technical guidance (modules, plugins, APIs, frontend approach)
- Maps dependencies between stories
- Suggests sprint groupings

**When to use:** After requirements analysis, when you need a development backlog

---

### 3. effort-item-extractor
**Purpose:** Decompose requirements/stories into atomic estimable items

**Input:** `01_requirements_analysis.md` OR `03_technical_backlog.md`

**Output:** `02_estimable_items.md`

**What it does:**
- Breaks requirements/stories into atomic work items
- Classifies by work type (CONFIG, BACKEND_CUSTOM, INTEGRATION, etc.)
- Identifies effort drivers (complexity, entities, screens, validations)
- Rates AI acceleration potential
- Groups items for estimation
- Flags dependencies

**When to use:** Before estimation, to ensure granular, estimable units

---

### 4. effort-estimator
**Purpose:** Produce detailed hour estimates comparing human vs AI-assisted development

**Input:** `02_estimable_items.md`

**Output:** `04_detailed_estimation.md`

**What it does:**
- Breaks items into subtasks
- Estimates human hours (baseline)
- Estimates AI-assisted hours (realistic savings)
- Calculates absolute and percentage savings
- Documents assumptions, risks, confidence
- Consolidates totals by domain, type, layer, role
- Provides AI optimization recommendations

**When to use:** Final step to generate proposal-ready estimates

---

## Quick Start

### Step 1: Analyze the RFP

```
Use the rfp-analyzer agent to analyze the RFP documents in ./rfp-documents/
```

Claude will invoke the `rfp-analyzer` agent, which will read all documents and generate `01_requirements_analysis.md`.

### Step 2: Generate User Stories (Optional)

If you want a detailed development backlog:

```
Use the story-generator agent to create user stories from 01_requirements_analysis.md
```

This generates `03_technical_backlog.md` with INVEST-compliant stories and acceptance criteria.

### Step 3: Extract Estimable Items

```
Use the effort-item-extractor agent to extract estimable items from 01_requirements_analysis.md
```

Or if you generated stories:

```
Use the effort-item-extractor agent to extract estimable items from 03_technical_backlog.md
```

This generates `02_estimable_items.md`.

### Step 4: Generate Estimates

```
Use the effort-estimator agent to produce detailed estimates from 02_estimable_items.md
```

This generates `04_detailed_estimation.md` with complete effort breakdown.

---

## Workflow Variations

### Fast Track (Skip Stories)
For quick estimates without detailed backlog:

```
RFP → rfp-analyzer → effort-item-extractor → effort-estimator
```

### Full Pipeline (With Stories)
For projects requiring detailed backlog + estimates:

```
RFP → rfp-analyzer → story-generator → effort-item-extractor → effort-estimator
```

### Backlog Only
For projects where you just need user stories:

```
RFP → rfp-analyzer → story-generator
```

---

## Best Practices

### 1. Provide Project Context

Before running any agent, provide context about the project:

```
This is an Adobe Commerce 2.4.7 project with Hyvä storefront.
Scope includes: backend, frontend, integrations, data migration.
Excluded: UX design, QA, training.
Key integrations: SAP (ERP), Akeneo (PIM), Stripe (PSP).
Multi-store: 3 countries (US, UK, DE), 3 languages, 3 currencies.
```

The agents use this context to improve accuracy.

### 2. Review and Refine

After each agent completes:
- Review the output
- Correct any misunderstandings
- Add clarifications
- Save the refined version as `01b_requirements_refined.md` if needed

The next agent can work from the refined version.

### 3. Iterate on Unclear Items

If agents flag items as "Not Ready" or "Low Confidence":
- Clarify with the client
- Run technical spikes
- Re-run the agent with updated information

### 4. Use Parallel Agents

For large RFPs, you can run agents in parallel on different sections:

```
Run rfp-analyzer on section 3.1-3.5 (Catalog requirements) in parallel
with rfp-analyzer on section 4.1-4.3 (Checkout requirements)
```

Then consolidate the outputs.

### 5. Validate Estimates

Compare agent estimates against:
- Historical project data
- Team velocity
- Industry benchmarks

Adjust contingency accordingly.

---

## Output Files Reference

| File | Generated By | Contains |
|------|--------------|----------|
| `01_requirements_analysis.md` | rfp-analyzer | Structured requirements, integrations, risks |
| `02_estimable_items.md` | effort-item-extractor | Atomic work items ready for estimation |
| `03_technical_backlog.md` | story-generator | User stories with acceptance criteria |
| `04_detailed_estimation.md` | effort-estimator | Hour estimates, human vs AI comparison |

---

## AI Acceleration Insights

The pipeline quantifies AI development impact:

**High AI Benefit (30-50% savings):**
- Boilerplate code generation (di.xml, registration.php)
- CRUD repository implementations
- GraphQL resolver scaffolding
- XML configuration files
- Test scaffolding
- Documentation drafting

**Medium AI Benefit (15-30% savings):**
- Business logic implementation
- Frontend components
- Integration mapping
- Data patches
- Admin interfaces

**Low AI Benefit (<15% savings):**
- Complex architecture decisions
- Performance optimization
- Security hardening
- Code review and validation
- Client communication

Use these insights to:
- Price competitively with AI-assisted hours
- Show ROI of AI investment to clients
- Allocate team resources effectively

---

## Troubleshooting

### Agent not found
Ensure agents are available in your Claude Code session. Agents from this repository should be in the `agents/` directory.

### Agent produces incomplete output
- Check if RFP documents are readable (not scanned PDFs without OCR)
- Provide more context about the project
- Break large RFPs into sections

### Estimates seem off
- Review assumptions in `04_detailed_estimation.md`
- Adjust for team experience level
- Compare with historical data
- Add contingency for uncertainty

### Dependencies unclear
- Review dependency maps in outputs
- Run story-generator to clarify relationships
- Create a spike for unclear technical dependencies

---

## Customizing the Pipeline

### Modify Agents
Agents are defined in `agents/*.md` files with YAML frontmatter.

Edit the agent file to:
- Adjust output format
- Change domain taxonomy
- Add custom validation rules
- Modify estimation approach

### Add Custom Agents
Create new agents for project-specific needs using `/agents` command in Claude Code.

### Extend Output Templates
Modify the "Output Format" section in each agent to match your proposal format.

---

## Integration with Proposal Tools

Export data from agent outputs to:

**Jira:**
- Import stories from `03_technical_backlog.md`
- Use story IDs for traceability

**Excel/Google Sheets:**
- Extract tables from `04_detailed_estimation.md`
- Create budget breakdowns

**Confluence:**
- Copy requirements from `01_requirements_analysis.md`
- Link to source RFP sections

**Proposal Software:**
- Use estimates from `04_detailed_estimation.md`
- Include AI savings as competitive advantage

---

## Next Steps

After completing the pipeline:

1. **Review with technical team**
   - Validate assumptions
   - Adjust complexity ratings
   - Add team-specific contingency

2. **Clarify with client**
   - Address "Open Questions" from requirements analysis
   - Confirm integration specifications
   - Validate non-functional requirements

3. **Refine estimates**
   - Re-run effort-estimator with clarifications
   - Adjust for team velocity
   - Add project management overhead

4. **Create proposal**
   - Use `04_detailed_estimation.md` as basis
   - Highlight AI-assisted savings
   - Structure by milestones
   - Include risks and mitigation

5. **Archive for future reference**
   - Store all outputs with RFP
   - Use for lessons learned
   - Refine agent prompts based on accuracy
