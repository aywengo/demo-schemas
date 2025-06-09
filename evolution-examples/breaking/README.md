# Breaking Schema Changes Examples

🚨 **Breaking changes prevent older schemas from reading data written with newer schemas**

This directory demonstrates schema changes that break compatibility, showing what changes to avoid in production systems and how to identify potential breaking changes before they cause issues.

## 📁 Files in this Directory

- `base-schema.avsc` - Original product inventory schema
- `type-change-example.avsc` - Breaking type changes
- `field-removal-example.avsc` - Breaking field removals
- `enum-change-example.avsc` - Breaking enum modifications
- `field-rename-example.avsc` - Breaking field renames

## 🚨 Breaking Changes Demonstrated

### 1. Type Changes

**Breaking Changes in `type-change-example.avsc`**:

```json
// BEFORE (int)
{
  "name": "quantity",
  "type": "int"
}

// AFTER (long) - BREAKING!
{
  "name": "quantity", 
  "type": "long"
}
```

**Why it breaks**: 
- Old consumers expect `int` (32-bit) but get `long` (64-bit)
- Data written as `long` cannot be read by `int` field definitions
- Logical type changes (`timestamp-millis` to `string`) are also breaking

### 2. Removing Required Fields

**Breaking Changes in `field-removal-example.avsc`**:

```json
// REMOVED FIELDS:
// - warehouse_id (required field)
// - notes (optional field without default handling)
```

**Why it breaks**:
- Old consumers expect `warehouse_id` field to be present
- Data written without these fields cannot populate required fields
- Forward compatibility is broken

### 3. Enum Symbol Changes

**Breaking Changes in `enum-change-example.avsc`**:

```json
// BEFORE
"symbols": ["IN_STOCK", "LOW_STOCK", "OUT_OF_STOCK", "DISCONTINUED"]

// AFTER - BREAKING!
"symbols": ["AVAILABLE", "LIMITED", "UNAVAILABLE", "RETIRED"]
```

**Why it breaks**:
- Existing data contains old enum values
- New schema cannot read old enum values
- Completely different enum symbols break all compatibility

### 4. Field Renames Without Aliases

**Breaking Changes in `field-rename-example.avsc`**:

```json
// BEFORE
{"name": "product_id", "type": "string"}

// AFTER - BREAKING!
{"name": "sku", "type": "string"}
```

**Why it breaks**:
- Field matching is done by name in Avro
- No alias provided to map old name to new name
- Old data has `product_id`, new schema expects `sku`

## 🧪 Testing Breaking Changes

### Using the MCP Server

```bash
# Register base schema
curl -X POST http://localhost:38000/schemas \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -d '{
    "registry": "development",
    "context": "evolution-examples",
    "subject": "inventory-breaking-test",
    "schema": @base-schema.avsc
  }'

# Test compatibility with breaking change
curl -X POST http://localhost:38000/compatibility \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -d '{
    "registry": "development",
    "context": "evolution-examples",
    "subject": "inventory-breaking-test",
    "schema": @type-change-example.avsc
  }'
```

### Expected Result
```json
{
  "is_compatible": false,
  "compatibility_level": "NONE",
  "messages": [
    "Field quantity changed type from int to long",
    "Field last_updated changed logical type"
  ]
}
```

## 📋 Impact Examples

### Type Change Impact

**Old data (base schema)**:
```json
{
  "product_id": "SKU-123",
  "warehouse_id": "WH-001",
  "quantity": 100,
  "status": "IN_STOCK",
  "last_updated": 1609459200000,
  "notes": null
}
```

**Attempting to read with type-change schema**:
```
❌ ERROR: Cannot read int value 100 as long
❌ ERROR: Cannot read timestamp-millis as string
```

### Field Removal Impact

**New data (field-removal schema)**:
```json
{
  "product_id": "SKU-456",
  "quantity": 50,
  "status": "LOW_STOCK",
  "last_updated": 1609459260000
}
```

**Attempting to read with base schema**:
```
❌ ERROR: Required field 'warehouse_id' not found
❌ ERROR: Field 'notes' missing and no default provided
```

## 🔧 How to Avoid Breaking Changes

### 1. Safe Type Evolution

✅ **DO**: Use union types for flexibility
```json
{
  "name": "quantity",
  "type": ["int", "long"],
  "default": 0
}
```

❌ **DON'T**: Change types directly
```json
{
  "name": "quantity",
  "type": "long"  // Breaking if was int before
}
```

### 2. Safe Field Changes

✅ **DO**: Use aliases for renames
```json
{
  "name": "sku",
  "aliases": ["product_id"],
  "type": "string"
}
```

❌ **DON'T**: Rename without aliases
```json
{
  "name": "sku",  // Breaking - no alias for product_id
  "type": "string"
}
```

### 3. Safe Enum Evolution

✅ **DO**: Add new symbols at the end
```json
{
  "type": "enum",
  "symbols": ["IN_STOCK", "LOW_STOCK", "OUT_OF_STOCK", "DISCONTINUED", "BACKORDERED"]
}
```

