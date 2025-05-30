# Testing Guidelines

## Core Principles

1. **Test Behavior, Not Implementation**: Focus on what the code does, not how
2. **Isolated Tests**: Each test should be independent
3. **Fast Execution**: Tests should run quickly
4. **Clear Naming**: Test names should describe the scenario
5. **Comprehensive Coverage**: Test happy paths, edge cases, and error scenarios

## Testing Stack

### Framework and Tools
```json
{
  "test": "tap",
  "devDependencies": {
    "tap": "^21.0.0",
    "@faker-js/faker": "^8.0.0",
    "mongodb-memory-server": "^9.0.0",
    "supertest": "^6.0.0",
    "jest": "^29.0.0"
  }
}
```

## Test File Organization

### Directory Structure
```
tests/
├── routes/              # API endpoint tests
│   ├── 01-statusRoutes.test.js
│   ├── 02-authRoutes.test.js
│   └── 03-usersRoutes.test.js
├── services/            # Service layer tests
│   ├── 01-ConfigService.test.js
│   ├── 02-UserService.test.js
│   └── 03-AuthService.test.js
├── utils/              # Test utilities
│   ├── testUtils.js
│   ├── factories.js
│   └── mocks.js
├── integration/        # Cross-service tests
└── setup/             # Test environment setup
```

### File Naming Convention
- **Pattern**: `[number]-[feature].test.js`
- **Purpose**: Numbers ensure execution order
- **Example**: `01-AuthService.test.js`

## Test Structure Patterns

### Basic Test Structure (TAP)
```javascript
import tap from "tap";
import { setupTestEnvironment, teardownTestEnvironment } from "../utils/testUtils.js";

tap.test("UserService", async (t) => {
  // Setup
  let userService;
  let testUser;
  
  t.beforeEach(async () => {
    await setupTestEnvironment();
    userService = new UserService(/* dependencies */);
    testUser = await createTestUser();
  });
  
  t.afterEach(async () => {
    await clearDatabase();
  });
  
  t.teardown(async () => {
    await teardownTestEnvironment();
  });
  
  // Test cases
  await t.test("getUserById", async (t) => {
    await t.test("should return user when exists", async (t) => {
      const user = await userService.getUserById(testUser.id);
      t.equal(user.email, testUser.email);
      t.equal(user.username, testUser.username);
    });
    
    await t.test("should throw NotFoundError when user doesn't exist", async (t) => {
      const fakeId = new mongoose.Types.ObjectId();
      await t.rejects(
        userService.getUserById(fakeId),
        NotFoundError
      );
    });
  });
});
```

### API Route Testing
```javascript
import tap from "tap";
import supertest from "supertest";
import { createApp } from "../../src/app.js";

tap.test("Auth Routes", async (t) => {
  let app;
  let request;
  
  t.beforeEach(async () => {
    app = await createApp();
    request = supertest(app);
  });
  
  await t.test("POST /api/v1/auth/login", async (t) => {
    await t.test("should return tokens for valid credentials", async (t) => {
      const response = await request
        .post("/api/v1/auth/login")
        .send({
          email: "test@example.com",
          password: "ValidPassword123!"
        })
        .expect(200);
      
      t.ok(response.body.data.accessToken);
      t.ok(response.body.data.refreshToken);
      t.equal(response.body.data.user.email, "test@example.com");
    });
    
    await t.test("should return 401 for invalid credentials", async (t) => {
      const response = await request
        .post("/api/v1/auth/login")
        .send({
          email: "test@example.com",
          password: "WrongPassword"
        })
        .expect(401);
      
      t.equal(response.body.error.code, "AUTH_INVALID_CREDENTIALS");
    });
    
    await t.test("should validate request body", async (t) => {
      const response = await request
        .post("/api/v1/auth/login")
        .send({
          email: "invalid-email"
        })
        .expect(400);
      
      t.equal(response.body.error.code, "VALIDATION_ERROR");
    });
  });
});
```

## Test Utilities

