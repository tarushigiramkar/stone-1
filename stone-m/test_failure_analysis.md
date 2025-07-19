# Test Failure Analysis: `test_route_attrs_schema`

## Summary
The test `test_route_attrs_schema` is failing due to a parser state issue. The error message indicates: `AssertionError: Must call get_parser() to reset state.`

## Root Cause
There is a bug in the `parse()` method in `stone/frontend/parser.py` at line 88. The code currently reads:

```python
def parse(self, data, path=None):
    """
    Args:
        data (str): Raw specification text.
        path (Optional[str]): Path to specification on filesystem. Only
            used to tag tokens with the file they originated from.
    """
    self.exhausted = True  # ← BUG: This line should not be here!
    assert not self.exhausted, 'Must call get_parser() to reset state.'
    self.path = path
    parsed_data = self.yacc.parse(data, lexer=self.lexer, debug=self.debug)
    # ... rest of method
    self.exhausted = True  # ← This is the correct place to mark as exhausted
    return parsed_data
```

The line `self.exhausted = True` at line 88 is setting the parser as exhausted immediately before checking that it's not exhausted, which will always fail. This line should be removed.

## Impact of the Bug

I've confirmed that this bug affects ALL tests that use `specs_to_ir()`. When I ran another test (`test_line_continuations`), it failed with the exact same error. The bug in line 88 of parser.py prevents ANY use of the parser's `parse()` method, which means:

1. **All `specs_to_ir()` calls fail** - This function internally calls `parser.parse()`
2. **Direct parser usage also fails** - Any code calling `parser.parse()` will hit this assertion
3. **The entire parsing functionality is broken** - This is a critical bug that breaks the core functionality

The bug was likely introduced when implementing the route attributes feature, possibly as an accidental edit or merge conflict.

## Test Structure Analysis

The `test_route_attrs_schema` test is designed to validate the new route attribute schema functionality. It tests several scenarios:

1. **Preventing routes in stone_cfg namespace** - Tests that routes cannot be defined in the special `stone_cfg` namespace
2. **Type validation** - Tests that attribute values match their declared types 
3. **Required attributes** - Tests that routes must define all required attributes
4. **Default values** - Tests that attributes with defaults are populated correctly
5. **Optional/nullable attributes** - Tests that nullable attributes can be omitted
6. **Unknown attributes** - Tests that undefined attributes are rejected
7. **Complex types** - Tests various primitive and composite types in attributes
8. **Union types** - Tests that union-typed attributes work correctly
9. **Duplicate attributes** - Tests that attributes cannot be defined twice

## Code Flow Analysis

The route attribute schema validation works as follows:

1. **Parser Phase**: 
   - Routes can have an `attrs` section after their definition
   - Attributes are parsed as key-value pairs

2. **IR Generation Phase**:
   - `_validate_stone_cfg()` method checks for a special `stone_cfg` namespace
   - If found, it extracts a `Route` struct that defines the schema
   - The schema is stored in `api.route_schema`

3. **Validation Phase**:
   - `_populate_route_attributes_helper()` validates route attributes against the schema
   - Uses `schema.check_attr_repr()` to validate each attribute
   - The Struct's `check_attr_repr()` method validates types and handles defaults/nullable fields

## Key Components

### Parser (stone/frontend/parser.py)
- Added `ATTRS` token to lexer
- Added grammar rules for parsing attrs section
- Stores attrs in `AstRouteDef.attrs`

### IR Generator (stone/frontend/ir_generator.py)
- `_validate_stone_cfg()` - Extracts route schema from stone_cfg namespace
- `_populate_route_attributes_helper()` - Validates route attrs against schema

### Data Types (stone/ir/data_types.py)
- `Struct.check_attr_repr()` - Validates attributes match struct fields
- Each data type has its own `check_attr_repr()` method for type validation

### API Model (stone/ir/api.py)
- `Api.route_schema` - Stores the route attribute schema
- `ApiRoute.attrs` - Stores validated attributes for each route

## Parser State Issue

The immediate failure is due to parser state management. The `parse()` method in `Parser` class checks:
```python
assert not self.exhausted, 'Must call get_parser() to reset state.'
```

This suggests the parser is being reused without proper reset between test invocations.

## Fix Required

The fix is simple and straightforward:

**In `stone/frontend/parser.py`, line 88 needs to be removed:**

```python
# Current (broken) code:
def parse(self, data, path=None):
    """
    Args:
        data (str): Raw specification text.
        path (Optional[str]): Path to specification on filesystem. Only
            used to tag tokens with the file they originated from.
    """
    self.exhausted = True  # ← DELETE THIS LINE
    assert not self.exhausted, 'Must call get_parser() to reset state.'
    # ... rest of method
```

**After the fix:**
```python
def parse(self, data, path=None):
    """
    Args:
        data (str): Raw specification text.
        path (Optional[str]): Path to specification on filesystem. Only
            used to tag tokens with the file they originated from.
    """
    assert not self.exhausted, 'Must call get_parser() to reset state.'
    # ... rest of method
```

## Conclusion

The failing test `test_route_attrs_schema` is not actually testing faulty logic. The test itself is well-designed and comprehensive. The failure is due to a critical bug in the parser that prevents any parsing from occurring. Once line 88 is removed from `parser.py`, all tests should pass and the route attributes schema functionality can be properly validated.

The route attributes implementation appears to be complete and follows the established patterns in the codebase:
- Schema definition via `stone_cfg.Route` struct
- Type validation for all primitive types
- Support for defaults and nullable fields
- Proper error handling for missing/unknown attributes
- Integration with the existing IR generation pipeline
