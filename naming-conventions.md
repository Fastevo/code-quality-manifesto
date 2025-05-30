# Naming Conventions

## The Fundamental Principle
Every name in the codebase must be self-documenting and unambiguous. If a developer needs to guess what something means, the name has failed.

## The Three-Question Test (MANDATORY)
Every variable, function, and class name must answer these three questions:
1. **What** is it exactly?
2. **Where** does it come from or what generates it?
3. **Why** does it exist or what is its purpose?

## Variable Naming Rules

### Zero Ambiguity Rule
No mental translation should be required to understand a variable's purpose.

### The Six-Month Test
Would someone who has never seen this code understand what this variable contains just from its name after 6 months?

### BANNED Variable Names (Automatic Rejection)
- **Generic names**: `data`, `item`, `temp`, `flag`, `val`, `result`, `obj`, `str`, `p`
- **Ambiguous booleans**: `isOk`, `isReady`, `isActive`, `isValid`
- **Vague API terms**: `response`, `request`, `payload`, `body`, `endpoint`
- **Generic loops**: `i`, `j`, `k` (use descriptive names)
- **Unclear time**: `time`, `duration`, `delay`
- **Generic elements**: `element`, `container`, `div`, `span`
- **Vague states**: `status`, `state`, `mode`
- **Database generics**: `user`, `record`, `row`, `doc`, `model`

### Required Standards

#### Boolean Variables
- **Pattern**: `is/has/can/should + descriptive phrase`
- **Good**: `isUserAuthenticationTokenCurrentlyValid`, `hasMediaContentFinishedProcessing`
- **Bad**: `isOk`, `valid`, `flag`

#### API Variables
- **Pattern**: `businessContext + operationType + dataType`
- **Good**: `userAuthenticationRequestBody`, `mediaContentCreationResponse`
- **Bad**: `response`, `payload`, `data`

#### Object Variables
- **Pattern**: `purposeFor + context` or `actionResult + businessContext`
- **Good**: `fieldsForNewPlaybackSession`, `fraudScoreCalculationResult`
- **Bad**: `config`, `data`, `result`

#### Function Result Variables
- **Pattern**: `businessConcept + purpose`
- **Good**: `loginTokensExpireIn`, `encryptedContentKeys`
- **Bad**: `result`, `output`, `ret`

#### Database Variables
- **Pattern**: `businessEntity + storageType + purpose`
- **Good**: `authenticatedUserDatabaseRecord`, `contentMetadataDocument`
- **Bad**: `user`, `record`, `doc`

#### Error Variables
- **Pattern**: `operationContext + failureType + errorCategory`
- **Good**: `databaseConnectionTimeoutError`, `userAuthenticationValidationException`
- **Bad**: `error`, `err`, `e`

#### Loop Variables
- **Pattern**: `current + entityType + iterationType`
- **Good**: `currentContentItemIndex`, `protectionRuleIterator`
- **Bad**: `i`, `j`, `index`

## Function Naming

### Core Patterns
- **get** + Entity + **By** + Criteria: `getUserById`, `getOrganizationByCode`
- **create** + Entity: `createOrganization`, `createPlaybackToken`
- **delete/remove** + Entity: `deleteContent`, `removeUserToken`
- **validate** + What + **AndAction**: `validateCredentialsAndGetUser`
- **generate** + What: `generateJwtToken`, `generateUniqueIdentifier`

### Layer-Specific Conventions

#### Controllers (HTTP handlers)
- HTTP method + resource: `postLogin`, `getUsers`, `putContent`

#### Services (Business logic)
- Action-oriented: `authenticateUser`, `processMediaContent`

#### Utils (Helpers)
- Action + type: `ensureArray`, `escapeJavascriptString`

#### Private Functions
- Underscore prefix: `_sanitizeUserInput`, `_buildRequestHeaders`

## Class and File Naming

### Classes
- **Pattern**: PascalCase describing the entity
- **Service Classes**: Append "Service" suffix
- **Examples**: `AuthenticationService`, `MediaProcessor`, `ConfigProvider`

### Files
- **Service Files**: `[Name]Service.js`
- **Schema Files**: `[Entity]Schema.js`
- **Configuration**: camelCase descriptive names
- **Examples**: `configProvider.js`, `UserSchema.js`, `AuthService.js`

## Constants and Enums

### Constants
- **Pattern**: SCREAMING_SNAKE_CASE
- **Include context**: `MEDIA_PROTECTION_STATUS_PROCESSING`
- **Not just**: `STATUS_PROCESSING`

### Configuration Keys
- **Pattern**: Domain prefix + descriptive key
- **Example**: `COMMON_CONFIG_KEY_AWS_REGION`

### Enums
- **Pattern**: Name + `_ENUM` suffix
- **Example**: `PROTECTION_LEVEL_ENUM`, `USER_ROLE_ENUM`

## CSS and Frontend

### CSS Classes
- **Pattern**: Kebab-case with semantic meaning
- **Component**: `.user-profile-card`
- **State**: `.is-loading`, `.has-error`
- **Modifier**: `.button--primary`

### CSS Variables
- **Pattern**: `--category-semantic-name`
- **Examples**: `--color-primary`, `--spacing-large`

### Data Attributes
- **Pattern**: `data-descriptive-name`
- **Examples**: `data-user-role`, `data-content-id`

## Domain-Specific Requirements

### When Working with External APIs
- Include the service name in variables
- Example: `stripePaymentIntentResponse` not just `paymentResponse`

### When Crossing Service Boundaries
- Include source context in variable names
- Example: `userDataFromAuthService` not just `userData`

### For Shared/Reusable Code
- Use the most specific name that covers all use cases
- Avoid domain-specific terms unless in domain-specific package

## Test-Specific Relaxations

In test files only, these are acceptable:
- Standard loops: `i`, `j`, `k` for simple iteration
- Generic test variables: `result`, `response` when context is clear
- Short identifiers: `user`, `org` for test data
- Common patterns: `actual`, `expected`, `error`

However, complex test scenarios should still use descriptive names.

## Enforcement

### Code Review Checklist
- [ ] Every variable name passes the Three-Question Test
- [ ] No banned variable names are used
- [ ] Boolean variables describe exact conditions
- [ ] API variables include business context
- [ ] Function names follow verb patterns
- [ ] Constants include domain context
- [ ] No mental translation required

### Automatic Rejection Triggers
1. Any banned variable name
2. Single-letter variables (except specific math contexts)
3. Generic names without context
4. Variables failing the Six-Month Test

## Remember
**Clarity is not optional. Self-documenting code is the only acceptable code.**