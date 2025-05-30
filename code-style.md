# Code Style Guidelines

## Language Requirements

### English Only

- **All code must be in English**: Variables, functions, classes, comments, commits
- **No exceptions**: Even internal tools must use English
- **Consistency**: Use American English spelling consistently

## ES Modules

### Import/Export Rules

```javascript
// ✅ GOOD - Named exports with explicit extensions
import { UserService } from "./services/UserService.js";
import logger from "./utils/logger.js";
import { CONSTANTS } from "../definitions/definitions.js";

// ❌ BAD - Missing extensions, require syntax
const UserService = require("./services/UserService");
import { UserService } from "./services/UserService"; // No .js
```

### Import Organization

```javascript
// 1. External packages
import express from "express";
import mongoose from "mongoose";
import bcrypt from "bcrypt";

// 2. Internal packages (commons)
import { MediaProtectionSchema } from "@visiontsnpm/mediaprotection-commons/schemas/MediaProtectionSchema.js";
import { COMMON_CONFIG_KEY_API_HOSTNAME } from "@visiontsnpm/commons/definitions/definitions.js";

// 3. Local imports - config and setup
import configProvider from "./config/configProvider.js";
import { connectToDatabase } from "./db/coreDb.js";

// 4. Local imports - services and models
import UserService from "./services/UserService.js";
import { UserModel } from "./models/User.js";

// 5. Local imports - utils and helpers
import logger from "./utils/logger.js";
import { validateEmail } from "./utils/validators.js";
```

## Async/Await

### Always Use Async/Await

```javascript
// ✅ GOOD - Clean async/await
async function getUser(userId) {
  try {
    const user = await UserModel.findById(userId);
    if (!user) {
      throw new NotFoundError("User", userId);
    }
    return user;
  } catch (error) {
    logger.error("Failed to get user", { userId, error });
    throw error;
  }
}

// ❌ BAD - Promise chains
function getUser(userId) {
  return UserModel.findById(userId)
    .then((user) => {
      if (!user) {
        throw new NotFoundError("User", userId);
      }
      return user;
    })
    .catch((error) => {
      logger.error("Failed to get user", { userId, error });
      throw error;
    });
}
```

### Parallel Execution

```javascript
// ✅ GOOD - Parallel when independent
const [user, organization, projects] = await Promise.all([
  getUserById(userId),
  getOrganizationById(orgId),
  getProjectsByOrgId(orgId),
]);

// ❌ BAD - Sequential when could be parallel
const user = await getUserById(userId);
const organization = await getOrganizationById(orgId);
const projects = await getProjectsByOrgId(orgId);
```

## Comments and Documentation

### When to Comment

```javascript
// ✅ GOOD - Explains complex business logic
// Calculate the effective plan by checking project-level first,
// then organization-level, finally falling back to default
async function getEffectivePlan(projectId) {
  // Project-level plans override organization plans
  const projectPlan = await checkProjectPlan(projectId);
  if (projectPlan) return projectPlan;

  // Check organization plan as fallback
  const project = await getProject(projectId);
  const orgPlan = await checkOrganizationPlan(project.organizationId);
  if (orgPlan) return orgPlan;

  // Use system default if no specific plan assigned
  return getDefaultPlan();
}

// ❌ BAD - Obvious comments
// Get user by id
async function getUserById(id) {
  // Find user in database
  const user = await UserModel.findById(id);
  // Return the user
  return user;
}
```

### JSDoc for every function

```javascript
/**
 * Creates a new playback token for protected content
 * @param {Object} options - Token creation options
 * @param {string} options.contentId - The content to access
 * @param {string} options.userId - The user requesting access
 * @param {number} options.expiresIn - Token expiry in seconds
 * @param {Object} options.restrictions - Playback restrictions
 * @returns {Promise<{token: string, expiresAt: Date}>}
 * @throws {NotFoundError} If content doesn't exist
 * @throws {ForbiddenError} If user lacks access
 */
async function createPlaybackToken(options) {
  // Implementation
}
```

## Function Design

### Single Responsibility

