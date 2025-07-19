# Route Attributes Schema Validation

This document explains how to use the new schema validation feature for route attributes in the Stone specification language.

## Overview

Route attributes provide a way to associate metadata with API routes. The new schema validation system enforces type safety and consistency for these attributes across your API.

## Defining a Route Schema

A special namespace called `stone_cfg` is reserved for configuration. Within this namespace, you can define a struct called `Route` that specifies the schema for route attributes:

```stone
namespace stone_cfg

struct Route
    auth Boolean = true       # Required authentication (defaults to true)
    cache_ttl Int64?          # Optional cache time-to-live in seconds
    rate_limit Int64 = 100    # Rate limit (defaults to 100)
    scope String              # Required scope
```

## Schema Rules

The `Route` struct follows these rules:

1. **Field Types**: Any valid Stone data type can be used, including primitives, aliases, and references to types in other namespaces.

2. **Optional Fields**: Fields can be made optional by using the nullable modifier (`?`).

3. **Default Values**: Fields can have default values specified with `=`.

4. **Documentation**: Fields can have doc strings just like regular struct fields.

## Using Route Attributes

Once you've defined a schema, you can specify attributes for each route:

```stone
namespace files

route list_folder(Void, Void, Void)
    "Lists the contents of a folder"
    attrs
        auth = true
        cache_ttl = 300
        rate_limit = 50
        scope = "files.read"
```

## Type Validation

The validation system ensures that attribute values match their declared types:

- **Primitives**: Values must match their type (e.g., strings for String, numbers for Int64)
- **Nullable Types**: Optional fields can be omitted
- **Default Values**: Fields with defaults will use their default value if not specified
- **Union Tags**: Only references to void union fields are allowed

## Examples

### Basic Attributes

```stone
namespace stone_cfg

struct Route
    log_level String = "INFO"
    target_users String?
```

```stone
namespace users

route get_info(Void, Void, Void)
    attrs
        log_level = "DEBUG"
        target_users = "premium"
```

### Referencing Types from Other Namespaces

```stone
namespace app

union Environment
    production
    staging
    development
```

```stone
namespace stone_cfg

import app

struct Route
    environment app.Environment = production
```

### Type-Specific Requirements

```stone
namespace stone_cfg

struct Route
    max_results Int64 = 100     # Must be a number
    scopes String                # Must be a string
    include_deleted Boolean?     # Optional boolean
```

## Best Practices

1. **Default Values**: Provide reasonable defaults to make route definitions cleaner.

2. **Documentation**: Add doc strings to schema fields to help API developers understand their purpose.

3. **Namespacing**: For complex attributes, define reusable types in a separate namespace and reference them in the schema.

4. **Versioning**: When making incompatible changes to attributes, use versioned routes.
