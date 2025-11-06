# Tooling Capability


## Purpose

Development tooling including TypeScript/Bun configuration, testing, linting, and build scripts.

## Requirements

This capability provides development tooling and build infrastructure. All tools MUST support the monorepo structure, enforce consistent code quality standards, and enable rapid development iteration.

### Requirement: TypeScript Configuration
The system SHALL provide TypeScript configuration optimized for Bun runtime with strict type checking.

#### Scenario: Configure TypeScript compiler
- **GIVEN** tsconfig.json file
- **WHEN** compiling TypeScript
- **THEN** use ES2022 target, moduleResolution: bundler, strict: true

#### Scenario: Include type definitions
- **GIVEN** TypeScript project
- **WHEN** configuring types
- **THEN** include types for vitest, bun-types, @types/node

#### Scenario: Path aliases for imports
- **GIVEN** source files in src/ directory
- **WHEN** configuring TypeScript
- **THEN** define path aliases (`@/*` → `./src/*`) for cleaner imports

### Requirement: Testing Framework Setup
The system SHALL provide Vitest test framework with coverage reporting for unit, integration, and e2e tests.

#### Scenario: Configure Vitest
- **GIVEN** vitest.config.ts file
- **WHEN** running tests
- **THEN** use test environment matching target (node for backend, jsdom for frontend)

#### Scenario: Coverage reporting
- **GIVEN** test suite execution
- **WHEN** running with coverage flag
- **THEN** generate coverage report with @vitest/coverage-v8, enforce 80% minimum

#### Scenario: Separate test types
- **GIVEN** tests in tests/unit, tests/integration, tests/e2e
- **WHEN** running specific test suite
- **THEN** execute only matching tests with appropriate configuration

### Requirement: Linting and Formatting
The system SHALL provide Biome for code linting and formatting with consistent project rules.

#### Scenario: Configure Biome
- **GIVEN** biome.json configuration
- **WHEN** defining rules
- **THEN** enable recommended rules, configure indentation (2 spaces), line width (100)

#### Scenario: Lint on pre-commit
- **GIVEN** staged files for commit
- **WHEN** running pre-commit hook
- **THEN** lint staged files, fail commit if errors found

#### Scenario: Auto-format on save
- **GIVEN** editor integration with Biome
- **WHEN** saving file
- **THEN** auto-format using Biome rules

### Requirement: Build Scripts
The system SHALL provide npm/bun scripts for common development tasks (dev, build, test, lint, format).

#### Scenario: Run development server
- **GIVEN** package.json with `dev` script
- **WHEN** running `bun run dev`
- **THEN** start development server with hot reload

#### Scenario: Build production bundle
- **GIVEN** package.json with `build` script
- **WHEN** running `bun run build`
- **THEN** compile TypeScript, bundle with esbuild, output to dist/

#### Scenario: Run all tests
- **GIVEN** package.json with test scripts
- **WHEN** running `bun test`
- **THEN** execute all unit, integration, e2e tests in order

### Requirement: Package Management
The system SHALL use Bun package manager with workspace support for monorepo structure.

#### Scenario: Install dependencies
- **GIVEN** package.json with dependencies
- **WHEN** running `bun install`
- **THEN** install packages to node_modules, generate bun.lockb

#### Scenario: Workspace configuration
- **GIVEN** monorepo with multiple packages
- **WHEN** defining workspaces in package.json
- **THEN** support shared dependencies and workspace references

#### Scenario: Add new dependency
- **GIVEN** need for new package
- **WHEN** running `bun add <package>`
- **THEN** install package, update package.json and lockfile

### Requirement: EditorConfig and GitIgnore
The system SHALL provide .editorconfig for consistent editor settings and .gitignore for version control exclusions.

#### Scenario: EditorConfig settings
- **GIVEN** .editorconfig file
- **WHEN** opening file in editor
- **THEN** apply indent_style=space, indent_size=2, end_of_line=lf

#### Scenario: GitIgnore exclusions
- **GIVEN** .gitignore file
- **WHEN** staging files for commit
- **THEN** exclude node_modules/, dist/, .env, *.log, coverage/

### Requirement: CI/CD Configuration
The system SHALL provide GitHub Actions workflow for automated testing and validation on pull requests.

#### Scenario: Run tests on PR
- **GIVEN** pull request opened
- **WHEN** GitHub Actions workflow triggers
- **THEN** install deps, run lint, typecheck, unit tests, integration tests

#### Scenario: Fail PR on test failure
- **GIVEN** workflow execution with failing tests
- **WHEN** tests fail
- **THEN** mark workflow as failed, block PR merge

#### Scenario: Coverage reporting in PR
- **GIVEN** workflow with coverage step
- **WHEN** tests complete
- **THEN** post coverage report as PR comment
