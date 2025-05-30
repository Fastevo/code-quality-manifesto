# Definitions and Schemas Guidelines

## Core Principle
All constants, enums, and schemas must be centralized and reusable. Magic strings and numbers are forbidden in business logic.

## Where Things Belong

### Decision Tree for Placement

1. **Is it used by multiple projects?**
   - YES → Goes in a `commons` package
   - NO → Goes in the project's `src/definitions/`

2. **Is it domain-specific?**
   - YES → Goes in the domain's commons package (e.g., `mediaprotection-commons`)
   - NO → Goes in the base commons package

3. **Is it a database schema?**
   - YES → Goes in `schemas/` directory
   - NO → Goes in `definitions/definitions.js`

## Commons Package Structure

### When to Create a Commons Package
- When 2+ projects need the same definitions
- When establishing a new business domain
- When schemas need to be shared across services

### Commons Package Organization
```
[domain]-commons/
├── index.js                    # Usually empty, direct imports preferred
├── definitions/
│   └── definitions.js          # Constants, enums, keys
├── schemas/                    # Mongoose/database schemas
│   ├── [Entity]Schema.js
│   └── subdomain/             # Complex domains can have subdirectories
├── models/                     # Alternative to schemas for non-Mongoose
├── utils/                      # Shared utility functions
└── services/                   # Shared service logic
```

## Definitions File Structure

### Organization Pattern
```javascript
// 1. Configuration Keys
export const COMMON_CONFIG_KEY_DATABASE_URL = "DATABASE_URL";
export const COMMON_CONFIG_KEY_API_KEY = "API_KEY";

// 2. Status Constants
export const STATUS_PENDING = "pending";
export const STATUS_PROCESSING = "processing";
export const STATUS_COMPLETED = "completed";

// 3. Enums (Arrays of valid values)
export const STATUS_ENUM = [STATUS_PENDING, STATUS_PROCESSING, STATUS_COMPLETED];

// 4. Numeric Constants
export const MAX_RETRY_ATTEMPTS = 3;
export const DEFAULT_TIMEOUT_SECONDS = 30;

// 5. Action Types
export const ACTION_CREATE = "create";
export const ACTION_UPDATE = "update";
export const ACTION_DELETE = "delete";
```

### Naming Conventions for Definitions

#### Configuration Keys
- **Pattern**: `[DOMAIN]_CONFIG_KEY_[NAME]`
- **Example**: `MEDIA_PROTECTION_CONFIG_KEY_STORAGE_BUCKET`

#### Status/Type Constants
- **Pattern**: `[DOMAIN]_[ENTITY]_[STATUS]`
- **Example**: `MEDIA_PROTECTION_JOB_STATUS_PROCESSING`

#### Enums
- **Pattern**: `[NAME]_ENUM`
- **Must be**: Array of the related constants
- **Example**: 
  ```javascript
  export const JOB_STATUS_ENUM = [
    JOB_STATUS_PENDING,
    JOB_STATUS_PROCESSING,
    JOB_STATUS_COMPLETED
  ];
  ```

#### Action Constants
- **Pattern**: `[DOMAIN]_ACTION_[VERB]`
- **Example**: `USER_ACTION_CREATE_ORGANIZATION`

## Schema Guidelines

### File Naming
- **Pattern**: `[Entity]Schema.js`
- **Always**: Singular entity name
- **Example**: `UserSchema.js`, `ContentSchema.js`

### Schema Structure
```javascript
import mongoose from "mongoose";
import validator from "validator";
import { STATUS_ENUM } from "../definitions/definitions.js";

const schema = new mongoose.Schema({
  // Required fields with validation
  name: {
    type: String,
    required: true,
    trim: true,
    minlength: 1,
    maxlength: 255
  },
  
  // Enum fields using definitions
  status: {
    type: String,
    required: true,
    enum: STATUS_ENUM,
    default: STATUS_ENUM[0]
  },
  
  // Nested objects
  configuration: {
    type: mongoose.Schema.Types.Mixed,
    required: false,
    default: {}
  },
  
  // References
  userId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "User",
    required: true
  }
}, {
  timestamps: true,  // adds createdAt, updatedAt
  collection: "contents"  // explicit collection name
});

// Indexes
schema.index({ userId: 1, status: 1 });

// Export pattern
const ContentSchema = mongoose.models.Content || mongoose.model("Content", schema);
export default ContentSchema;
```

### Schema Best Practices

1. **Always use timestamps**: `{ timestamps: true }`
2. **Explicit collection names**: Prevent pluralization issues
3. **Use definitions for enums**: Never hardcode values
4. **Add appropriate indexes**: Based on query patterns
5. **Validate at schema level**: Use built-in validators
6. **Document complex fields**: Add comments for non-obvious fields

## Usage Patterns

### Importing from Commons
```javascript
// Import specific constants
import { 
  MEDIA_PROTECTION_STATUS_PROCESSING,
  MEDIA_PROTECTION_STATUS_ENUM 
} from "@visiontsnpm/mediaprotection-commons/definitions/definitions.js";

// Import schemas
import ContentSchema from "@visiontsnpm/mediaprotection-commons/schemas/ContentSchema.js";
```

### Project-Specific Definitions
```javascript
// In src/definitions/definitions.js
export const INTERNAL_QUEUE_NAME = "processing-queue";
export const INTERNAL_RETRY_DELAY_MS = 5000;
```

## Anti-Patterns to Avoid

### ❌ Magic Strings
```javascript
// BAD
if (status === "processing") { }

// GOOD
if (status === JOB_STATUS_PROCESSING) { }
```

### ❌ Hardcoded Configuration
```javascript
// BAD
const timeout = 30000;

// GOOD
const timeout = configProvider.API_TIMEOUT_MS;
```

### ❌ Scattered Schemas
```javascript
// BAD - Schema defined in service file
const userSchema = { name: String, email: String };

// GOOD - Import from schemas
import UserSchema from "../schemas/UserSchema.js";
```

### ❌ Duplicate Definitions
```javascript
// BAD - Same constant in multiple files
// File A: const MAX_RETRIES = 3;
// File B: const MAX_RETRIES = 3;

// GOOD - Single source of truth
import { MAX_RETRY_ATTEMPTS } from "../definitions/definitions.js";
```

## Migration Guidelines

### When Moving to Commons

1. **Identify shared definitions** across projects
2. **Create or update** the appropriate commons package
3. **Move definitions** maintaining the same names
4. **Update imports** in all consuming projects
5. **Version bump** the commons package
6. **Update dependencies** in consuming projects

### Versioning Commons
- Use semantic versioning
- Patch: New constants/definitions
- Minor: New schemas or breaking constant renames
- Major: Schema structure changes

## Domain-Specific Patterns

### For Configuration Services
- Group by feature: `AUTH_`, `STORAGE_`, `API_`
- Include units in names: `_TIMEOUT_MS`, `_SIZE_BYTES`

### For Status Management
- Complete lifecycle: `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED`
- Include transition states: `CANCELLING`, `CANCELLED`

### For API Responses
- Standard fields: `SUCCESS`, `ERROR`, `VALIDATION_FAILED`
- HTTP status mappings as constants

## Testing Definitions

### Test-Specific Constants
```javascript
// In tests/utils/testConstants.js
export const TEST_USER_EMAIL = "test@example.com";
export const TEST_TIMEOUT_MS = 10000;
```

### Never Use Production Definitions for Test Data
- Create separate test constants
- Prefix with `TEST_` for clarity
- Keep in test directories only

## Remember
**Every magic value must be a named constant. Every shared definition must be in a commons package.**