```javascript
// ✅ GOOD - Each function has one clear purpose
async function validateUserCredentials(email, password) {
  const user = await UserModel.findOne({ email });
  if (!user) return null;

  const isValid = await bcrypt.compare(password, user.password);
  return isValid ? user : null;
}

async function createAuthTokens(user) {
  const accessToken = generateAccessToken(user);
  const refreshToken = generateRefreshToken(user);

  await saveRefreshToken(user.id, refreshToken);

  return { accessToken, refreshToken };
}

// ❌ BAD - Multiple responsibilities
async function loginUser(email, password) {
  // Validation, authentication, token generation all in one
  if (!email || !password) throw new Error("Invalid input");

  const user = await UserModel.findOne({ email });
  if (!user) throw new Error("User not found");

  const isValid = await bcrypt.compare(password, user.password);
  if (!isValid) throw new Error("Invalid password");

  const token = jwt.sign({ userId: user.id }, SECRET);
  await TokenModel.create({ userId: user.id, token });

  await UserModel.updateOne({ _id: user.id }, { lastLogin: new Date() });

  return { user, token };
}
```

### Early Returns

```javascript
// ✅ GOOD - Early returns for cleaner code
async function processPayment(amount, userId) {
  if (amount <= 0) {
    throw new ValidationError("Amount must be positive");
  }

  const user = await getUserById(userId);
  if (!user.paymentMethodId) {
    throw new PaymentError("No payment method on file");
  }

  if (user.balance < amount) {
    throw new InsufficientFundsError("Insufficient balance");
  }

  // Main logic here with no deep nesting
  const payment = await chargePaymentMethod(user.paymentMethodId, amount);
  await updateUserBalance(userId, -amount);

  return payment;
}

// ❌ BAD - Nested conditions
async function processPayment(amount, userId) {
  if (amount > 0) {
    const user = await getUserById(userId);
    if (user.paymentMethodId) {
      if (user.balance >= amount) {
        const payment = await chargePaymentMethod(user.paymentMethodId, amount);
        await updateUserBalance(userId, -amount);
        return payment;
      } else {
        throw new Error("Insufficient balance");
      }
    } else {
      throw new Error("No payment method");
    }
  } else {
    throw new Error("Invalid amount");
  }
}
```

## Error Handling

### Consistent Error Handling

```javascript
// ✅ GOOD - Consistent error handling pattern
class UserController {
  async getUser(request, reply) {
    try {
      const { userId } = request.params;
      const user = await userService.getUserById(userId);

      return reply.send({
        success: true,
        data: user,
      });
    } catch (error) {
      // Let error handler middleware handle it
      throw error;
    }
  }
}

// ❌ BAD - Inconsistent error handling
async function getUser(request, reply) {
  try {
    const user = await userService.getUserById(request.params.userId);
    reply.send({ user });
  } catch (error) {
    console.log(error); // Don't use console.log
    reply.status(500).send({ error: "Something went wrong" }); // Too generic
  }
}
```

## Object and Array Operations

### Destructuring

```javascript
// ✅ GOOD - Use destructuring
const { email, username, role } = user;
const [first, second, ...rest] = array;

// Function parameters
async function createUser({ email, username, password, organizationId }) {
  // Use directly
}

// ❌ BAD - Repetitive property access
const email = user.email;
const username = user.username;
const role = user.role;
```

### Spread Operator

```javascript
// ✅ GOOD - Use spread for immutability
const updatedUser = {
  ...user,
  lastLogin: new Date(),
  loginCount: user.loginCount + 1,
};

const newArray = [...existingArray, newItem];

// ❌ BAD - Mutating objects
user.lastLogin = new Date();
user.loginCount++;
```

## Logging

### Structured Logging

```javascript
// ✅ GOOD - Structured with context
logger.info("User login successful", {
  userId: user.id,
  email: user.email,
  ip: request.ip,
  userAgent: request.headers["user-agent"],
});

logger.error("Payment processing failed", {
  error: {
    message: error.message,
    code: error.code,
    stack: error.stack,
  },
  context: {
    userId,
    amount,
    paymentMethodId,
  },
});

// ❌ BAD - Unstructured logging
console.log("User logged in: " + user.email);
logger.info(`Payment failed for user ${userId}: ${error.message}`);
```

