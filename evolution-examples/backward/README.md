# Backward Compatible Schema Evolution Examples

🔄 **Backward compatibility allows new schema versions to read data written with older schemas**

This directory demonstrates schema changes that maintain backward compatibility, ensuring that consumers using the new schema can still process messages written with the old schema.

## 📁 Files in this Directory

- `base-schema.avsc` - Original customer profile schema (v1)
- `evolved-schema.avsc` - Backward compatible evolution (v2)

## ✅ Backward Compatible Changes Demonstrated

### 1. Adding Optional Fields with Defaults

```json
{
  "name": "phone",
  "type": ["null", "string"],
  "default": null,
  "doc": "Customer phone number (optional field added)"
}
```

**Why it works**: When reading old data, the new field gets the default value (`null`).

### 2. Adding Complex Fields with Defaults

```json
{
  "name": "preferences",
  "type": {
    "type": "record",
    "name": "CustomerPreferences",
    "fields": [...]
  },
  "default": {"marketing_emails": true, "language": "en"}
}
```

**Why it works**: The entire nested record has a default value, so old data can be read successfully.

### 3. Adding Enum Fields with Defaults

```json
{
  "name": "status",
  "type": {
    "type": "enum",
    "name": "CustomerStatus",
    "symbols": ["ACTIVE", "INACTIVE", "SUSPENDED"]
  },
  "default": "ACTIVE"
}
```

**Why it works**: Old records get the default enum value when read with the new schema.

### 4. Adding Array Fields with Empty Defaults

```json
{
  "name": "tags",
  "type": {
    "type": "array",
    "items": "string"
  },
  "default": []
}
```

**Why it works**: Old records get an empty array, which is a sensible default for collections.

## 🧪 Testing Backward Compatibility

### Using the MCP Server

```bash
# Register the base schema
curl -X POST http://localhost:38000/schemas \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -d '{
    "registry": "development",
    "context": "evolution-examples",
    "subject": "customer-profile-backward",
    "schema": @base-schema.avsc
  }'

# Test compatibility with evolved schema
curl -X POST http://localhost:38000/compatibility \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -d '{
    "registry": "development",
    "context": "evolution-examples", 
    "subject": "customer-profile-backward",
    "schema": @evolved-schema.avsc
  }'
```

### Expected Result
```json
{
  "is_compatible": true,
  "compatibility_level": "BACKWARD",
  "messages": []
}
```

## 📋 Sample Data Evolution

### Data Written with Base Schema (v1)
```json
{
  "id": "cust-123",
  "email": "john@example.com",
  "name": "John Doe",
  "created_at": 1609459200000
}
```

### Same Data Read with Evolved Schema (v2)
```json
{
  "id": "cust-123",
  "email": "john@example.com",
  "name": "John Doe",
  "phone": null,
  "preferences": {
    "marketing_emails": true,
    "language": "en"
  },
  "status": "ACTIVE",
  "tags": [],
  "created_at": 1609459200000,
  "updated_at": 0
}
```

## 🎯 Business Use Cases

### Gradual Feature Rollout
- Add new optional features without breaking existing consumers
- Allow old producers to continue working while new features are developed
- Enable progressive enhancement of data models

### Legacy System Integration
- Maintain compatibility with legacy systems that can't be updated immediately
- Provide default values for new requirements
- Allow for graceful migration periods

### Multi-Version Deployments
- Support blue-green deployments where old and new versions coexist
- Enable rollback scenarios without data compatibility issues
- Allow for staged rollouts across different environments

## ⚠️ Best Practices

1. **Always provide sensible defaults** for new fields
2. **Use optional (union with null) types** for non-critical fields
3. **Test compatibility** before deploying schema changes
4. **Document the business reason** for each new field
5. **Consider the performance impact** of default value processing

## 🔗 Related Examples

- [Forward Compatible Changes](../forward/README.md)
- [Full Compatibility](../full/README.md) 
- [Breaking Changes to Avoid](../breaking/README.md)
