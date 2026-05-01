# agents-skills

A curated repository of agent skills for .NET engineering workflows.

This project organizes reusable skills into plugins, each focused on a specific area such as architecture design and testing practices. The goal is to provide high-signal, practical instructions that help coding agents produce better results on real tasks.

## Skills Scope

The repository currently focuses on:

- **.NET architecture guidance** for framework and domain design decisions
- **.NET testing guidance** for xUnit project structure and test authoring practices
- **Validation-ready skill authoring** with scenario-driven evaluation files under `tests/`

Each skill is versioned as documentation (`SKILL.md`) and evaluated through dedicated scenarios to ensure it adds measurable value.

## Plugins and Skills

| Plugin | Scope | Skills |
|---|---|---|
| `dotnet-arch` | Architecture and solution-level design for .NET libraries and applications | `custom-framework-arch` - organize multi-library framework structure and packaging<br>`entity-mgmt-arch` - design DDD entity lifecycle management with `Deveel.Repository` patterns<br>`xunit-test-arch` - structure xUnit test projects, layout, and build conventions |
| `dotnet-tests` | xUnit test coding and organization conventions | `xunit-test-organization` - write and organize xUnit tests with naming, fixtures, traits, and Bogus data patterns |

## Repository Layout

```text
plugins/                 # Plugin manifests and skill definitions
  <plugin>/
    plugin.json
    skills/
tests/                   # Skill evaluation scenarios (eval.yaml)
validation/              # Local skill-validator binaries/docs for validation workflows
marketplace.json         # Discovery index of available plugins
```

## How to Use

- Browse available plugins under `plugins/`.
- Open a skill's `SKILL.md` to understand when and how it should be applied.
- Use corresponding eval scenarios under `tests/cases/<plugin>/<skill>/eval.yaml` to validate skill impact.
- Use the validator tooling under `validation/` for static checks and A/B evaluation workflows.

For validator command details, see `validation/README.md`.

## Contributing

Contributions are welcome, whether you are improving existing skills or adding new ones.

### Ways to Contribute

- Add a new skill to an existing plugin
- Improve clarity, examples, or decision rules in a `SKILL.md`
- Add or refine evaluation scenarios in `tests/cases/.../eval.yaml`
- Expand plugin metadata and marketplace discoverability

### Contribution Flow

1. Fork and create a branch for your change.
2. Update or add skill content under `plugins/<plugin>/skills/<skill>/SKILL.md`.
3. Add or update evaluation scenarios under `tests/cases/<plugin>/<skill>/eval.yaml`.
4. Run validation checks using the tooling in `validation/`.
5. Open a pull request with:
   - what problem the change addresses,
   - what behavior improves,
   - and how you validated it.

If you are proposing a brand new plugin, include `plugin.json`, skill directories, and matching test cases from the start.

## License

This repository is licensed under the MIT License. See `LICENSE` for details.
