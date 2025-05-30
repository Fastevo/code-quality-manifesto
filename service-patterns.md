# Service Pattern Guidelines

## Core Principles

1. **Single Responsibility**: Each service handles one business domain
2. **Dependency Injection**: Services receive dependencies through constructors
3. **Stateless Design**: Services don't maintain request-specific state
4. **Clear Interfaces**: Public methods define the service contract
5. **Testability First**: Design for easy mocking and testing

## Service Architecture

### Service Class Structure
```javascript
import BaseService from "./BaseService.js";
import { ValidationError, NotFoundError } from "../errors/index.js";
import logger from "../utils/logger.js";

export default class UserService extends BaseService {
  constructor({ 
    userModel, 
    emailService, 
    cacheService,
    configProvider 
  }) {
    super();
    
    // Dependency injection
    this._userModel = userModel;
    this._emailService = emailService;
    this._cacheService = cacheService;
    this._config = configProvider;
    
    // Service configuration
    this._cachePrefix = "user:";
    this._cacheTTL = 3600; // 1 hour
  }
  
  // Public interface methods
  async createUser(userData) {
    // Implementation
  }
  
  async getUserById(userId) {
    // Implementation
  }
  
  // Private helper methods
  async _validateUserData(userData) {
    // Implementation
  }
  
  _sanitizeUserObject(user) {
    // Implementation
  }
}
```

### Base Service Pattern
```javascript
export default class BaseService {
  constructor() {
    if (new.target === BaseService) {
      throw new Error("BaseService cannot be instantiated directly");
    }
  }
  
  // Common logging method
  _logOperation(operation, data = {}) {
    logger.info(`${this.constructor.name}.${operation}`, data);
  }
  
  // Common error logging
  _logError(operation, error, context = {}) {
    logger.error(`${this.constructor.name}.${operation} failed`, {
      error: {
        message: error.message,
        code: error.code,
        stack: error.stack
      },
      context
    });
  }
}
```

## Service Instantiation

### Singleton Pattern
```javascript
// In services/index.js
import UserService from "./UserService.js";
import EmailService from "./EmailService.js";
import CacheService from "./CacheService.js";
import configProvider from "../config/configProvider.js";
import { UserModel } from "../models/index.js";

// Create singleton instances
const cacheService = new CacheService({ configProvider });
const emailService = new EmailService({ configProvider });
const userService = new UserService({
  userModel: UserModel,
  emailService,
  cacheService,
  configProvider
});

// Export instances
export {
  userService,
  emailService,
  cacheService
};
```

### Factory Pattern (Alternative)
```javascript
export class ServiceFactory {
  static _instances = new Map();
  
  static create(ServiceClass, dependencies) {
    const className = ServiceClass.name;
    
    if (!this._instances.has(className)) {
      this._instances.set(className, new ServiceClass(dependencies));
    }
    
    return this._instances.get(className);
  }
  
  static reset() {
    this._instances.clear();
  }
}
```

## Common Service Patterns