## Security Practices

### Never Log Sensitive Data

```javascript
// ✅ GOOD - Sanitize sensitive data
logger.info("User authenticated", {
  userId: user.id,
  email: user.email,
  // Never log password, tokens, keys
});

const sanitizedConfig = {
  ...config,
  apiKey: "[REDACTED]",
  password: "[REDACTED]",
  secretKey: "[REDACTED]",
};

// ❌ BAD - Logging sensitive data
logger.info("Login attempt", {
  email: user.email,
  password: password, // NEVER!
  token: authToken, // NEVER!
});
```

### Input Validation

```javascript
// ✅ GOOD - Validate all inputs
function validateEmail(email) {
  if (!email || typeof email !== "string") {
    throw new ValidationError("Email is required");
  }

  if (!validator.isEmail(email)) {
    throw new ValidationError("Invalid email format");
  }

  return email.toLowerCase().trim();
}

// ❌ BAD - Trust user input
function processEmail(email) {
  // Using email directly without validation
  return UserModel.findOne({ email });
}
```

## Performance Considerations

### Avoid N+1 Queries

```javascript
// ✅ GOOD - Use populate or join
const projectsWithUsers = await ProjectModel.find({ organizationId })
  .populate("createdBy", "email username")
  .populate("members", "email username role");

// ❌ BAD - N+1 query pattern
const projects = await ProjectModel.find({ organizationId });
for (const project of projects) {
  project.creator = await UserModel.findById(project.createdBy);
  project.memberDetails = await UserModel.find({
    _id: { $in: project.members },
  });
}
```

### Batch Operations

```javascript
// ✅ GOOD - Batch database operations
const userIds = [id1, id2, id3, id4, id5];
const users = await UserModel.find({ _id: { $in: userIds } });

// ❌ BAD - Individual queries in loop
const users = [];
for (const userId of userIds) {
  const user = await UserModel.findById(userId);
  users.push(user);
}
```

## Code Organization

### File Length

- **Maximum**: 300 lines per file
- **Ideal**: 100-200 lines
- **Action**: Split large files into smaller, focused modules

### Function Length

- **Maximum**: 50 lines per function
- **Ideal**: 10-30 lines
- **Action**: Extract complex logic into helper functions

### Export Patterns

```javascript
// ✅ GOOD - Clear exports
// Named exports for utilities
export function validateEmail(email) { }
export function validateUsername(username) { }
export const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

// Default export for classes/main functionality
export default class UserService { }

// ❌ BAD - Mixed export styles in same file
export default validateEmail;
export { validateUsername };
module.exports.EMAIL_REGEX = /.../; // Never use module.exports
```

## Git Commit Messages

### Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types

- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation only
- `style`: Code style (formatting, missing semicolons, etc)
- `refactor`: Code change that neither fixes a bug nor adds a feature
- `perf`: Performance improvement
- `test`: Adding missing tests
- `chore`: Changes to build process or auxiliary tools

### Examples

```
feat(auth): add two-factor authentication support

- Implement TOTP generation and verification
- Add QR code generation for authenticator apps
- Update user model with 2FA fields
- Add API endpoints for 2FA setup and verification

Closes #123
```

## Code Review Checklist

### Before Submitting PR

- [ ] All variable names pass the naming conventions
- [ ] No commented-out code
- [ ] No console.log statements
- [ ] All functions have single responsibility
- [ ] Error handling is consistent
- [ ] No hardcoded values (use config/constants)
- [ ] Sensitive data is not logged
- [ ] Tests are included and passing
- [ ] Code is formatted consistently

## Forbidden Patterns

### Never Use

- ❌ `var` keyword (use `const` or `let`)
- ❌ `==` comparison (use `===`)
- ❌ `eval()` function
- ❌ `with` statement
- ❌ Synchronous file operations in production code
- ❌ Callbacks for async operations (use async/await)
- ❌ Global variables
- ❌ Magic numbers/strings
- ❌ `any` type (in TypeScript projects)
- ❌ Direct `process.env` access (use configProvider)

## Remember

**Clean code is written for humans to read, not just for machines to execute. Optimize for clarity and maintainability.**