### Database Setup
```javascript
// tests/utils/testUtils.js
import mongoose from "mongoose";
import { MongoMemoryServer } from "mongodb-memory-server";
import { connectToCoreDb, connectToAnalyticsDb, connectToSystemLogsDb } from "../../src/db/index.js";

let mongoServer;

export async function setupTestEnvironment() {
  // Set test environment
  process.env.NODE_ENV = "test";
  process.env.ALLOW_EXECUTING_TESTS = "true";
  process.env.TAP_TIMEOUT = "0";
  
  // Start in-memory MongoDB
  mongoServer = await MongoMemoryServer.create();
  const uri = mongoServer.getUri();
  
  // Override database URIs
  process.env.CORE_DB_URI = uri + "core";
  process.env.ANALYTICS_DB_URI = uri + "analytics";
  process.env.SYSTEMLOGS_DB_URI = uri + "logs";
  
  // Connect to databases
  await Promise.all([
    connectToCoreDb(),
    connectToAnalyticsDb(),
    connectToSystemLogsDb()
  ]);
}

export async function teardownTestEnvironment() {
  await mongoose.disconnect();
  if (mongoServer) {
    await mongoServer.stop();
  }
}

export async function clearDatabase() {
  const connections = [
    mongoose.connections.find(c => c.name === "core"),
    mongoose.connections.find(c => c.name === "analytics"),
    mongoose.connections.find(c => c.name === "logs")
  ];
  
  for (const connection of connections) {
    if (connection) {
      const collections = connection.collections;
      for (const key in collections) {
        await collections[key].deleteMany({});
      }
    }
  }
}
```

### Test Data Factories
```javascript
// tests/utils/factories.js
import { faker } from "@faker-js/faker";
import bcrypt from "bcrypt";

export async function createTestUser(overrides = {}) {
  const userData = {
    email: faker.internet.email(),
    username: faker.internet.userName(),
    password: await bcrypt.hash("TestPassword123!", 10),
    firstName: faker.person.firstName(),
    lastName: faker.person.lastName(),
    role: USER_ROLE_MEMBER,
    status: USER_STATUS_ACTIVE,
    organizationId: new mongoose.Types.ObjectId(),
    ...overrides
  };
  
  return await UserModel.create(userData);
}

export async function createTestOrganization(overrides = {}) {
  const orgData = {
    name: faker.company.name(),
    code: faker.string.alphanumeric(8).toUpperCase(),
    status: ORGANIZATION_STATUS_ACTIVE,
    settings: {
      maxUsers: 10,
      features: ["basic"]
    },
    ...overrides
  };
  
  return await OrganizationModel.create(orgData);
}

export async function createTestProject(organizationId, overrides = {}) {
  const projectData = {
    name: faker.commerce.productName(),
    organizationId,
    status: PROJECT_STATUS_ACTIVE,
    settings: {},
    ...overrides
  };
  
  return await ProjectModel.create(projectData);
}

// Bulk creation helpers
export async function createTestUsers(count, overrides = {}) {
  const users = [];
  for (let i = 0; i < count; i++) {
    users.push(await createTestUser(overrides));
  }
  return users;
}
```

### Mock Services
```javascript
// tests/utils/mocks.js
export class MockEmailService {
  constructor() {
    this.sentEmails = [];
  }
  
  async sendEmail(to, subject, body) {
    this.sentEmails.push({ to, subject, body });
    return { messageId: faker.string.uuid() };
  }
  
  async sendWelcomeEmail(email) {
    return this.sendEmail(email, "Welcome", "Welcome body");
  }
  
  reset() {
    this.sentEmails = [];
  }
  
  getLastEmail() {
    return this.sentEmails[this.sentEmails.length - 1];
  }
}

export class MockCacheService {
  constructor() {
    this.cache = new Map();
  }
  
  async get(key) {
    return this.cache.get(key);
  }
  
  async set(key, value, ttl) {
    this.cache.set(key, value);
  }
  
  async delete(key) {
    return this.cache.delete(key);
  }
  
  async flush() {
    this.cache.clear();
  }
}

export class MockConfigService {
  constructor(configs = {}) {
    this.configs = new Map(Object.entries(configs));
  }
  
  async getConfigValue(key) {
    if (!this.configs.has(key)) {
      throw new NotFoundError(`Config not found: ${key}`);
    }
    return this.configs.get(key);
  }
  
  setConfig(key, value) {
    this.configs.set(key, value);
  }
}
```

## Testing Patterns