### CRUD Service Pattern
```javascript
export default class ResourceService extends BaseService {
  constructor({ model, cacheService }) {
    super();
    this._model = model;
    this._cache = cacheService;
  }
  
  async create(data) {
    this._logOperation("create", { dataKeys: Object.keys(data) });
    
    try {
      // Validate
      await this._validateData(data);
      
      // Create
      const resource = await this._model.create(data);
      
      // Cache
      await this._cache.set(
        this._getCacheKey(resource.id),
        resource,
        this._cacheTTL
      );
      
      return this._sanitizeObject(resource);
    } catch (error) {
      this._logError("create", error, { data });
      throw this._transformError(error);
    }
  }
  
  async findById(id) {
    this._logOperation("findById", { id });
    
    // Check cache first
    const cached = await this._cache.get(this._getCacheKey(id));
    if (cached) {
      return cached;
    }
    
    // Fetch from database
    const resource = await this._model.findById(id);
    
    if (!resource) {
      throw new NotFoundError(this._model.modelName, id);
    }
    
    // Cache for next time
    await this._cache.set(
      this._getCacheKey(id),
      resource,
      this._cacheTTL
    );
    
    return this._sanitizeObject(resource);
  }
  
  async update(id, updates) {
    this._logOperation("update", { id, updateKeys: Object.keys(updates) });
    
    try {
      // Validate updates
      await this._validateData(updates, true);
      
      // Update
      const resource = await this._model.findByIdAndUpdate(
        id,
        updates,
        { new: true, runValidators: true }
      );
      
      if (!resource) {
        throw new NotFoundError(this._model.modelName, id);
      }
      
      // Invalidate cache
      await this._cache.delete(this._getCacheKey(id));
      
      return this._sanitizeObject(resource);
    } catch (error) {
      this._logError("update", error, { id, updates });
      throw this._transformError(error);
    }
  }
  
  async delete(id) {
    this._logOperation("delete", { id });
    
    const resource = await this._model.findByIdAndDelete(id);
    
    if (!resource) {
      throw new NotFoundError(this._model.modelName, id);
    }
    
    // Invalidate cache
    await this._cache.delete(this._getCacheKey(id));
    
    return { success: true, id };
  }
  
  // Helper methods
  _getCacheKey(id) {
    return `${this._model.modelName.toLowerCase()}:${id}`;
  }
  
  _sanitizeObject(obj) {
    const sanitized = obj.toObject ? obj.toObject() : obj;
    delete sanitized.__v;
    return sanitized;
  }
  
  async _validateData(data, isUpdate = false) {
    // Implement validation logic
  }
  
  _transformError(error) {
    // Transform database errors to domain errors
    if (error.name === 'ValidationError') {
      return new ValidationError(error.message, "VALIDATION_FAILED", {
        fields: Object.keys(error.errors)
      });
    }
    return error;
  }
}
```

### External API Service Pattern
```javascript
export default class ExternalAPIService extends BaseService {
  constructor({ httpClient, configProvider, retryService }) {
    super();
    this._client = httpClient;
    this._config = configProvider;
    this._retry = retryService;
    
    // Configuration
    this._baseURL = this._config.EXTERNAL_API_BASE_URL;
    this._apiKey = this._config.EXTERNAL_API_KEY;
    this._timeout = this._config.EXTERNAL_API_TIMEOUT_MS || 30000;
  }
  
  async fetchResource(resourceId) {
    return this._retry.execute(
      async () => {
        const response = await this._client.get(
          `${this._baseURL}/resources/${resourceId}`,
          {
            headers: this._getHeaders(),
            timeout: this._timeout
          }
        );
        
        return this._transformResponse(response.data);
      },
      {
        retries: 3,
        delay: 1000,
        backoff: 2,
        shouldRetry: (error) => {
          return error.response?.status >= 500 || error.code === 'ETIMEDOUT';
        }
      }
    );
  }
  
  _getHeaders() {
    return {
      'Authorization': `Bearer ${this._apiKey}`,
      'Content-Type': 'application/json',
      'User-Agent': 'Internal-Service/1.0'
    };
  }
  
  _transformResponse(data) {
    // Transform external API response to internal format
    return {
      id: data.external_id,
      name: data.resource_name,
      status: this._mapStatus(data.status),
      metadata: data.extra_info
    };
  }
  
  _mapStatus(externalStatus) {
    const statusMap = {
      'active': 'ACTIVE',
      'inactive': 'INACTIVE',
      'pending': 'PENDING'
    };
    
    return statusMap[externalStatus] || 'UNKNOWN';
  }
}
```

