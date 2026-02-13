# Contributing to Evolynt Blueprints

This repository is a **read-only mirror** automatically synced from the main Evolynt project. The examples, schemas, and documentation here are the canonical source used by the Evolynt docs site and end-to-end test suite.

## How This Repo Is Updated

A GitHub Actions workflow in the main Evolynt repository automatically syncs changes to this public repo whenever Blueprint examples or JSON schemas are modified. Direct commits to this repository will be overwritten by the next sync.

## Reporting Issues

If you find an error in an example Blueprint or JSON schema:

1. **Open an issue** in this repository describing the problem.
2. Include which file is affected and what you expected vs. what you found.

## Requesting New Examples

If you'd like to see a Blueprint example for a specific business vertical or use case, open an issue with:

1. The type of business (e.g., "yoga studio", "SaaS onboarding").
2. What resources you'd expect (e.g., course, community, coaching, funnel).
3. Price points and payment structure.

## Validating Your Own Blueprints

You can validate your Blueprint YAML against the JSON schemas in the `schemas/` directory. Each schema corresponds to a single resource type.

## Full Documentation

For the complete Blueprint reference, visit [docs.evolynt.com](https://docs.evolynt.com).

- [Blueprint Introduction](https://docs.evolynt.com/docs/blueprints/introduction)
- [Getting Started](https://docs.evolynt.com/docs/blueprints/getting-started)
- [Full Reference](https://docs.evolynt.com/docs/blueprints/reference)
- [All Examples](https://docs.evolynt.com/docs/blueprints/examples)

## License

This repository is licensed under the MIT License. See [LICENSE](./LICENSE) for details.
