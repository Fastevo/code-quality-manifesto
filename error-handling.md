# Error Handling Guidelines

## Core Principles

1. **Errors are first-class citizens** - Design for failure scenarios
2. **Fail fast, fail descriptive** - Detect problems early with clear messages
3. **Never swallow errors silently** - Always log or propagate
4. **Consistent error structure** - Standardized format across all services

## Error Class Hierarchy

### Base Error Class
Every project should have a base error class:

```javascript
export class BaseError extends Error {
  constructor(message, code, statusCode, details = {}) {
    super(message);
    this.name = this.constructor.name;
    this.code = code;
    this.statusCode = statusCode;
    this.details = details;
    this.timestamp = new Date().toISOString();
    Error.captureStackTrace(this, this.constructor);
  }
}
```

### Domain-Specific Error Classes
```javascript
// In errors/ directory
export class AuthError extends BaseError {
  constructor(message, code = "AUTH_ERROR", details = {}) {
    super(message, code, 401, details);
  }
}

export class ValidationError extends BaseError {
  constructor(message, code = "VALIDATION_ERROR", details = {}) {
    super(message, code, 400, details);
  }
}

export class NotFoundError extends BaseError {
  constructor(resource, identifier) {
    super(
      `${resource} not found with identifier: ${identifier}`,
      "RESOURCE_NOT_FOUND",
      404,
      { resource, identifier }
    );
  }
}
```

## Error Codes

### Structure
- **Pattern**: `[DOMAIN]_[SPECIFIC_ERROR]`
- **Examples**: 
  - `AUTH_INVALID_CREDENTIALS`
  - `MEDIA_PROCESSING_FAILED`
  - `DATABASE_CONNECTION_TIMEOUT`

### Error Code Registry
Keep all error codes in definitions:
```javascript
// In definitions/definitions.js
export const ERROR_CODES = {
  // Auth errors
  AUTH_INVALID_CREDENTIALS: "AUTH_INVALID_CREDENTIALS",
  AUTH_TOKEN_EXPIRED: "AUTH_TOKEN_EXPIRED",
  AUTH_INSUFFICIENT_PERMISSIONS: "AUTH_INSUFFICIENT_PERMISSIONS",
  
  // Validation errors
  VALIDATION_MISSING_REQUIRED_FIELD: "VALIDATION_MISSING_REQUIRED_FIELD",
  VALIDATION_INVALID_FORMAT: "VALIDATION_INVALID_FORMAT",
  
  // System errors
  SYSTEM_DATABASE_ERROR: "SYSTEM_DATABASE_ERROR",
  SYSTEM_EXTERNAL_SERVICE_ERROR: "SYSTEM_EXTERNAL_SERVICE_ERROR"
};
```

## Error Handling Patterns

### Service Layer
```javascript
class UserService {
  async getUserById(userId) {
    try {
      const user = await UserModel.findById(userId);
      
      if (!user) {
        throw new NotFoundError("User", userId);
      }
      
      return user;
    } catch (error) {
      // Re-throw known errors
      if (error instanceof BaseError) {
        throw error;
      }
      
      // Wrap unknown errors
      logger.error("Unexpected error fetching user", { userId, error });
      throw new DatabaseError(
        "Failed to fetch user",
        ERROR_CODES.SYSTEM_DATABASE_ERROR,
        { originalError: error.message }
      );
    }
  }
}
```

### Controller Layer
```javascript
async function getUser(request, reply) {
  try {
    const user = await userService.getUserById(request.params.userId);
    return reply.send({ success: true, data: user });
  } catch (error) {
    // Let error handler middleware handle it
    throw error;
  }
}
```