### Queue Processing Service Pattern
```javascript
export default class JobProcessorService extends BaseService {
  constructor({ jobModel, workerService, metricsService }) {
    super();
    this._jobModel = jobModel;
    this._workerService = workerService;
    this._metrics = metricsService;
    
    this._isProcessing = false;
    this._shouldStop = false;
  }
  
  async start() {
    if (this._isProcessing) {
      throw new Error("Job processor is already running");
    }
    
    this._isProcessing = true;
    this._shouldStop = false;
    
    this._logOperation("start");
    
    while (!this._shouldStop) {
      try {
        await this._processNextJob();
      } catch (error) {
        this._logError("processNextJob", error);
        await this._waitBeforeRetry();
      }
    }
    
    this._isProcessing = false;
  }
  
  async stop() {
    this._logOperation("stop");
    this._shouldStop = true;
  }
  
  async _processNextJob() {
    // Fetch next job
    const job = await this._jobModel.findOneAndUpdate(
      { 
        status: JOB_STATUS_PENDING,
        scheduledAt: { $lte: new Date() }
      },
      { 
        status: JOB_STATUS_PROCESSING,
        startedAt: new Date()
      },
      { 
        new: true,
        sort: { priority: -1, createdAt: 1 }
      }
    );
    
    if (!job) {
      await this._waitForJobs();
      return;
    }
    
    const startTime = Date.now();
    
    try {
      // Process job
      const result = await this._workerService.process(job);
      
      // Update job status
      await this._jobModel.findByIdAndUpdate(job.id, {
        status: JOB_STATUS_COMPLETED,
        completedAt: new Date(),
        result,
        processingTime: Date.now() - startTime
      });
      
      // Record metrics
      this._metrics.recordJobSuccess(job.type, Date.now() - startTime);
      
    } catch (error) {
      await this._handleJobError(job, error);
      this._metrics.recordJobFailure(job.type, error.code);
    }
  }
  
  async _handleJobError(job, error) {
    const updates = {
      lastError: {
        message: error.message,
        code: error.code,
        timestamp: new Date()
      },
      failureCount: (job.failureCount || 0) + 1
    };
    
    if (updates.failureCount >= MAX_RETRIES) {
      updates.status = JOB_STATUS_FAILED;
      updates.failedAt = new Date();
    } else {
      updates.status = JOB_STATUS_PENDING;
      updates.scheduledAt = this._calculateRetryTime(updates.failureCount);
    }
    
    await this._jobModel.findByIdAndUpdate(job.id, updates);
  }
  
  _calculateRetryTime(attemptNumber) {
    const delayMinutes = Math.pow(2, attemptNumber); // Exponential backoff
    return new Date(Date.now() + delayMinutes * 60 * 1000);
  }
  
  async _waitForJobs() {
    await new Promise(resolve => setTimeout(resolve, 5000));
  }
  
  async _waitBeforeRetry() {
    await new Promise(resolve => setTimeout(resolve, 10000));
  }
}
```

## Service Communication

### Inter-Service Communication
```javascript
// Services should communicate through well-defined interfaces
export default class OrderService extends BaseService {
  constructor({ orderModel, userService, paymentService, notificationService }) {
    super();
    this._orderModel = orderModel;
    this._userService = userService;
    this._paymentService = paymentService;
    this._notificationService = notificationService;
  }
  
  async createOrder(userId, orderData) {
    // Get user through service interface
    const user = await this._userService.getUserById(userId);
    
    // Validate user can create orders
    if (!user.canCreateOrders) {
      throw new ForbiddenError("User cannot create orders");
    }
    
    // Create order
    const order = await this._orderModel.create({
      userId,
      ...orderData,
      status: ORDER_STATUS_PENDING
    });
    
    try {
      // Process payment through service
      const payment = await this._paymentService.processPayment({
        amount: order.total,
        userId,
        orderId: order.id
      });
      
      // Update order with payment info
      order.paymentId = payment.id;
      order.status = ORDER_STATUS_PAID;
      await order.save();
      
      // Send notification through service
      await this._notificationService.sendOrderConfirmation(user.email, order);
      
    } catch (error) {
      // Rollback on failure
      await this._rollbackOrder(order.id);
      throw error;
    }
    
    return order;
  }
}
```

