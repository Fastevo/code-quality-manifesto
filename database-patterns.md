# Database Patterns Guidelines

## Core Principles

1. **Three-Database Architecture**: Separate databases for core, analytics, and system logs
2. **Mongoose Best Practices**: Use schema validation, indexes, and middleware
3. **Caching First**: All queries should support caching
4. **Connection Pooling**: Reuse connections across services
5. **Schema Versioning**: Plan for migration and backward compatibility

## Three-Database Pattern

### Database Separation
```
core-db/          # Users, organizations, projects, auth
analytics-db/     # Metrics, usage stats, playback data
systemlogs-db/    # Application logs, errors, audit trails
```

### Connection Pattern
```javascript
// db/coreDb.js
import mongoose from "mongoose";
import logger from "../utils/logger.js";

let coreDbConnection;

export async function connectToCoreDb() {
  if (coreDbConnection?.readyState === 1) {
    return coreDbConnection;
  }

  try {
    coreDbConnection = await mongoose.createConnection(
      configProvider.CORE_DB_URI,
      {
        maxPoolSize: configProvider.CORE_DB_POOL_SIZE || 10,
        minPoolSize: configProvider.CORE_DB_MIN_POOL_SIZE || 2,
        serverSelectionTimeoutMS: 30000,
        socketTimeoutMS: 45000,
        family: 4 // Force IPv4
      }
    );

    coreDbConnection.on('error', (error) => {
      logger.error('Core database connection error', { error });
    });

    coreDbConnection.on('disconnected', () => {
      logger.warn('Core database disconnected');
    });

    logger.info('Core database connected successfully');
    return coreDbConnection;
  } catch (error) {
    logger.error('Failed to connect to core database', { error });
    throw error;
  }
}

export function getCoreDbConnection() {
  if (!coreDbConnection || coreDbConnection.readyState !== 1) {
    throw new Error('Core database not connected');
  }
  return coreDbConnection;
}
```

### Model Registration Pattern
```javascript
// db/coreDbModels.js
import { getCoreDbConnection } from "./coreDb.js";

// Import schemas from commons
import UserSchema from "@visiontsnpm/core-commons/schemas/UserSchema.js";
import OrganizationSchema from "@visiontsnpm/core-commons/schemas/OrganizationSchema.js";
import ProjectSchema from "@visiontsnpm/core-commons/schemas/ProjectSchema.js";

// Register models with connection
export function getCoreModels() {
  const connection = getCoreDbConnection();
  
  return {
    User: connection.models.User || connection.model("User", UserSchema),
    Organization: connection.models.Organization || connection.model("Organization", OrganizationSchema),
    Project: connection.models.Project || connection.model("Project", ProjectSchema),
    // ... other models
  };
}

// Export individual models for convenience
export const UserModel = () => getCoreModels().User;
export const OrganizationModel = () => getCoreModels().Organization;
export const ProjectModel = () => getCoreModels().Project;
```

## Schema Design Patterns

### Base Schema Pattern
```javascript
import mongoose from "mongoose";
import validator from "validator";

const baseSchemaOptions = {
  timestamps: true, // Adds createdAt and updatedAt
  toJSON: {
    virtuals: true,
    transform: (doc, ret) => {
      delete ret.__v;
      return ret;
    }
  },
  toObject: {
    virtuals: true
  }
};

export default baseSchemaOptions;
```

### Schema with Validation
```javascript
const UserSchema = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true,
    validate: [validator.isEmail, "Invalid email format"]
  },
  
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true,
    minlength: [3, "Username must be at least 3 characters"],
    maxlength: [30, "Username cannot exceed 30 characters"],
    match: [/^[a-zA-Z0-9_-]+$/, "Username can only contain letters, numbers, underscores, and hyphens"]
  },
  
  role: {
    type: String,
    enum: USER_ROLE_ENUM,
    default: USER_ROLE_ENUM[0]
  },
  
  status: {
    type: String,
    enum: USER_STATUS_ENUM,
    default: USER_STATUS_ACTIVE
  },
  
  metadata: {
    type: mongoose.Schema.Types.Mixed,
    default: {}
  }
}, baseSchemaOptions);

// Indexes
UserSchema.index({ email: 1 });
UserSchema.index({ username: 1 });
UserSchema.index({ organizationId: 1, status: 1 });
UserSchema.index({ createdAt: -1 });

// Compound indexes for common queries
UserSchema.index({ organizationId: 1, role: 1, status: 1 });
```