### API Error Handler Middleware
```javascript
function errorHandler(error, request, reply) {
  // Log all errors
  logger.error("Request failed", {
    error: {
      message: error.message,
      code: error.code,
      stack: error.stack,
      details: error.details
    },
    request: {
      method: request.method,
      url: request.url,
      params: request.params,
      query: request.query
    }
  });

  // Handle known errors
  if (error instanceof BaseError) {
    return reply.status(error.statusCode).send({
      success: false,
      error: {
        code: error.code,
        message: error.message,
        details: error.details,
        timestamp: error.timestamp
      }
    });
  }

  // Handle validation errors (from schemas)
  if (error.validation) {
    return reply.status(400).send({
      success: false,
      error: {
        code: ERROR_CODES.VALIDATION_INVALID_FORMAT,
        message: "Validation failed",
        details: error.validation
      }
    });
  }

  // Handle unknown errors
  return reply.status(500).send({
    success: false,
    error: {
      code: ERROR_CODES.SYSTEM_INTERNAL_ERROR,
      message: "An unexpected error occurred",
      timestamp: new Date().toISOString()
    }
  });
}
```

## Async Error Handling

### Promise Chains
```javascript
// BAD - Error might be swallowed
someAsyncOperation()
  .then(result => processResult(result))
  .then(processed => saveToDatabase(processed));

// GOOD - Explicit error handling
someAsyncOperation()
  .then(result => processResult(result))
  .then(processed => saveToDatabase(processed))
  .catch(error => {
    logger.error("Operation failed", { error });
    throw new OperationError("Failed to complete operation", { cause: error.message });
  });
```

### Async/Await
```javascript
// GOOD - Try/catch with proper error transformation
async function processUserData(userId) {
  let user;
  let processedData;
  
  try {
    user = await getUserById(userId);
  } catch (error) {
    throw new DataFetchError(
      `Failed to fetch user ${userId}`,
      ERROR_CODES.USER_FETCH_FAILED,
      { userId, originalError: error.message }
    );
  }
  
  try {
    processedData = await processData(user);
  } catch (error) {
    throw new ProcessingError(
      `Failed to process user data`,
      ERROR_CODES.DATA_PROCESSING_FAILED,
      { userId, stage: "processing" }
    );
  }
  
  return processedData;
}
```

## Logging Errors

### What to Log
```javascript
logger.error("Descriptive error message", {
  // Context about the operation
  operation: "createUser",
  userId: user.id,
  
  // Error details
  error: {
    message: error.message,
    code: error.code,
    stack: error.stack,
    type: error.constructor.name
  },
  
  // Request context (if applicable)
  request: {
    method: req.method,
    path: req.path,
    ip: req.ip
  },
  
  // Additional debugging info
  metadata: {
    timestamp: new Date().toISOString(),
    environment: process.env.NODE_ENV
  }
});
```

### Log Levels for Errors
- **error**: Actual errors that need attention
- **warn**: Recoverable issues or potential problems
- **info**: Expected failures (e.g., validation failures)

## Database Error Handling

### Connection Errors
```javascript
async function connectToDatabase() {
  const maxRetries = 3;
  let retries = 0;
  
  while (retries < maxRetries) {
    try {
      await mongoose.connect(uri, options);
      logger.info("Database connected successfully");
      return;
    } catch (error) {
      retries++;
      
      if (retries === maxRetries) {
        throw new DatabaseError(
          "Failed to connect to database after maximum retries",
          ERROR_CODES.DATABASE_CONNECTION_FAILED,
          { retries: maxRetries, lastError: error.message }
        );
      }
      
      logger.warn(`Database connection attempt ${retries} failed, retrying...`, { error: error.message });
      await new Promise(resolve => setTimeout(resolve, 5000 * retries));
    }
  }
}
```

### Query Errors
```javascript
async function findUserByEmail(email) {
  try {
    return await UserModel.findOne({ email }).exec();
  } catch (error) {
    // Check for specific MongoDB errors
    if (error.name === 'CastError') {
      throw new ValidationError(
        "Invalid email format",
        ERROR_CODES.VALIDATION_INVALID_FORMAT,
        { field: "email", value: email }
      );
    }
    
    throw new DatabaseError(
      "Database query failed",
      ERROR_CODES.DATABASE_QUERY_FAILED,
      { operation: "findUserByEmail", error: error.message }
    );
  }
}
```

