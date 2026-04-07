# Technical Preferences

<!-- Populated by /setup-engine. Updated as the user makes decisions throughout development. -->
<!-- All agents reference this file for project-specific standards and conventions. -->

## Engine & Language

- **Engine**: Unity 6.3 LTS (6000.3)
- **Language**: C# (primary)
- **Build System**: Unity Build Pipeline
- **Asset Pipeline**: Unity Asset Import Pipeline + Addressables

## Naming Conventions

- **Classes**: PascalCase (e.g., `PlayerController`)
- **Public fields/properties**: PascalCase (e.g., `MoveSpeed`)
- **Private fields**: _camelCase (e.g., `_moveSpeed`)
- **Methods**: PascalCase (e.g., `TakeDamage()`)
- **Files**: PascalCase matching class (e.g., `PlayerController.cs`)
- **Constants**: PascalCase or UPPER_SNAKE_CASE

## Performance Budgets

- **Target Framerate**: 60fps (16.6ms frame budget)
- **Frame Budget**: 12ms for game logic, 4ms for rendering
- **Draw Calls**: < 2000 per frame (use SRP Batcher and GPU Instancing)
- **Memory Ceiling**: 4GB total, < 2GB for game assets

## Testing

- **Framework**: NUnit (Unity Test Framework)
- **Minimum Coverage**: Core systems (player controller, LOS, damage)
- **Required Tests**: Balance formulas, gameplay systems, save/load

## Forbidden Patterns

<!-- Add patterns that should never appear in this project's codebase -->
- [None configured yet — add as architectural decisions are made]

## Allowed Libraries / Addons

<!-- Add approved third-party dependencies here -->
- [None configured yet — add as dependencies are approved]

## Architecture Decisions Log

<!-- Quick reference linking to full ADRs in docs/architecture/ -->
- [No ADRs yet — use /architecture-decision to create one]