### Schema Middleware
```javascript
// Pre-save middleware
UserSchema.pre('save', async function(next) {
  // Only hash password if modified
  if (!this.isModified('password')) return next();
  
  try {
    const salt = await bcrypt.genSalt(10);
    this.password = await bcrypt.hash(this.password, salt);
    next();
  } catch (error) {
    next(error);
  }
});

// Post-save middleware
UserSchema.post('save', async function(doc) {
  logger.info('User created', { userId: doc._id, email: doc.email });
});

// Pre-remove middleware
UserSchema.pre('remove', async function(next) {
  // Clean up related data
  await TokenModel.deleteMany({ userId: this._id });
  await SessionModel.deleteMany({ userId: this._id });
  next();
});
```

### Virtual Properties
```javascript
// Add computed properties that don't get stored
UserSchema.virtual('fullName').get(function() {
  return `${this.firstName} ${this.lastName}`;
});

UserSchema.virtual('isActive').get(function() {
  return this.status === USER_STATUS_ACTIVE;
});

// Virtual with setter
UserSchema.virtual('displayName')
  .get(function() {
    return this.nickname || this.username;
  })
  .set(function(value) {
    this.nickname = value;
  });
```

## Caching Patterns

### Smart Cache Implementation
```javascript
// db/cachingUtils.js
import mongoose from "mongoose";
import Keyv from "keyv";
import QuickLRU from "quick-lru";

const lru = new QuickLRU({ maxSize: 1000 });
const cache = new Keyv({ store: lru });

// Extend Mongoose Query with caching
mongoose.Query.prototype.withCache = function(options = {}) {
  this._cacheOptions = {
    ttl: options.ttl || 300, // 5 minutes default
    key: options.key || null,
    compress: options.compress || false
  };
  
  return this;
};

// Override exec to implement caching
const originalExec = mongoose.Query.prototype.exec;

mongoose.Query.prototype.exec = async function() {
  if (!this._cacheOptions || !configProvider.CACHING_STRATEGY_ENABLED) {
    return originalExec.apply(this);
  }
  
  // Generate cache key
  const key = this._cacheOptions.key || this.getCacheKey();
  
  // Check cache
  const cached = await cache.get(key);
  if (cached) {
    return Array.isArray(cached) 
      ? cached.map(doc => this.model.hydrate(doc))
      : this.model.hydrate(cached);
  }
  
  // Execute query
  const result = await originalExec.apply(this);
  
  // Cache result
  if (result) {
    const dataToCache = Array.isArray(result)
      ? result.map(doc => doc.toObject())
      : result.toObject();
      
    await cache.set(key, dataToCache, this._cacheOptions.ttl * 1000);
  }
  
  return result;
};

mongoose.Query.prototype.getCacheKey = function() {
  return JSON.stringify({
    collection: this.model.collection.name,
    op: this.op,
    filter: this.getFilter(),
    options: this.getOptions(),
    populate: this.getPopulatedPaths()
  });
};
```

### Cache Usage Patterns
```javascript
// Simple caching
const user = await UserModel
  .findById(userId)
  .withCache({ ttl: 3600 }); // 1 hour

// Custom cache key
const activeUsers = await UserModel
  .find({ status: 'active', organizationId })
  .withCache({ 
    key: `org:${organizationId}:active-users`,
    ttl: 600 // 10 minutes
  });

// Disable cache for specific query
const freshData = await UserModel
  .findById(userId); // No withCache = no caching
```

## Query Patterns

