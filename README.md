# Hermes Agent Skills

Production skills for Hermes Agent (Nous Research), the self-improving autonomous AI agent framework.

## Skills (18)

| Skill | Description |
|-------|-------------|
| [agent-audit](skills/agent-audit/) | AI agent security scanner — OWASP Agentic Top 10 |
| [agent-tools](skills/agent-tools/) | Agent tool usage patterns and best practices |
| [agentic-patterns](skills/agentic-patterns/) | Reusable agentic software design patterns |
| [autoreason](skills/autoreason/) | Autonomous reasoning and intent routing |
| [blacksmith](skills/blacksmith/) | Security tool orchestration |
| [evolution](skills/evolution/) | Skill evolution — extract patterns → new skills |
| [find-skills](skills/find-skills/) | Skill discovery and gap analysis |
| [focused-fix](skills/focused-fix/) | Concentrated debugging for complex failures |
| [observability-designer](skills/observability-designer/) | Design observability for agent systems |
| [openspace-evolution](skills/openspace-evolution/) | Open space技术在AI工作流中的应用 |
| [price-action](skills/price-action/) | Crypto price action trading patterns |
| [production-safety](skills/production-safety/) | Guardrails and circuit breakers for agents |
| [revenue-intel](skills/revenue-intel/) | Revenue opportunity discovery |
| [security-recon](skills/security-recon/) | Security reconnaissance automation |
| [skill-tester](skills/skill-tester/) | Automated skill quality testing |
| [system-health](skills/system-health/) | Multi-service health monitoring |
| [zenda-manager](skills/zenda-manager/) | Zenda MCP server management |
| [zenops-ops](skills/zenops-ops/) | ZenOps agent operations patterns |

## Index

See [`index.json`](index.json) for full machine-readable skill index.

## CI

Every push validates:
- All `SKILL.md` files have required frontmatter (`name`, `description`)
- `index.json` is in sync with `skills/` directory
- Skill count badge generation

## Usage

Skills are loaded by Hermes Agent from this repo. See the [Hermes Agent Orange Book](https://github.com/alchaincyf/hermes-agent-orange-book) for full documentation.

## License

CC BY-NC-SA 4.0 (inherited from Hermes Agent ecosystem)
