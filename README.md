# Edgy Stonks — Monorepo

Edgy Stonks is a spec-first virtual financial casino and stock-market simulation platform. The product combines high-dopamine casino interactions with real-world market data, while the economics remain isolated inside a closed-loop virtual wagering system.

---

## Documentation Map

| File | Responsibility |
| --- | --- |
| `README.md` | High-level overview for stakeholders and new readers. |
| `docs/CONTRIBUTING.md` | Developer and AI-agent onboarding, workflow, and validation gates. |
| `docs/CONSTITUTION.md` | Immutable architectural boundaries and systemic constraints. |
| `docs/SPECS.md` | Technical index, global infrastructure decisions, and deployment layout. |
| `docs/specs/*.md` | Domain-isolated implementation specifications. |

---

## Product Scope

- Users place virtual `LONG` and `SHORT` bets against market movements.
- Internal casino currency is tracked separately from the withdrawable mETH ledger.
- Market data is treated as an asynchronous hydration signal, not a blocking user-path dependency.
- The platform is designed around Cloudflare's edge runtime model and the `$0 idle cost` constraint.

---

## Engineering Model

The repository follows a modular architecture blueprint:

1. `docs/CONSTITUTION.md` defines the non-negotiable constraints.
2. `docs/SPECS.md` indexes the technical architecture and global decisions.
3. Domain specs define isolated implementation details.
4. `docs/CONTRIBUTING.md` defines how changes are made and validated.

No feature branch or AI-generated changeset should modify implementation code until the relevant specification is updated and internally consistent.

---

## License

This project is licensed under the terms specified in the `LICENSE` file.