### Pagination Pattern
```javascript
async function getPaginatedResults(model, filter, options = {}) {
  const {
    page = 1,
    limit = 20,
    sort = { createdAt: -1 },
    select,
    populate
  } = options;
  
  const skip = (page - 1) * limit;
  
  // Execute count and data queries in parallel
  const [totalCount, results] = await Promise.all([
    model.countDocuments(filter),
    model
      .find(filter)
      .sort(sort)
      .skip(skip)
      .limit(limit)
      .select(select)
      .populate(populate)
      .withCache({ 
        ttl: 300,
        key: `${model.modelName}:page:${page}:${JSON.stringify(filter)}`
      })
  ]);
  
  return {
    results,
    pagination: {
      page,
      limit,
      totalCount,
      totalPages: Math.ceil(totalCount / limit),
      hasNextPage: page * limit < totalCount,
      hasPrevPage: page > 1
    }
  };
}
```

### Aggregation Pattern
```javascript
async function getProjectStats(projectId) {
  const stats = await AnalyticsModel.aggregate([
    // Match stage
    { $match: { projectId: mongoose.Types.ObjectId(projectId) } },
    
    // Date range filter
    {
      $match: {
        createdAt: {
          $gte: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) // Last 30 days
        }
      }
    },
    
    // Group by day
    {
      $group: {
        _id: {
          $dateToString: { format: "%Y-%m-%d", date: "$createdAt" }
        },
        totalViews: { $sum: 1 },
        uniqueUsers: { $addToSet: "$userId" },
        totalDuration: { $sum: "$duration" }
      }
    },
    
    // Transform the output
    {
      $project: {
        date: "$_id",
        totalViews: 1,
        uniqueUsers: { $size: "$uniqueUsers" },
        totalDuration: 1,
        _id: 0
      }
    },
    
    // Sort by date
    { $sort: { date: 1 } }
  ]);
  
  return stats;
}
```

### Bulk Operations Pattern
```javascript
async function bulkUpdateUsers(updates) {
  const bulkOps = updates.map(update => ({
    updateOne: {
      filter: { _id: update.userId },
      update: { $set: update.data },
      upsert: false
    }
  }));
  
  const result = await UserModel.bulkWrite(bulkOps, {
    ordered: false, // Continue on error
    writeConcern: { w: 'majority' }
  });
  
  logger.info('Bulk update completed', {
    matched: result.matchedCount,
    modified: result.modifiedCount,
    errors: result.writeErrors
  });
  
  return result;
}
```

## Transaction Patterns

### Multi-Document Transactions
```javascript
async function transferCredits(fromUserId, toUserId, amount) {
  const session = await mongoose.startSession();
  
  try {
    await session.withTransaction(async () => {
      // Deduct from sender
      const sender = await UserModel.findOneAndUpdate(
        { 
          _id: fromUserId,
          credits: { $gte: amount } // Ensure sufficient balance
        },
        { $inc: { credits: -amount } },
        { session, new: true }
      );
      
      if (!sender) {
        throw new InsufficientCreditsError('Insufficient credits');
      }
      
      // Add to receiver
      const receiver = await UserModel.findByIdAndUpdate(
        toUserId,
        { $inc: { credits: amount } },
        { session, new: true }
      );
      
      if (!receiver) {
        throw new NotFoundError('Receiver not found');
      }
      
      // Log transaction
      await TransactionLogModel.create([{
        type: 'credit_transfer',
        fromUserId,
        toUserId,
        amount,
        timestamp: new Date()
      }], { session });
    });
    
    logger.info('Credit transfer successful', { fromUserId, toUserId, amount });
  } finally {
    await session.endSession();
  }
}
```

## Migration Patterns

### Schema Versioning
```javascript
const SchemaWithVersion = new mongoose.Schema({
  // ... fields
  schemaVersion: {
    type: Number,
    default: 1,
    required: true
  }
});

// Migration middleware
SchemaWithVersion.pre('save', async function(next) {
  if (this.schemaVersion < CURRENT_SCHEMA_VERSION) {
    await migrateDocument(this);
  }
  next();
});

async function migrateDocument(doc) {
  while (doc.schemaVersion < CURRENT_SCHEMA_VERSION) {
    const migrator = migrations[doc.schemaVersion];
    if (migrator) {
      await migrator(doc);
      doc.schemaVersion++;
    } else {
      break;
    }
  }
}
```

