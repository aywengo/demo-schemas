# Full Compatible Schema Evolution Examples

🔄 **Full compatibility allows both old and new schema versions to read data written by either version**

This directory demonstrates schema changes that maintain full (bidirectional) compatibility, ensuring that both old and new consumers can process data written by either old or new producers.

## 📁 Files in this Directory

- `base-schema.avsc` - Original user activity schema (v1)
- `evolved-schema.avsc` - Fully compatible evolution (v2)

## ✅ Full Compatible Changes Demonstrated

### 1. Adding New Enum Symbols (at the end)

**Base Schema**:
```json
{
  "type": "enum",
  "name": "ActivityType",
  "symbols": ["LOGIN", "LOGOUT", "PAGE_VIEW", "CLICK"]
}
```

**Evolved Schema**:
```json
{
  "type": "enum",
  "name": "ActivityType",
  "symbols": ["LOGIN", "LOGOUT", "PAGE_VIEW", "CLICK", "DOWNLOAD", "UPLOAD"]
}
```

**Why it works**: 
- **Forward**: Old consumers ignore new enum values they don't understand
- **Backward**: New consumers can handle all old enum values

### 2. Adding Optional Fields with Complete Defaults

```json
{
  "name": "device_info",
  "type": {
    "type": "record",
    "name": "DeviceInfo",
    "fields": [
      {
        "name": "device_type",
        "type": {
          "type": "enum",
          "name": "DeviceType", 
          "symbols": ["DESKTOP", "MOBILE", "TABLET"]
        },
        "default": "DESKTOP"
      },
      {
        "name": "user_agent",
        "type": ["null", "string"],
        "default": null
      }
    ]
  },
  "default": {"device_type": "DESKTOP", "user_agent": null}
}
```

**Why it works**:
- **Backward**: New consumers use defaults when reading old data
- **Forward**: Old consumers ignore the field completely

### 3. Removing Optional Fields

The `page_url` field was removed from the evolved schema.

**Why it works**:
- **Forward**: Old consumers use their default when the field is missing
- **Backward**: New consumers ignore extra fields in old data

### 4. Field Reordering

Fields have been reordered in the evolved schema while maintaining compatibility.

## 🧪 Testing Full Compatibility

### Using the MCP Server

```bash
# Register base schema
curl -X POST http://localhost:38000/schemas \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -d '{
    "registry": "development",
    "context": "evolution-examples",
    "subject": "user-activity-full",
    "schema": @base-schema.avsc
  }'

# Test full compatibility with evolved schema
curl -X POST http://localhost:38000/compatibility \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -d '{
    "registry": "development",
    "context": "evolution-examples",
    "subject": "user-activity-full",
    "schema": @evolved-schema.avsc
  }'
```

### Expected Result
```json
{
  "is_compatible": true,
  "compatibility_level": "FULL",
  "messages": []
}
```

## 📋 Sample Data Evolution Scenarios

### Scenario 1: Old Producer → New Consumer

**Data written with base schema (v1)**:
```json
{
  "user_id": "user-123",
  "session_id": "sess-456",
  "activity_type": "PAGE_VIEW",
  "page_url": "/products/laptop",
  "duration_ms": 5000,
  "timestamp": 1609459200000
}
```

**Data read by evolved schema (v2)**:
```json
{
  "user_id": "user-123",
  "session_id": "sess-456",
  "activity_type": "PAGE_VIEW",
  "duration_ms": 5000,
  "device_info": {
    "device_type": "DESKTOP",
    "user_agent": null
  },
  "referrer": null,
  "timestamp": 1609459200000
}
```

### Scenario 2: New Producer → Old Consumer

**Data written with evolved schema (v2)**:
```json
{
  "user_id": "user-789",
  "session_id": "sess-012",
  "activity_type": "DOWNLOAD",
  "duration_ms": 2000,
  "device_info": {
    "device_type": "MOBILE",
    "user_agent": "Mozilla/5.0..."
  },
  "referrer": "https://google.com",
  "timestamp": 1609459260000
}
```

**Data read by base schema (v1)**:
```json
{
  "user_id": "user-789",
  "session_id": "sess-012",
  "activity_type": "DOWNLOAD",
  "page_url": null,
  "duration_ms": 2000,
  "timestamp": 1609459260000
}
```

**Note**: The old consumer sees "DOWNLOAD" as an unknown enum value and may handle it according to its error handling strategy.

## 🎯 Business Use Cases

### Gradual Migration Scenarios
- **Zero-downtime deployments** with mixed old/new versions
- **Canary releases** where some instances use new schemas
- **Multi-region deployments** with different rollout schedules

### Multi-Team Development
- **Different teams** can upgrade at their own pace
- **Shared data contracts** that don't break existing integrations
- **API versioning** where multiple versions coexist

### Data Pipeline Evolution
- **Stream processing** systems with mixed consumer versions
- **Data lake ingestion** supporting multiple schema versions
- **Real-time analytics** with evolving event definitions

## 🏗️ Design Patterns for Full Compatibility

### 1. Additive-Only Changes
```json
{
  "name": "new_optional_field",
  "type": ["null", "string"],
  "default": null
}
```

### 2. Enum Extension Pattern
```json
{
  "type": "enum",
  "symbols": ["EXISTING_1", "EXISTING_2", "NEW_VALUE"]
}
```

### 3. Graceful Field Removal
```bash
# Phase 1: Mark as deprecated, make optional
# Phase 2: Stop producing the field
# Phase 3: Remove from schema
```

### 4. Complex Default Values
```json
{
  "name": "config",
  "type": {
    "type": "record",
    "name": "Config",
    "fields": [...]
  },
  "default": {
    "setting1": "default_value",
    "setting2": false
  }
}
```

## ⚠️ Challenges and Limitations

### 1. Enum Handling
Old consumers may not handle new enum values gracefully:
```java
// Java example - may throw exception
switch (activityType) {
    case LOGIN: // handle
    case LOGOUT: // handle  
    default: // Must handle unknown values!
}
```

### 2. Performance Considerations
- Default value computation overhead
- Increased payload size with optional fields
- Schema resolution complexity

### 3. Testing Complexity
Must test all combinations:
- Old producer → Old consumer ✓
- Old producer → New consumer ✓  
- New producer → Old consumer ✓
- New producer → New consumer ✓

## 🔧 Implementation Best Practices

### 1. Comprehensive Testing Matrix
```bash
# Test all producer/consumer combinations
for producer_version in [v1, v2]; do
  for consumer_version in [v1, v2]; do
    test_compatibility $producer_version $consumer_version
  done
done
```

### 2. Monitoring and Alerting
```yaml
alerts:
  - name: unknown_enum_values
    condition: count(unknown_enum_errors) > threshold
  - name: default_value_usage
    condition: rate(default_values_used) > baseline
```

### 3. Documentation Standards
```json
{
  "doc": "Field added in v2.1. Safe default: null. Consumers should handle missing values gracefully."
}
```

## 🚨 Avoid These Pitfalls

❌ **Changing field types** - Breaks both directions  
❌ **Removing required fields** - Breaks backward compatibility  
❌ **Reordering enum symbols** - Can break enum handling  
❌ **Changing logical types** - May cause data corruption  
❌ **Modifying default values** - Can cause semantic changes

## 🔗 Related Examples

- [Backward Compatible Changes](../backward/README.md)
- [Forward Compatible Changes](../forward/README.md)
- [Breaking Changes to Avoid](../breaking/README.md)
