# AGENTS.md - Developer Guidelines

This file provides guidance for AI agents working in this repository.

## Project Overview

This is a TypeScript/JavaScript project using [bun](https://bun.sh) as the package manager and runtime. The main code resides in `.opencode/` directory.

## Build, Lint, and Test Commands

```bash
# Install dependencies
bun install

# Add a dependency
bun add <package>

# Build the project
bun run build

# Run in development mode
bun run dev

# Start the application
bun start

# Type checking
bun run typecheck

# Run linter
bun run lint

# Format code
bun run fmt

# Run all tests
bun test

# Run a single test file
bun test <path-to-test-file>

# Run tests matching a pattern
bun test --grep "<pattern>"

# Run tests in watch mode
bun test --watch
```

## Code Style Guidelines

### General Principles

- Write clean, readable, and maintainable code
- Keep functions small and focused (single responsibility)
- Use meaningful variable and function names
- Avoid magic numbers - use constants instead

### TypeScript Usage

- **Always use TypeScript** for new code - no plain JavaScript
- Enable `strict: true` in tsconfig.json
- Use explicit types for function parameters and return types
- Prefer interfaces over types for object shapes
- Use `unknown` instead of `any` - avoid `any` at all costs
- Leverage generics for reusable components

### Naming Conventions

- **Variables/functions**: camelCase (`getUserData`, `isActive`)
- **Classes/interfaces/types**: PascalCase (`UserService`, `ApiResponse`)
- **Constants**: SCREAMING_SNAKE_CASE (`MAX_RETRY_COUNT`)
- **Files**: kebab-case (`user-service.ts`)
- **Booleans**: Use `is`, `has`, `should` prefixes (`isActive`, `hasPermission`)

### Imports

- Use absolute imports when possible (configure `paths` in tsconfig.json)
- Group imports in order: External libraries, Internal modules, Type imports
- Use named exports instead of default exports

```typescript
// Preferred
import { useState, useEffect } from 'react';
import type { FC } from 'react';
import { UserService } from '@/services/user';

// Avoid
import React, { useState, useEffect } from 'react';
import UserService from '@/services/user';
```

### Error Handling

- Use try/catch for async operations
- Create custom error classes for domain-specific errors
- Always handle promises with `.catch()` or try/catch
- Log errors with appropriate context
- Never expose raw error messages to users

### Async/Await

- Always handle errors in async functions
- Use `Promise.all()` for parallel operations when appropriate
- Avoid unnecessary async/await nesting

### File Organization

- One export per file when possible (barrel files for directories)
- Use index.ts files for clean exports
- Maximum ~300 lines per file

### Testing Guidelines

- Write tests alongside code (TDD encouraged)
- Use descriptive test names
- Follow AAA pattern: Arrange, Act, Assert
- Mock external dependencies
- Test edge cases and error scenarios

## Project Structure

```
.opencode/
├── src/
│   ├── index.ts          # Entry point
│   ├── services/         # Business logic
│   ├── utils/            # Helper functions
│   ├── types/            # TypeScript types
│   └── __tests__/        # Test files
├── package.json
└── tsconfig.json
```

## Dependencies

- **@opencode-ai/plugin**: OpenCode AI integration

## Notes for AI Agents

1. Always run `bun run typecheck` and `bun test` before completing a task
2. If adding new dependencies, ensure they are compatible with bun
3. Keep the project minimal - avoid adding unnecessary dependencies
4. Follow the existing code patterns when modifying files
