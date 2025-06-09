# Forward Compatible Schema Evolution Examples

🔄 **Forward compatibility allows older schema versions to read data written with newer schemas**

This directory demonstrates schema changes that maintain forward compatibility, ensuring that consumers using the old schema can still process messages written with the new schema.

## 📁 Files in this Directory

- `base-schema.avsc` - Original order event schema (v1)
- `evolved-schema.avsc` - Forward compatible evolution (v2)

## ✅ Forward Compatible Changes Demonstrated

### 1. Removing Optional Fields

The evolved schema removes the optional `shipping_address` field:

**Base Schema (v1)**:
```json
{
  "name": "shipping_address",
  "type": ["null", "string"],
  "default": null,
  "doc": "Shipping address (optional)"
}
```

**Evolved Schema (v2)**: Field is completely removed

**Why it works**: When old consumers read new data, they simply ignore the missing field since it was optional.

### 2. Field Reordering

Notice that in the evolved schema, the field order has changed:
- `shipping_address` removed
- `order_date` moved up
- `metadata` remains at the end

**Why it works**: Avro uses field names for matching, not positions, so reordering doesn't break compatibility.

## 🧪 Testing Forward Compatibility

### Using the MCP Server

```bash
# Register the evolved schema first
curl -X POST http://localhost:38000/schemas \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -d '{
    "registry": "development",
    "context": "evolution-examples",
    "subject": "order-event-forward",
    "schema": @evolved-schema.avsc
  }'

# Test if base schema can read data written with evolved schema
curl -X POST http://localhost:38000/compatibility \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -d '{
    "registry": "development",
    "context": "evolution-examples",
    "subject": "order-event-forward",
    "schema": @base-schema.avsc
  }'
```

### Expected Result
```json
{
  "is_compatible": true,
  "compatibility_level": "FORWARD",
  "messages": []
}
```

## 📋 Sample Data Evolution

### Data Written with Evolved Schema (v2)
```json
{
  "order_id": "ord-456",
  "customer_id": "cust-789",
  "total_amount": "\u0000\u00C7\u0010",
  "currency": "USD",
  "status": "CONFIRMED",
  "order_date": 1609459200000,
  "metadata": {
    "source": "mobile_app",
    "promotion_code": "SAVE10"
  }
}
```

### Same Data Read with Base Schema (v1)
```json
{
  "order_id": "ord-456",
  "customer_id": "cust-789",
  "total_amount": "\u0000\u00C7\u0010",
  "currency": "USD",
  "status": "CONFIRMED",
  "shipping_address": null,
  "order_date": 1609459200000,
  "metadata": {
    "source": "mobile_app",
    "promotion_code": "SAVE10"
  }
}
```

**Note**: The old schema populates `shipping_address` with its default value (`null`) even though this field doesn't exist in the new data.

## 🎯 Business Use Cases

### Data Model Simplification
- Remove deprecated or unused fields from the schema
- Clean up technical debt without breaking existing consumers
- Simplify data models based on usage analytics

### Privacy and Compliance
- Remove PII fields that are no longer needed
- Comply with data retention policies
- Implement "right to be forgotten" requirements

### Performance Optimization
- Remove large optional fields that impact performance
- Reduce payload size for high-volume streams
- Optimize for specific consumer patterns

### Schema Cleanup
- Remove experimental fields that didn't work out
- Consolidate similar fields
- Align schemas with current business requirements

## ⚠️ Limitations and Considerations

### 1. Information Loss
Removing fields means that information is permanently lost for consumers using the old schema.

### 2. Default Value Behavior
Old consumers will see default values for removed fields, which might not represent the actual business state.

### 3. Monitoring Requirements
Ensure monitoring is in place to detect when old consumers are still expecting removed fields.

## 🔧 Implementation Best Practices

### 1. Gradual Removal Process
```bash
# Phase 1: Mark field as deprecated (documentation)
# Phase 2: Make field optional if not already
# Phase 3: Stop writing to the field
# Phase 4: Remove from schema after all consumers updated
```

### 2. Feature Flags for Gradual Migration
```json
{
  "name": "legacy_field",
  "type": ["null", "string"],
  "default": null,
  "doc": "DEPRECATED: Will be removed in v3.0. Use new_field instead."
}
```

### 3. Consumer Version Tracking
Track which schema versions your consumers are using before removing fields.

## 🚨 What NOT to Do

❌ **Don't remove required fields** - This breaks forward compatibility
❌ **Don't remove fields with complex defaults** - May cause unexpected behavior  
❌ **Don't remove enum symbols** - Existing data becomes unreadable
❌ **Don't change field types** - Not a forward-compatible change

## 🔗 Related Examples

- [Backward Compatible Changes](../backward/README.md)
- [Full Compatibility](../full/README.md)
- [Breaking Changes to Avoid](../breaking/README.md)
