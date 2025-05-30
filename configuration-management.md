# Configuration Management Guidelines

## Core Principles

1. **Multi-Tier Configuration**: Environment variables, database configs, and service-specific settings
2. **Inheritance Hierarchy**: System → Organization → Project level configs
3. **Security First**: Encrypt sensitive configurations
4. **Cache Everything**: Database configs must be cached
5. **Type Safety**: Parse and validate all configuration values

## Three-Tier Configuration System

### Tier 1: Environment Configuration (configProvider)

#### Standard Implementation
```javascript
// config/configProvider.js
import dotenvExtended from "dotenv-extended";
import dotenvParseVariables from "dotenv-parse-variables";

const configProvider = dotenvParseVariables(
  dotenvExtended.load({
    silent: true,
    errorOnMissing: false,
    errorOnExtra: true,
    includeProcessEnv: true,
    assignToProcessEnv: true,
    overrideProcessEnv: false,
    errorOnRegex: true,
  })
);

export default configProvider;
```

#### Environment Variables Categories
- **System Settings**: Ports, hosts, timeouts
- **Database Connections**: MongoDB URIs
- **Caching Settings**: TTL, strategies
- **Encryption Keys**: For sensitive config crypto
- **Feature Flags**: Enable/disable features

### Tier 2: Database Common Configuration

#### CommonConfig Schema
```javascript
const CommonConfigSchema = new mongoose.Schema({
  key: {
    type: String,
    required: true,
    unique: true,
    index: true
  },
  value: {
    type: mongoose.Schema.Types.Mixed,
    required: true
  },
  isTest: {
    type: Boolean,
    default: false
  }
}, {
  timestamps: true,
  collection: "commonconfigs"
});
```

#### Common Config Keys Pattern
```javascript
// In definitions/definitions.js
export const COMMON_CONFIG_KEY_API_HOSTNAME = "API_HOSTNAME";
export const COMMON_CONFIG_KEY_PRODUCT = "PRODUCT";
export const COMMON_CONFIG_KEY_ENVIRONMENT = "ENVIRONMENT";
export const COMMON_CONFIG_KEY_AWS_ACCESS_KEY_ID = "AWS_ACCESS_KEY_ID";
export const COMMON_CONFIG_KEY_AWS_SECRET_ACCESS_KEY = "AWS_SECRET_ACCESS_KEY";

// Keys that must be encrypted
export const COMMON_CONFIG_KEYS_THAT_ARE_ENCRYPTED = [
  COMMON_CONFIG_KEY_CLIENT_FACING_SIGNING_PRIVATE_KEY,
  COMMON_CONFIG_KEY_AWS_SECRET_ACCESS_KEY,
  // ... other sensitive keys
];
```

### Tier 3: Service-Specific Configuration

#### Service Config Models
```javascript
// Core service configs
const ConfigSchema = new mongoose.Schema({
  key: { type: String, unique: true, required: true },
  value: { type: mongoose.Schema.Types.Mixed, required: true },
  description: { type: String }
});

// Domain-specific configs (e.g., MediaProtection)
const MediaProtectionConfigSchema = new mongoose.Schema({
  key: { type: String, unique: true, required: true },
  value: { type: mongoose.Schema.Types.Mixed, required: true },
  isDeprecated: { type: Boolean, default: false }
});
```

## Configuration Service Pattern

### Base Configuration Service
```javascript
export default class BaseConfigService {
  constructor({ model, encryptionService, cacheService, configProvider }) {
    this._model = model;
    this._encryptionService = encryptionService;
    this._cacheService = cacheService;
    this._configTTL = configProvider.CONFIG_CACHE_TTL_SECONDS || 300;
  }

  async getConfigValue(key, failOnNotFound = true) {
    const config = await this.getConfig(key, failOnNotFound);
    return config?.value;
  }

  async getConfig(key, failOnNotFound = true) {
    // Check cache first
    const cached = await this._cacheService.get(`config:${key}`);
    if (cached) return cached;

    // Fetch from database
    const config = await this._model.findOne({ key }).withCache({
      ttl: this._configTTL
    });

    if (!config && failOnNotFound) {
      throw new NotFoundError(`Configuration key not found: ${key}`);
    }

    // Decrypt if necessary
    if (config && this._shouldDecrypt(key)) {
      config.value = await this._encryptionService.decrypt(config.value);
    }

    // Cache the result
    if (config) {
      await this._cacheService.set(`config:${key}`, config, this._configTTL);
    }

    return config;
  }

  async setConfig(key, value, options = {}) {
    // Encrypt if necessary
    if (this._shouldEncrypt(key)) {
      value = await this._encryptionService.encrypt(value);
    }

    const config = await this._model.findOneAndUpdate(
      { key },
      { 
        value,
        ...options
      },
      { 
        new: true, 
        upsert: true,
        runValidators: true
      }
    );

    // Invalidate cache
    await this._cacheService.delete(`config:${key}`);

    return config;
  }

  async deleteConfig(key, failOnNotFound = true) {
    const config = await this._model.findOneAndDelete({ key });

    if (!config && failOnNotFound) {
      throw new NotFoundError(`Configuration key not found: ${key}`);
    }

    // Invalidate cache
    await this._cacheService.delete(`config:${key}`);

    return config;
  }

  _shouldEncrypt(key) {
    // Override in subclasses
    return false;
  }

  _shouldDecrypt(key) {
    // Override in subclasses
    return false;
  }
}
```

