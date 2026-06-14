```markdown
# claude-desktop-buddy Development Patterns

> Auto-generated skill from repository analysis

## Overview
This skill teaches you the core development patterns and conventions used in the `claude-desktop-buddy` TypeScript codebase. You'll learn how to structure files, write imports/exports, follow commit message standards, and organize tests. While no frameworks or automated workflows are detected, this guide provides best practices for contributing and maintaining consistency.

## Coding Conventions

### File Naming
- Use **camelCase** for file names.
  - Example: `mainWindow.ts`, `userSettings.ts`

### Import Style
- Use **relative imports** for modules within the project.
  - Example:
    ```typescript
    import { getUserConfig } from './userConfig';
    ```

### Export Style
- Use **named exports** for all modules.
  - Example:
    ```typescript
    // userConfig.ts
    export function getUserConfig() { ... }
    export const DEFAULT_CONFIG = { ... };
    ```

### Commit Messages
- Follow **Conventional Commits** with the `feat` prefix for new features.
  - Example:
    ```
    feat: add tray icon for quick access
    ```

## Workflows

### Adding a New Feature
**Trigger:** When implementing a new feature or functionality  
**Command:** `/add-feature`

1. Create a new file using camelCase naming.
2. Implement the feature using TypeScript.
3. Use relative imports to include dependencies.
4. Export functions or constants using named exports.
5. Write or update corresponding test files (`*.test.ts`).
6. Commit your changes with a message starting with `feat:`.
   - Example: `feat: implement auto-update functionality`

### Writing Tests
**Trigger:** When adding or updating code that requires testing  
**Command:** `/write-test`

1. Create a test file with the same name as the source file, ending with `.test.ts`.
   - Example: `userSettings.test.ts`
2. Write tests using the project's preferred (unknown) testing framework.
3. Use relative imports to bring in the module under test.
4. Run tests manually (since no workflow is detected).

## Testing Patterns

- Test files follow the `*.test.*` pattern.
  - Example: `mainWindow.test.ts`
- The specific testing framework is not detected; use standard TypeScript testing practices.
- Place test files alongside or near the files they cover.

## Commands
| Command      | Purpose                                            |
|--------------|----------------------------------------------------|
| /add-feature | Steps to add a new feature following conventions   |
| /write-test  | Steps to write and organize tests                  |
```