## External Service Errors

### HTTP Client Errors
```javascript
async function callExternalAPI(endpoint, data) {
  try {
    const response = await axios.post(endpoint, data, {
      timeout: 30000
    });
    
    return response.data;
  } catch (error) {
    // Axios error
    if (error.response) {
      throw new ExternalServiceError(
        `External API returned error: ${error.response.statusText}`,
        ERROR_CODES.EXTERNAL_API_ERROR,
        {
          status: error.response.status,
          data: error.response.data,
          endpoint
        }
      );
    }
    
    // Network error
    if (error.request) {
      throw new NetworkError(
        "Network error calling external service",
        ERROR_CODES.NETWORK_ERROR,
        { endpoint, timeout: error.code === 'ECONNABORTED' }
      );
    }
    
    // Unknown error
    throw new SystemError(
      "Unexpected error calling external service",
      ERROR_CODES.SYSTEM_EXTERNAL_SERVICE_ERROR,
      { endpoint, error: error.message }
    );
  }
}
```

## Worker Process Errors

### Job Processing
```javascript
class JobProcessor {
  async processJob(job) {
    const startTime = Date.now();
    
    try {
      logger.info("Starting job processing", { jobId: job.id, type: job.type });
      
      const result = await this._executeJob(job);
      
      logger.info("Job completed successfully", {
        jobId: job.id,
        duration: Date.now() - startTime
      });
      
      return result;
    } catch (error) {
      logger.error("Job processing failed", {
        jobId: job.id,
        type: job.type,
        duration: Date.now() - startTime,
        error: {
          message: error.message,
          code: error.code,
          stack: error.stack
        }
      });
      
      // Update job status
      await this._markJobAsFailed(job, error);
      
      // Re-throw for caller to handle
      throw error;
    }
  }
  
  async _markJobAsFailed(job, error) {
    try {
      await JobModel.updateOne(
        { _id: job.id },
        {
          status: JOB_STATUS_FAILED,
          error: {
            message: error.message,
            code: error.code,
            timestamp: new Date()
          },
          failedAt: new Date()
        }
      );
    } catch (updateError) {
      // Critical: couldn't update job status
      logger.error("CRITICAL: Failed to update job status", {
        jobId: job.id,
        originalError: error.message,
        updateError: updateError.message
      });
    }
  }
}
```

## Best Practices

### DO:
- ✅ Create specific error classes for different domains
- ✅ Include relevant context in error details
- ✅ Log errors with structured data
- ✅ Use error codes for programmatic handling
- ✅ Wrap third-party errors in your own error types
- ✅ Include timestamps in errors
- ✅ Test error scenarios explicitly

### DON'T:
- ❌ Catch errors without re-throwing or logging
- ❌ Use generic error messages
- ❌ Expose internal details in production error responses
- ❌ Mix error handling logic with business logic
- ❌ Use string matching for error identification
- ❌ Log sensitive data in error messages

## Testing Error Scenarios

```javascript
describe("UserService", () => {
  describe("getUserById", () => {
    it("should throw NotFoundError when user doesn't exist", async () => {
      const nonExistentId = "507f1f77bcf86cd799439011";
      
      await expect(userService.getUserById(nonExistentId))
        .rejects
        .toThrow(NotFoundError);
    });
    
    it("should throw DatabaseError on connection failure", async () => {
      // Mock database failure
      jest.spyOn(UserModel, 'findById').mockRejectedValue(new Error("Connection lost"));
      
      await expect(userService.getUserById("validId"))
        .rejects
        .toThrow(DatabaseError);
    });
  });
});
```

## Remember
**Every error should tell a story: what went wrong, where it happened, and what context is needed to fix it.**