### CommonConfig Service Implementation
```javascript
import { COMMON_CONFIG_KEYS_THAT_ARE_ENCRYPTED } from "../definitions/definitions.js";

export default class CommonConfigService extends BaseConfigService {
  constructor(dependencies) {
    super({
      ...dependencies,
      model: CommonConfigModel
    });
  }

  _shouldEncrypt(key) {
    return COMMON_CONFIG_KEYS_THAT_ARE_ENCRYPTED.includes(key);
  }

  _shouldDecrypt(key) {
    return COMMON_CONFIG_KEYS_THAT_ARE_ENCRYPTED.includes(key);
  }

  // Common config specific methods
  async getMultipleConfigs(keys) {
    const configs = await Promise.all(
      keys.map(key => this.getConfig(key, false))
    );
    
    return configs.reduce((acc, config) => {
      if (config) {
        acc[config.key] = config.value;
      }
      return acc;
    }, {});
  }
}
```

### Domain-Specific Config Service
```javascript
export default class MediaProtectionConfigService extends BaseConfigService {
  constructor(dependencies) {
    super({
      ...dependencies,
      model: MediaProtectionConfigModel
    });
    
    this._organizationModel = dependencies.organizationModel;
    this._projectModel = dependencies.projectModel;
  }

  // Domain-specific configuration methods
  async getProjectConfiguration(projectId) {
    const project = await this._projectModel
      .findById(projectId)
      .populate('mediaProtectionExtension')
      .withCache({ ttl: this._configTTL });

    if (!project?.mediaProtectionExtension) {
      throw new NotFoundError('MediaProtection not enabled for project');
    }

    return project.mediaProtectionExtension;
  }

  async getEffectivePlanForProject(projectId) {
    // Check project-level plan first
    const projectExtension = await this.getProjectConfiguration(projectId);
    if (projectExtension.plan) {
      return projectExtension.plan;
    }

    // Fall back to organization-level plan
    const project = await this._projectModel.findById(projectId);
    const orgExtension = await this._organizationModel
      .findById(project.organizationId)
      .select('mediaProtectionExtension.plan');

    if (orgExtension?.mediaProtectionExtension?.plan) {
      return orgExtension.mediaProtectionExtension.plan;
    }

    // Fall back to default plan
    return await this.getDefaultPlan();
  }
}
```

## Configuration Inheritance

### Hierarchy Levels
1. **System Level**: Global configurations
2. **Organization Level**: Organization-wide settings
3. **Project Level**: Project-specific overrides

### Implementation Pattern
```javascript
class HierarchicalConfigService {
  async getEffectiveConfig(key, projectId) {
    // 1. Check project level
    const projectConfig = await this.getProjectConfig(key, projectId);
    if (projectConfig !== null) return projectConfig;

    // 2. Check organization level
    const project = await ProjectModel.findById(projectId);
    const orgConfig = await this.getOrganizationConfig(key, project.organizationId);
    if (orgConfig !== null) return orgConfig;

    // 3. Fall back to system level
    return await this.getSystemConfig(key);
  }
}
```

## Sensitive Configuration Encryption

### Encryption Service
```javascript
import crypto from 'crypto';

export class SensitiveConfigEncryptionService {
  constructor(encryptionKey, iv) {
    this._key = Buffer.from(encryptionKey, 'hex');
    this._iv = Buffer.from(iv, 'hex');
  }

  encrypt(plaintext) {
    const cipher = crypto.createCipheriv('aes-256-cbc', this._key, this._iv);
    let encrypted = cipher.update(plaintext, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    return encrypted;
  }

  decrypt(ciphertext) {
    const decipher = crypto.createDecipheriv('aes-256-cbc', this._key, this._iv);
    let decrypted = decipher.update(ciphertext, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    return decrypted;
  }
}
```

### Defining Encrypted Keys
```javascript
// In definitions/definitions.js
export const ENCRYPTED_CONFIG_KEYS = {
  COMMON: [
    'CLIENT_FACING_SIGNING_PRIVATE_KEY',
    'AWS_SECRET_ACCESS_KEY',
    'STRIPE_SECRET_KEY',
    'EMAIL_SMTP_PASSWORD'
  ],
  MEDIA_PROTECTION: [
    'DRM_PRIVATE_KEY',
    'PLAYER_EXCHANGE_PRIVATE_KEY',
    'CLOUDFRONT_PRIVATE_KEY'
  ]
};
```