## Testing Services

### Service Test Structure
```javascript
describe("UserService", () => {
  let userService;
  let mockUserModel;
  let mockEmailService;
  let mockCacheService;
  
  beforeEach(() => {
    // Create mocks
    mockUserModel = {
      create: jest.fn(),
      findById: jest.fn(),
      findOne: jest.fn()
    };
    
    mockEmailService = {
      sendWelcomeEmail: jest.fn(),
      sendPasswordReset: jest.fn()
    };
    
    mockCacheService = {
      get: jest.fn(),
      set: jest.fn(),
      delete: jest.fn()
    };
    
    // Create service instance with mocks
    userService = new UserService({
      userModel: mockUserModel,
      emailService: mockEmailService,
      cacheService: mockCacheService,
      configProvider: { CACHE_TTL: 3600 }
    });
  });
  
  describe("createUser", () => {
    it("should create user and send welcome email", async () => {
      const userData = {
        email: "test@example.com",
        name: "Test User"
      };
      
      const createdUser = { id: "123", ...userData };
      mockUserModel.create.mockResolvedValue(createdUser);
      mockEmailService.sendWelcomeEmail.mockResolvedValue(true);
      
      const result = await userService.createUser(userData);
      
      expect(mockUserModel.create).toHaveBeenCalledWith(userData);
      expect(mockEmailService.sendWelcomeEmail).toHaveBeenCalledWith(userData.email);
      expect(mockCacheService.set).toHaveBeenCalledWith(
        "user:123",
        createdUser,
        3600
      );
      expect(result).toEqual(createdUser);
    });
    
    it("should throw ValidationError for invalid data", async () => {
      const invalidData = { email: "invalid-email" };
      
      await expect(userService.createUser(invalidData))
        .rejects
        .toThrow(ValidationError);
    });
  });
});
```

## Best Practices

### DO:
- ✅ Use dependency injection for all external dependencies
- ✅ Keep services stateless
- ✅ Define clear public interfaces
- ✅ Use private methods for internal logic
- ✅ Log all operations with context
- ✅ Handle errors consistently
- ✅ Design for testability
- ✅ Use caching where appropriate
- ✅ Implement retry logic for external calls

### DON'T:
- ❌ Import models directly in services
- ❌ Mix HTTP concerns with business logic
- ❌ Create circular dependencies between services
- ❌ Store request-specific state in services
- ❌ Use static methods for stateful operations
- ❌ Expose internal implementation details

## Service Registry Pattern (Advanced)
```javascript
export class ServiceRegistry {
  constructor() {
    this._services = new Map();
    this._dependencies = new Map();
  }
  
  register(name, ServiceClass, dependencies = {}) {
    this._dependencies.set(name, dependencies);
    
    // Lazy instantiation
    Object.defineProperty(this, name, {
      get: () => {
        if (!this._services.has(name)) {
          const resolvedDeps = this._resolveDependencies(dependencies);
          this._services.set(name, new ServiceClass(resolvedDeps));
        }
        return this._services.get(name);
      }
    });
  }
  
  _resolveDependencies(deps) {
    const resolved = {};
    
    for (const [key, value] of Object.entries(deps)) {
      if (typeof value === 'string' && this._dependencies.has(value)) {
        resolved[key] = this[value]; // Triggers lazy instantiation
      } else {
        resolved[key] = value;
      }
    }
    
    return resolved;
  }
}

// Usage
const registry = new ServiceRegistry();

registry.register('cacheService', CacheService, { configProvider });
registry.register('emailService', EmailService, { configProvider });
registry.register('userService', UserService, {
  userModel: UserModel,
  emailService: 'emailService', // Reference to registered service
  cacheService: 'cacheService',
  configProvider
});

export default registry;
```

## Remember
**Services are the heart of business logic. Keep them clean, testable, and focused on their single responsibility.**