### Testing Async Operations
```javascript
tap.test("Async operation testing", async (t) => {
  await t.test("should handle successful async operation", async (t) => {
    const result = await someAsyncFunction();
    t.equal(result.status, "success");
  });
  
  await t.test("should handle async errors", async (t) => {
    await t.rejects(
      someAsyncFunctionThatFails(),
      {
        message: "Expected error message",
        code: "EXPECTED_ERROR_CODE"
      }
    );
  });
  
  await t.test("should timeout long operations", async (t) => {
    const timeoutPromise = new Promise((_, reject) => {
      setTimeout(() => reject(new Error("Timeout")), 1000);
    });
    
    await t.rejects(
      Promise.race([longRunningOperation(), timeoutPromise]),
      /Timeout/
    );
  });
});
```

### Testing Database Operations
```javascript
tap.test("Database operations", async (t) => {
  await t.test("should handle concurrent updates", async (t) => {
    const userId = new mongoose.Types.ObjectId();
    
    // Create user
    await UserModel.create({
      _id: userId,
      credits: 100
    });
    
    // Simulate concurrent updates
    const updates = Array(10).fill().map((_, i) => 
      UserModel.findByIdAndUpdate(
        userId,
        { $inc: { credits: 10 } },
        { new: true }
      )
    );
    
    await Promise.all(updates);
    
    const finalUser = await UserModel.findById(userId);
    t.equal(finalUser.credits, 200, "All increments should be applied");
  });
  
  await t.test("should rollback transaction on error", async (t) => {
    const session = await mongoose.startSession();
    
    try {
      await session.withTransaction(async () => {
        await UserModel.create([{ email: "test1@example.com" }], { session });
        
        // This should fail due to duplicate email
        await UserModel.create([{ email: "test1@example.com" }], { session });
      });
      
      t.fail("Transaction should have failed");
    } catch (error) {
      t.ok(error, "Transaction rolled back");
    } finally {
      await session.endSession();
    }
    
    const count = await UserModel.countDocuments({ email: "test1@example.com" });
    t.equal(count, 0, "No users should be created");
  });
});
```

### Testing Error Scenarios
```javascript
tap.test("Error handling", async (t) => {
  await t.test("should handle validation errors", async (t) => {
    try {
      await userService.createUser({
        email: "invalid-email",
        username: "a" // Too short
      });
      t.fail("Should have thrown validation error");
    } catch (error) {
      t.ok(error instanceof ValidationError);
      t.equal(error.code, "VALIDATION_ERROR");
      t.ok(error.details.fields.includes("email"));
      t.ok(error.details.fields.includes("username"));
    }
  });
  
  await t.test("should handle database connection errors", async (t) => {
    // Temporarily close connection
    await mongoose.connection.close();
    
    try {
      await userService.getUserById("someId");
      t.fail("Should have thrown database error");
    } catch (error) {
      t.ok(error instanceof DatabaseError);
      t.equal(error.code, "DATABASE_CONNECTION_ERROR");
    } finally {
      // Reconnect
      await mongoose.connect(process.env.CORE_DB_URI);
    }
  });
});
```

### Testing with Mocks
```javascript
tap.test("Service with mocked dependencies", async (t) => {
  let userService;
  let mockEmailService;
  let mockCacheService;
  
  t.beforeEach(() => {
    mockEmailService = new MockEmailService();
    mockCacheService = new MockCacheService();
    
    userService = new UserService({
      emailService: mockEmailService,
      cacheService: mockCacheService,
      userModel: UserModel
    });
  });
  
  await t.test("should send welcome email on user creation", async (t) => {
    const userData = {
      email: "newuser@example.com",
      username: "newuser"
    };
    
    const user = await userService.createUser(userData);
    
    // Verify email was sent
    t.equal(mockEmailService.sentEmails.length, 1);
    const sentEmail = mockEmailService.getLastEmail();
    t.equal(sentEmail.to, userData.email);
    t.equal(sentEmail.subject, "Welcome");
  });
  
  await t.test("should cache user after fetch", async (t) => {
    const user = await createTestUser();
    
    // First fetch - from database
    const fetchedUser = await userService.getUserById(user.id);
    t.equal(fetchedUser.id, user.id);
    
    // Verify cached
    const cachedUser = await mockCacheService.get(`user:${user.id}`);
    t.ok(cachedUser);
    t.equal(cachedUser.id, user.id);
    
    // Delete from database
    await UserModel.findByIdAndDelete(user.id);
    
    // Second fetch - should come from cache
    const cachedFetch = await userService.getUserById(user.id);
    t.equal(cachedFetch.id, user.id);
  });
});
```

