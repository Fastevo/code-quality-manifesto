# Code Quality Manifesto

This manifesto defines the coding standards, patterns, and best practices that MUST be followed across all projects in this codebase. These guidelines ensure consistency, maintainability, and quality.

## 📋 Guidelines Overview

### 1. [Naming Conventions](./naming-conventions.md)

- **The Three-Question Test**: Every name must answer What, Where, and Why
- **Zero Ambiguity Rule**: No mental translation required
- **Banned Variable Names**: Comprehensive list of forbidden names
- **Function Naming Patterns**: Consistent verb-based naming

### 2. [Project Structure](./project-structure.md)

- **Directory Organization**: Standard folder structure for all projects
- **Service Boundaries**: Clear separation of concerns
- **Three-Database Architecture**: Core, Analytics, and System Logs
- **Microservice-Ready Design**: Easy service extraction

### 3. [Definitions and Schemas](./definitions-and-schemas.md)

- **Commons Package System**: Shared code organization
- **Schema Design Patterns**: Mongoose best practices
- **Constants Management**: No magic strings or numbers
- **Migration Strategies**: Schema versioning approach

### 4. [Configuration Management](./configuration-management.md)

- **Three-Tier System**: Environment, Database Common, Service-Specific
- **Inheritance Hierarchy**: System → Organization → Project
- **Encryption Standards**: Sensitive config protection
- **Plan-Based Features**: Feature restrictions and limits

### 5. [Service Patterns](./service-patterns.md)

- **Dependency Injection**: Constructor-based dependencies
- **Singleton Pattern**: Service instantiation
- **Base Service Class**: Common functionality
- **Inter-Service Communication**: Clear interfaces

### 6. [Error Handling](./error-handling.md)

- **Error Class Hierarchy**: Consistent error types
- **Error Codes Registry**: Standardized error codes
- **Logging Standards**: Structured error logging
- **Recovery Patterns**: Retry and fallback strategies

### 7. [Database Patterns](./database-patterns.md)

- **Connection Management**: Pooling and monitoring
- **Schema Design**: Validation, indexes, middleware
- **Caching Strategy**: Query result caching
- **Transaction Patterns**: Multi-document operations

### 8. [Testing Guidelines](./testing-guidelines.md)

- **TAP Framework**: Standard testing approach
- **Test Organization**: Numbered file execution
- **Mock Patterns**: Service dependency mocking
- **Coverage Requirements**: Minimum 80% overall

### 9. [Code Style](./code-style.md)

- **ES Modules Only**: Import/export patterns
- **Async/Await**: No callbacks or promise chains
- **Security First**: Input validation, no sensitive logging
- **Performance**: Batch operations, avoid N+1 queries

## 🚨 Critical Rules

### Automatic Code Rejection Triggers

1. **Generic variable names**: `data`, `item`, `temp`, `flag`, `result`
2. **Missing file extensions** in imports
3. **Hardcoded configuration values**
4. **Unhandled promise rejections**
5. **Console.log statements**
6. **Sensitive data in logs**
7. **Direct process.env access**
8. **Magic strings/numbers**

### The Golden Rules

1. **English Only**: All code, comments, and documentation in English
2. **Self-Documenting Code**: Names should explain purpose without comments
3. **Fail Fast**: Validate early, throw descriptive errors
4. **Cache Everything**: Database queries must support caching
5. **Test Everything**: Minimum 80% coverage, test edge cases

## 🏗️ Architecture Principles

### Microservice-Ready Design

- Clear service boundaries through route organization
- Independent database models per domain
- Service-specific configuration sections
- Commons packages for truly shared code only

### Configuration Hierarchy

```
Environment Variables (System Level)
    ↓
Database Common Configs (Shared Services)
    ↓
Service-Specific Configs (Domain Level)
    ↓
Organization Configs (Tenant Level)
    ↓
Project Configs (Instance Level)
```

### Database Separation

```
core-db:       Authentication, Users, Organizations, Projects
analytics-db:  Metrics, Usage Statistics, Performance Data
systemlogs-db: Application Logs, Errors, Audit Trails
```

## 📝 Enforcement

### Code Review Checklist

Every PR must pass these checks:

- [ ] Variable names follow naming conventions
- [ ] No banned patterns or anti-patterns
- [ ] Proper error handling with specific error types
- [ ] Configuration through appropriate tier
- [ ] Tests included with good coverage
- [ ] No hardcoded values
- [ ] Consistent code style
- [ ] Clear commit messages

### AI Assistant Instructions

When working with AI assistants, they should:

1. **ALWAYS run in ultrathink mode** for deep analysis
2. **Reject code with banned variable names**
3. **Enforce the Three-Question Test** for all names
4. **Ensure proper error handling patterns**
5. **Validate configuration management approach**
6. **Check for security best practices**

## 🎯 Goal

The goal of this manifesto is to ensure that any developer (human or AI) can:

- Understand code without additional context
- Maintain consistent patterns across all projects
- Avoid common pitfalls and anti-patterns
- Build scalable, maintainable services
- Extract microservices when needed

## 💡 Remember

> **"Code is written once but read many times. Optimize for the reader, not the writer."**

Every decision should prioritize:

1. **Clarity** over cleverness
2. **Consistency** over personal preference
3. **Explicitness** over implicitness
4. **Safety** over convenience
5. **Maintainability** over premature optimization

---

_This manifesto is a living document. Update it as patterns evolve, but maintain backward compatibility with existing code._