❌ **DON'T**: Change existing symbols
```json
{
  "type": "enum",
  "symbols": ["AVAILABLE", "LIMITED", "UNAVAILABLE", "RETIRED"]
}
```

### 4. Safe Field Removal

✅ **DO**: Gradual deprecation process
```bash
# Phase 1: Mark as deprecated
# Phase 2: Make optional with default
# Phase 3: Stop writing to field  
# Phase 4: Remove after all consumers updated
```

❌ **DON'T**: Remove required fields immediately

## 📋 Migration Strategies

### 1. Dual Schema Approach

```bash
# Support both old and new schemas simultaneously
register_schema "inventory-v1" base-schema.avsc
register_schema "inventory-v2" evolved-schema.avsc

# Gradually migrate consumers
# Deprecate v1 after full migration
```

### 2. Data Transformation Layer

```python
# Example transformation layer
def transform_legacy_data(old_data):
    new_data = old_data.copy()
    
    # Handle type changes
    if 'quantity' in new_data:
        new_data['quantity'] = long(new_data['quantity'])
    
    # Handle field renames
    if 'product_id' in new_data:
        new_data['sku'] = new_data.pop('product_id')
    
    return new_data
```

### 3. Schema Registry Compatibility Policies

```yaml
# Enforce compatibility at registry level
compatibility_policy: BACKWARD  # Prevent breaking changes

# Or use FORWARD/FULL for stricter compatibility
# NONE only for emergency situations
```

## 🚨 Emergency Breaking Change Process

Sometimes breaking changes are unavoidable (security, legal, performance):

### 1. Plan the Migration
```bash
# 1. Assess impact
find_all_consumers_of_schema inventory-v1

# 2. Create migration timeline
# 3. Prepare rollback plan
# 4. Set up monitoring
```

### 2. Communication Strategy
```markdown
# Breaking Change Notice

**Schema**: ProductInventory
**Effective Date**: 2024-03-01
**Breaking Changes**:
- Field `quantity` changed from int to long
- Field `warehouse_id` removed

**Migration Required**: Yes
**Deadline**: 2024-02-28
**Support Contact**: data-platform@company.com
```

### 3. Gradual Rollout
```bash
# Stage 1: Deploy to dev environment
# Stage 2: Deploy to staging with monitoring
# Stage 3: Canary deployment (5% of traffic)
# Stage 4: Full deployment with rollback ready
```

## 📊 Monitoring Breaking Changes

### 1. Schema Registry Metrics
```yaml
metrics:
  - compatibility_check_failures
  - schema_registration_errors  
  - consumer_schema_version_lag
```

### 2. Application-Level Monitoring
```yaml
alerts:
  - name: deserialization_errors
    condition: rate(avro_deserialization_errors) > 0
  - name: unknown_field_warnings
    condition: count(unknown_field_logs) > threshold
```

### 3. Consumer Health Checks
```python
# Health check endpoint
def check_schema_compatibility():
    try:
        latest_schema = registry.get_latest_schema('inventory')
        test_deserialize(sample_data, latest_schema)
        return {"status": "healthy"}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}
```

## 🔗 Tools and Automation

### 1. Pre-commit Schema Validation
```yaml
# .github/workflows/schema-check.yml
name: Schema Compatibility Check
on: [pull_request]
jobs:
  check-compatibility:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Check Schema Compatibility
        run: |
          for schema in schemas/*.avsc; do
            check_compatibility $schema
          done
```

### 2. Automated Migration Testing
```bash
# Test data migration with all schema versions
for old_version in $(get_schema_versions); do
  for new_version in $(get_schema_versions); do
    test_migration $old_version $new_version
  done
done
```

### 3. Breaking Change Detection
```python
# Automated breaking change detection
def detect_breaking_changes(old_schema, new_schema):
    breaking_changes = []
    
    # Check for type changes
    for field in old_schema.fields:
        new_field = new_schema.get_field(field.name)
        if new_field and field.type != new_field.type:
            breaking_changes.append(f"Type change: {field.name}")
    
    # Check for removed fields
    old_fields = {f.name for f in old_schema.fields}
    new_fields = {f.name for f in new_schema.fields}
    removed_fields = old_fields - new_fields
    
    for field_name in removed_fields:
        breaking_changes.append(f"Removed field: {field_name}")
    
    return breaking_changes
```

## 🔗 Related Examples

- [Backward Compatible Changes](../backward/README.md)
- [Forward Compatible Changes](../forward/README.md)
- [Full Compatibility](../full/README.md)

## 📚 Additional Resources

- [Apache Avro Schema Evolution](https://avro.apache.org/docs/current/spec.html#Schema+Resolution)
- [Confluent Schema Evolution Best Practices](https://docs.confluent.io/platform/current/schema-registry/avro.html#schema-evolution)
- [Schema Registry Compatibility Types](https://docs.confluent.io/platform/current/schema-registry/avro.html#compatibility-types)