### Safe Field Addition
```javascript
// Adding new required field with migration
async function addRequiredField() {
  // 1. Add field as optional
  await UserModel.updateMany(
    { newField: { $exists: false } },
    { $set: { newField: 'default_value' } }
  );
  
  // 2. Update schema to make it required
  // 3. Deploy new code
}
```

## Performance Patterns

### Index Strategy
```javascript
// Analyze slow queries
async function analyzeSlowQueries() {
  const result = await mongoose.connection.db
    .collection('system.profile')
    .find({ millis: { $gt: 100 } })
    .sort({ ts: -1 })
    .limit(10)
    .toArray();
    
  result.forEach(query => {
    logger.warn('Slow query detected', {
      collection: query.ns,
      duration: query.millis,
      command: query.command,
      planSummary: query.planSummary
    });
  });
}

// Create indexes based on query patterns
const indexes = [
  { fields: { userId: 1, createdAt: -1 }, options: { name: 'user_recent' } },
  { fields: { status: 1, priority: -1 }, options: { name: 'status_priority' } },
  { fields: { 'metadata.category': 1 }, options: { sparse: true } }
];

for (const index of indexes) {
  await Model.createIndex(index.fields, index.options);
}
```

### Connection Pooling Best Practices
```javascript
const connectionOptions = {
  maxPoolSize: 100,        // Maximum connections
  minPoolSize: 10,         // Minimum connections
  maxIdleTimeMS: 30000,    // Close idle connections after 30s
  waitQueueTimeoutMS: 5000, // Timeout waiting for connection
  serverSelectionTimeoutMS: 30000,
  socketTimeoutMS: 45000,
  family: 4,               // Use IPv4
  
  // Connection monitoring
  heartbeatFrequencyMS: 10000,
  minHeartbeatFrequencyMS: 500
};
```

## Testing Database Patterns

### Test Database Setup
```javascript
// tests/setup/database.js
import mongoose from "mongoose";
import { MongoMemoryServer } from "mongodb-memory-server";

let mongoServer;

export async function setupTestDatabase() {
  mongoServer = await MongoMemoryServer.create();
  const uri = mongoServer.getUri();
  
  await mongoose.connect(uri, {
    useNewUrlParser: true,
    useUnifiedTopology: true
  });
}

export async function teardownTestDatabase() {
  await mongoose.disconnect();
  await mongoServer.stop();
}

export async function clearDatabase() {
  const collections = mongoose.connection.collections;
  
  for (const key in collections) {
    await collections[key].deleteMany({});
  }
}
```

### Test Data Factories
```javascript
// tests/factories/userFactory.js
import { faker } from "@faker-js/faker";

export function createUserData(overrides = {}) {
  return {
    email: faker.internet.email(),
    username: faker.internet.userName(),
    firstName: faker.person.firstName(),
    lastName: faker.person.lastName(),
    role: USER_ROLE_MEMBER,
    status: USER_STATUS_ACTIVE,
    ...overrides
  };
}

export async function createUser(overrides = {}) {
  const userData = createUserData(overrides);
  return await UserModel.create(userData);
}

export async function createUsers(count, overrides = {}) {
  const users = [];
  for (let i = 0; i < count; i++) {
    users.push(await createUser(overrides));
  }
  return users;
}
```

## Best Practices

### DO:
- ✅ Use connection pooling appropriately
- ✅ Create indexes for all query patterns
- ✅ Use transactions for multi-document operations
- ✅ Implement caching for read-heavy operations
- ✅ Use aggregation pipeline for complex queries
- ✅ Add schema versioning for migrations
- ✅ Use bulk operations for batch updates
- ✅ Monitor slow queries and optimize

### DON'T:
- ❌ Use unbounded queries without pagination
- ❌ Create too many indexes (write performance impact)
- ❌ Store large binary data in documents
- ❌ Use complex nested documents (prefer references)
- ❌ Skip validation at schema level
- ❌ Ignore connection errors
- ❌ Mix database concerns across services

## Remember
**The three-database pattern provides clear separation of concerns. Always use caching, proper indexes, and transactions where appropriate.**