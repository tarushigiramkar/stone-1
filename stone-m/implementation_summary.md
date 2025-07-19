# Route Attributes Schema Validation Implementation

This document summarizes the changes made to implement schema validation for route attributes in Stone.

## Changes Overview

1. **Fixed Parser Bug**
   - Removed the problematic line that was setting `self.exhausted = True` prematurely in the parse method

2. **Added Route Schema Validator**
   - Created a new class `RouteAttributeValidator` to handle validation of route attributes
   - Added support for validating primitive types, nullable types, and union tags
   - Implemented default value handling and optional attribute support

3. **Enhanced IR Generation**
   - Updated `_validate_stone_cfg` to properly extract the Route schema
   - Modified `_populate_route_attributes_helper` to use the new validator
   - Added type-checking for all attributes against their schema definitions

4. **Improved Error Handling**
   - Added clear error messages for type mismatches
   - Added validation for duplicate attributes
   - Added checks for missing required attributes

5. **Extended API Model**
   - Added a new `get_route_schema()` method to the Api class to expose the schema to consumers

## New Features

### 1. Type-Safe Route Attributes

Route attributes are now validated against a schema defined in the `stone_cfg` namespace:

```stone
namespace stone_cfg

struct Route
    auth Boolean = true
    cache_ttl Int64?
```

### 2. Default Values for Optional Attributes

Attributes with default values are automatically populated when not specified:

```stone
namespace stone_cfg

struct Route
    log_level String = "INFO"  # Will default to "INFO" if not specified
```

### 3. Nullable Attribute Support

Attributes can be marked as optional using the nullable modifier:

```stone
namespace stone_cfg

struct Route
    rate_limit Int64?  # Can be omitted in route definitions
```

### 4. Union Tag Validation

Union tags can be used as attribute values, with validation to ensure they are void tags:

```stone
namespace app

union Environment
    production
    staging
    development

namespace stone_cfg
import app

struct Route
    environment app.Environment = production
```

## Implementation Details

### RouteAttributeValidator

The core of the implementation is the `RouteAttributeValidator` class which:

1. Validates that all required attributes are provided
2. Checks that attribute values match their defined types
3. Handles nullable attributes and default values
4. Verifies that union tags refer to void fields
5. Ensures no unknown attributes are specified

### Schema Loading

The route schema is loaded from the `stone_cfg` namespace during IR generation:

1. The system looks for a struct named `Route` in the `stone_cfg` namespace
2. If found, it's stored in the API object via `api.add_route_schema()`
3. If not found, a default empty schema is created

### Type Validation

Each attribute is validated according to its type:

- **Primitives** are checked using their `check_attr_repr()` methods
- **Nullable types** allow for `None` values
- **Aliases** are properly unwrapped and checked
- **Union tags** are verified to exist and be void fields

## Future Improvements

1. **Schema Versioning**: Add support for versioned schemas to handle API evolution
2. **Schema Inheritance**: Allow schemas to extend from base schemas for better reuse
3. **Complex Validation Rules**: Support more advanced validation beyond simple type checking