## Caching Strategy

### Cache Configuration Pattern
```javascript
// All database configs must use withCache
const config = await ConfigModel
  .findOne({ key })
  .withCache({
    ttl: configProvider.CONFIG_CACHE_TTL_SECONDS || 300,
    key: `config:${key}`,
    compress: true
  });
```

### Cache Invalidation
```javascript
// When updating config
await ConfigModel.findOneAndUpdate({ key }, { value });
await cacheService.delete(`config:${key}`);
await cacheService.deletePattern(`project:${projectId}:*`); // If affects projects
```

## Plan-Based Feature Configuration

### Plan Schema Pattern
```javascript
const PlanSchema = new mongoose.Schema({
  name: { type: String, required: true },
  isDefault: { type: Boolean, default: false },
  features: {
    maxStorageGB: { type: Number, required: true },
    maxBandwidthGB: { type: Number, required: true },
    enableDRM: { type: Boolean, default: false },
    enableAdvancedAnalytics: { type: Boolean, default: false },
    maxConcurrentTranscoding: { type: Number, default: 1 }
  },
  restrictions: {
    allowedVideoQualities: [String],
    maxVideoDurationMinutes: Number,
    watermarkRequired: Boolean
  }
});
```

### Feature Checking
```javascript
async checkFeatureAvailability(feature, projectId) {
  const plan = await this.getEffectivePlanForProject(projectId);
  
  if (!plan.features[feature]) {
    throw new FeatureNotAvailableError(
      `Feature '${feature}' not available in plan '${plan.name}'`
    );
  }
  
  return plan.features[feature];
}
```

## Configuration Access Patterns

### Environment Config Access
```javascript
// Direct access for system configs
const port = configProvider.API_PORT || 3000;
const isDev = configProvider.NODE_ENV === 'development';
```

### Database Config Access
```javascript
// Through service for database configs
const apiHostname = await commonConfigService.getConfigValue(
  COMMON_CONFIG_KEY_API_HOSTNAME
);

// Bulk fetch for performance
const configs = await commonConfigService.getMultipleConfigs([
  COMMON_CONFIG_KEY_AWS_REGION,
  COMMON_CONFIG_KEY_AWS_ACCESS_KEY_ID,
  COMMON_CONFIG_KEY_S3_BUCKET
]);
```

### Project Config Access
```javascript
// Get project-specific settings
const projectConfig = await mediaProtectionConfigService
  .getProjectConfiguration(projectId);

const {
  s3BucketName,
  cloudFrontDomain,
  defaultProtectionLevel
} = projectConfig;
```

## Best Practices

### DO:
- ✅ Use appropriate tier for each config type
- ✅ Cache all database configurations
- ✅ Encrypt sensitive values before storage
- ✅ Implement proper inheritance hierarchy
- ✅ Use withCache() for all config queries
- ✅ Define all keys in definitions.js
- ✅ Validate configs at service startup
- ✅ Use plan-based feature restrictions

### DON'T:
- ❌ Store system configs in database
- ❌ Store secrets in plain text
- ❌ Skip cache for database configs
- ❌ Hard-code configuration keys
- ❌ Mix environment and database configs
- ❌ Access configs directly from models
- ❌ Forget to invalidate cache on updates

## Testing Configuration

### Mock Configuration Services
```javascript
class MockCommonConfigService {
  constructor() {
    this._configs = new Map();
  }
  
  async getConfigValue(key) {
    return this._configs.get(key);
  }
  
  async setConfig(key, value) {
    this._configs.set(key, value);
  }
  
  // Add test data
  seed(configs) {
    Object.entries(configs).forEach(([key, value]) => {
      this._configs.set(key, value);
    });
  }
}

// In tests
const mockConfigService = new MockCommonConfigService();
mockConfigService.seed({
  [COMMON_CONFIG_KEY_API_HOSTNAME]: 'test.example.com',
  [COMMON_CONFIG_KEY_AWS_REGION]: 'us-east-1'
});
```

## Migration Guide

### Adding New Configuration
1. **Determine tier**: Environment, Common, or Service-specific?
2. **Add to definitions**: Define key constant
3. **Update schema**: If new config type
4. **Add encryption**: If sensitive value
5. **Update documentation**: Document purpose and format
6. **Add validation**: Ensure valid values
7. **Set defaults**: Provide sensible defaults

### Moving Between Tiers
```javascript
// From environment to database
const value = configProvider.OLD_CONFIG_KEY;
await commonConfigService.setConfig(
  COMMON_CONFIG_KEY_NEW_CONFIG,
  value
);

// Update all references
// Before: configProvider.OLD_CONFIG_KEY
// After: await commonConfigService.getConfigValue(COMMON_CONFIG_KEY_NEW_CONFIG)
```

## Remember
**Configuration is hierarchical: Environment → Common → Service → Organization → Project. Each tier serves a specific purpose.**