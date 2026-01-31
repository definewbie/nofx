# AGENTS.md

This document provides build, test, and lint commands plus code style guidelines for agentic coding assistants.

## Commands

### Backend (Go)

```bash
# Build
make build                    # Build backend binary to ./nofx

# Tests
make test-backend             # Run all backend tests
go test -v ./...              # Run all tests with verbose output
go test ./path/to/package -v  # Run tests for specific package
go test -run TestFunctionName ./path/to/package  # Run specific test

# Linting & Formatting
make fmt                      # Format Go code with gofmt
make lint                     # Run golangci-lint
go vet ./...                  # Run go vet static analysis
gofmt -l .                   # Check for unformatted files

# Coverage
make test-coverage            # Generate coverage.html report
```

### Frontend (TypeScript/React)

```bash
cd web  # Frontend commands must be run from web/ directory

# Build & Type Check
npm run build                 # Build frontend (runs tsc + vite build)

# Tests
npm run test                  # Run all tests with Vitest
npm run test path/to/test.test.tsx  # Run specific test file

# Linting & Formatting
npm run lint                  # Run ESLint
npm run lint:fix              # Fix ESLint issues automatically
npm run format                # Format code with Prettier
npm run format:check          # Check formatting without changing files

# Development
npm run dev                   # Start Vite dev server
```

## Code Style Guidelines

### Go Backend

#### Imports
- Group in order: stdlib, third-party, internal packages
- No blank lines between groups
- Use consistent aliasing (e.g., `logger "nofx/logger"`)

#### Naming Conventions
- **Packages**: lowercase, single words (e.g., `package mcp`)
- **Constants/Exports**: PascalCase (e.g., `const ProviderCustom`, `func NewClient`)
- **Private vars**: camelCase (e.g., `var httpClient`)
- **Interfaces**: Simple, often -er suffix (e.g., `AIClient`, `Logger`)

#### Error Handling
- Always wrap errors with context: `fmt.Errorf("failed to open: %w", err)`
- Use SafeError helpers in API layer to avoid exposing sensitive info
- Log internal errors, return generic messages to clients
- Check `!= nil` for errors early (fast fail)

#### Testing
- Test names: `Test{Type}{Action}` (e.g., `TestNewClient_WithOptions`)
- Use table-driven tests for multiple scenarios
- Validate nil pointers and zero values explicitly
- Group related tests with comment sections

#### Project Structure
- Feature-based packages (mcp, store, trader, api, backtest)
- Stores use GORM with struct tags: `gorm:"primaryKey" json:"id"`
- API handlers in `api/` package using Gin framework
- Configuration via environment variables

#### Logging
- Use structured logging from `nofx/logger` package
- `logger.Infof()`, `logger.Errorf()`, `logger.Debugf()`
- Never log sensitive data (passwords, API keys, internal paths)

### TypeScript/React Frontend

#### Imports
```typescript
// 1. External libraries
import { useState } from 'react'
import axios from 'axios'

// 2. Internal modules
import { useLanguage } from '../contexts/LanguageContext'
import { Container } from './Container'

// 3. Types
import type { AIModel, Exchange } from '../types'
```

#### Naming Conventions
- **Components**: PascalCase (e.g., `HeaderBar`, `TraderConfigModal`)
- **Functions/vars**: camelCase (e.g., `loadConfigs`, `isAuthenticated`)
- **Types/interfaces**: PascalCase (e.g., `interface TraderConfigData`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `const API_BASE`)

#### TypeScript Rules
- Type explicitly in function signatures, but inline is acceptable
- Export interfaces for public contracts
- Use `any` sparingly - prefer `unknown` with type guards
- Use `interface` for public APIs, `type` for unions

#### React Patterns
- Functional components with hooks
- Zustand for global state management (stores/ directory)
- Context providers for app-wide concerns (AuthContext, LanguageContext)
- SWR for data fetching
- Class names: `className="flex items-center justify-between"`
- Inline styles for dynamic values: `style={{ color: '#EAECEF' }}`

#### Error Handling
- Custom HttpClient wraps all API calls
- Network errors auto-displayed via toast (sonner)
- Business logic errors (4xx) returned to caller
- Use ErrorBoundary for component-level error catching

#### API Integration
- Use `api` object from `lib/api.ts` (typed wrapper)
- Returns `ApiResponse<T>` with `{ success, data?, message? }`
- Auth token auto-added to requests via interceptor
- 401 errors trigger logout and redirect to login

#### Testing
- Vitest with jsdom environment
- Test files: `*.test.tsx` or `*.test.ts`
- Import from vitest: `import { describe, it, expect } from 'vitest'`
- Setup file: `src/test/setup.ts`

#### Styling
- Tailwind CSS for most styling
- Inline styles for dynamic values
- Use clsx + tailwind-merge for conditional classes
- Component libraries: Radix UI, Lucide icons

## Important Notes

- **NEVER commit secrets** (API keys, passwords, tokens)
- **Pre-commit hooks** run lint-staged on frontend files automatically
- **CI runs tests** on push to main/dev and on PRs
- **Backend** supports SQLite and PostgreSQL via GORM
- **Frontend build** includes TypeScript compilation (tsc)
- **ESLint rules** are relaxed in this project (see eslint.config.js)
- **Always run tests** before committing changes
