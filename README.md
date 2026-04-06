# Hiberus AI Tools for Magento 2

AI-powered Skills and Agents for Magento 2 development. This collection extends AI coding assistants (Claude, Codex, OpenCode, Copilot, Cursor, Gemini, etc.) with expert knowledge for Magento 2, PHP, and specialized frameworks. From backend modules and core architecture to modern frontends like Hyvä Themes and Adobe Commerce Storefront (ACS), these skills and agents accelerate the creation of production-ready components, logic, and CMS tools.

---

## 🚀 Integration with Hiberus Dockergento

This repository is designed to work seamlessly with **[Hiberus Dockergento](https://github.com/hiberus-magento/hiberus-dockergento)** — our open-source CLI tool for managing Docker-based development environments for Magento 2 projects.

Dockergento already implements logic to **initialize and integrate AI Tools** into your Magento 2 development workflow automatically. If you are working with Magento 2 and want an agile, productive, and well-structured development environment, give Dockergento a try!

---

## 📁 Repository Structure

```
AI-Tools/
├── skills/     # Reusable skills for AI coding assistants
└── agents/     # Custom agents for specific workflows
```

### Skills

Skills are instruction files that extend AI assistants with specialized knowledge for specific tasks in Magento 2 development. They are stored in the `skills/` directory, where each subdirectory represents one skill.

### Agents

Agents are custom AI agents designed to handle complete workflows or complex multi-step tasks. They are stored in the `agents/` directory.

---

## 🏷️ Naming Conventions

### Skills (`skills/`)

Each skill lives in its own subdirectory inside `skills/`. The **directory name is the skill name** and must follow this convention:

```
<prefix>-<short-description>
```

The **prefix** identifies the scope or technology of the skill:

| Prefix | Scope / Technology |
|--------|--------------------|
| `magento-` | General Magento 2 backend (modules, plugins, observers, etc.) |
| `php-` | Generic PHP patterns used in Magento 2 context |
| `hyva-` | Hyvä Theme frontend (Alpine.js, Tailwind CSS, PHTML) |
| `acs-` | Adobe Commerce Storefront / Edge Delivery Services |
| `db-` | Database operations, schema, patches |
| `api-` | REST / GraphQL API development |
| `test-` | Testing — unit, integration, functional |
| `devops-` | CI/CD, Docker, deployment-related tasks |

**Examples:**

- `magento-create-module` — scaffold a new Magento 2 module
- `magento-create-plugin` — create a Magento 2 plugin (interceptor)
- `hyva-alpine-component` — write an Alpine.js component for a Hyvä theme
- `acs-create-ui-component` — create a UI component for Adobe Commerce Storefront
- `php-value-object` — implement a PHP value object pattern
- `test-unit` — write PHPUnit unit tests for Magento 2

### Agents (`agents/`)

Each agent lives in its own subdirectory inside `agents/`. The **directory name is the agent name** and follows a similar convention:

```
<prefix>-<short-description>-agent
```

**Examples:**

- `magento-module-agent` — agent for scaffolding complete Magento 2 modules end-to-end
- `hyva-theme-agent` — agent for building or customizing Hyvä themes
- `acs-storefront-agent` — agent focused on Adobe Commerce Storefront development

---

## 📚 Available Skills

> Skills will be added progressively. The table below will be kept up to date as new skills are contributed.

| Skill | Prefix | Description |
|-------|--------|-------------|
| _(coming soon)_ | — | — |

---

## 🤖 Available Agents

> Custom agents will be added progressively. The table below will be kept up to date as new agents are contributed.

### RFP Analysis Pipeline

Complete workflow for analyzing Adobe Commerce RFPs and generating effort estimates. [See full documentation](docs/agents/rfp-analysis-pipeline.md).

| Agent | Description | Input | Output |
|-------|-------------|-------|--------|
| **rfp-analyzer** | Extract and structure requirements from RFP documents | RFP docs (.pdf, .docx, .md) | `01_requirements_analysis.md` |
| **story-generator** | Convert requirements into INVEST-compliant user stories with BDD acceptance criteria | `01_requirements_analysis.md` | `03_technical_backlog.md` |
| **effort-item-extractor** | Decompose requirements/stories into atomic estimable work items | Requirements or stories | `02_estimable_items.md` |
| **effort-estimator** | Produce detailed hour estimates comparing human vs AI-assisted development | `02_estimable_items.md` | `04_detailed_estimation.md` |

**Quick Start:**
```bash
# 1. Analyze RFP
Use the rfp-analyzer agent to analyze the RFP documents in ./rfp-documents/

# 2. Generate user stories (optional)
Use the story-generator agent to create user stories from 01_requirements_analysis.md

# 3. Extract estimable items
Use the effort-item-extractor agent to extract estimable items from 01_requirements_analysis.md

# 4. Generate estimates
Use the effort-estimator agent to produce detailed estimates from 02_estimable_items.md
```

For complete workflow documentation, examples, and best practices, see [RFP Analysis Pipeline Guide](docs/agents/rfp-analysis-pipeline.md).

---

## 🤝 Contributing

Contributions are welcome! Whether you want to add a new skill, improve an existing one, or report an issue, feel free to open a pull request or an issue.

Please follow the [naming conventions](#naming-conventions) described above when adding new skills or agents.

---

## License

Licensed under the [OSL-3.0](LICENSE).

Copyright (c) Hiberus Tecnología S.A. [https://www.hiberus.com](https://www.hiberus.com). All rights reserved.