## Performance Testing

### Load Testing Pattern
```javascript
tap.test("Performance tests", async (t) => {
  await t.test("should handle 1000 concurrent requests", async (t) => {
    const requests = Array(1000).fill().map((_, i) => 
      request
        .get(`/api/v1/users/${i}`)
        .set("Authorization", `Bearer ${authToken}`)
    );
    
    const startTime = Date.now();
    const results = await Promise.allSettled(requests);
    const duration = Date.now() - startTime;
    
    const successful = results.filter(r => r.status === "fulfilled").length;
    const failed = results.filter(r => r.status === "rejected").length;
    
    t.ok(successful > 950, `At least 95% success rate: ${successful}/1000`);
    t.ok(duration < 5000, `Should complete within 5 seconds: ${duration}ms`);
    
    console.log(`Performance: ${successful} successful, ${failed} failed in ${duration}ms`);
  });
});
```

## Test Environment Variables

### Required Test Environment
```bash
# .env.test
NODE_ENV=test
ALLOW_EXECUTING_TESTS=true
TAP_TIMEOUT=0
LOG_LEVEL=error
DISABLE_RATE_LIMITING=true
DISABLE_AUTH_CACHE=true
USE_IN_MEMORY_DB=true
```

## Coverage Requirements

### Minimum Coverage Targets
- **Overall**: 80%
- **Critical Services**: 90%
- **API Routes**: 85%
- **Utilities**: 75%

### Running Coverage
```bash
# Run tests with coverage
npm test -- --coverage

# Generate HTML report
npm test -- --coverage-report=html

# Check coverage thresholds
npm test -- --coverage --check-coverage --lines=80 --functions=80 --branches=70
```

## Best Practices

### DO:
- ✅ Write descriptive test names
- ✅ Test one behavior per test
- ✅ Use factories for test data
- ✅ Mock external dependencies
- ✅ Test error scenarios
- ✅ Clean up after each test
- ✅ Use beforeEach/afterEach for setup
- ✅ Test edge cases and boundaries

### DON'T:
- ❌ Share state between tests
- ❌ Test implementation details
- ❌ Use production database
- ❌ Skip error scenario tests
- ❌ Write overly complex tests
- ❌ Ignore flaky tests
- ❌ Test external services directly

## Integration Testing

### Cross-Service Testing
```javascript
tap.test("Integration: User registration flow", async (t) => {
  const mockEmailService = new MockEmailService();
  const app = await createApp({ emailService: mockEmailService });
  const request = supertest(app);
  
  await t.test("complete registration flow", async (t) => {
    // 1. Register user
    const registerResponse = await request
      .post("/api/v1/auth/register")
      .send({
        email: "newuser@example.com",
        username: "newuser",
        password: "SecurePassword123!"
      })
      .expect(201);
    
    const { userId } = registerResponse.body.data;
    
    // 2. Verify email was sent
    t.equal(mockEmailService.sentEmails.length, 1);
    const verificationEmail = mockEmailService.getLastEmail();
    const verificationToken = extractTokenFromEmail(verificationEmail.body);
    
    // 3. Verify email
    await request
      .post("/api/v1/auth/verify-email")
      .send({ token: verificationToken })
      .expect(200);
    
    // 4. Login with verified account
    const loginResponse = await request
      .post("/api/v1/auth/login")
      .send({
        email: "newuser@example.com",
        password: "SecurePassword123!"
      })
      .expect(200);
    
    t.ok(loginResponse.body.data.accessToken);
    
    // 5. Access protected route
    await request
      .get("/api/v1/users/me")
      .set("Authorization", `Bearer ${loginResponse.body.data.accessToken}`)
      .expect(200);
  });
});
```

## Remember
**Tests are documentation. They should clearly show how the system is supposed to work and what happens when things go wrong.**