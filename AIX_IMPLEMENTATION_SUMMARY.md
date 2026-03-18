# AIX Support Implementation Summary

## Overview
Added support for AIX platform to the ctor-rust library. AIX uses a different constructor mechanism than other Unix-like systems, relying on specially-named functions instead of linker sections.

## Changes Made

### 1. Core Macro Implementation (`shared/src/macros/mod.rs`)

#### Added AIX-specific entry point
- Modified `__ctor_entry!` macro to detect AIX platform and route to AIX-specific implementation
- Added new `(aix, ...)` arm that generates AIX-compatible constructor code

#### AIX Constructor Pattern
On AIX, constructors must be named with the prefix `__sinit80000000_`. The implementation:
- Preserves the original user function
- Creates a wrapper function with the AIX-required naming convention
- Uses `#[export_name]` to set the correct symbol name
- Wraps the user function call in `unsafe` block as required

Example transformation:
```rust
#[ctor]
fn foo() { /* ... */ }
```

Becomes:
```rust
fn foo() { /* ... */ }

#[unsafe(no_mangle)]
#[export_name = "__sinit80000000_foo"]
extern "C" fn aix_ctor_wrapper() {
    unsafe { foo(); }
}
```

#### Platform Detection
- Added `target_os = "aix"` to the list of supported platforms in compile-time checks
- AIX is now recognized alongside Linux, macOS, Windows, BSD variants, etc.

### 2. Test Cases (`shared/tests/expand-aix/`)

Created AIX-specific test files:
- `ctor.rs`: Input test case with `#[ctor]` annotation
- `ctor.expanded.rs`: Expected macro expansion output

These follow the same pattern as existing platform-specific tests (expand-linux, expand-darwin).

### 3. Documentation Updates

#### README.md
- Added AIX to the list of supported platforms
- Added new "AIX Support" section explaining the implementation details
- Documented the `__sinit80000000_` naming convention

#### ctor/src/lib.rs
- Updated platform support documentation
- Added AIX-specific implementation notes
- Explained the difference between AIX and other platforms

## Technical Details

### AIX Constructor Mechanism
Unlike most Unix-like systems that use linker sections (`.init_array`, `.ctors`, etc.), AIX recognizes constructor functions by their name prefix. Functions starting with `__sinit80000000_` are automatically called during module initialization.

### Implementation Approach
1. **Conditional Compilation**: Uses `#[cfg(target_os = "aix")]` to apply AIX-specific code generation
2. **Symbol Export**: Uses `#[export_name]` attribute to control the exact symbol name
3. **Wrapper Pattern**: Maintains the original function and creates a separate wrapper for AIX
4. **Safety**: Properly handles unsafe requirements for constructor execution

### Compatibility
- Works with both named and anonymous constructors
- Supports all existing `#[ctor]` attribute parameters
- Maintains backward compatibility with all other platforms
- No changes required to user code when targeting AIX

## Testing
- Created test cases following existing patterns
- Tests verify correct macro expansion for AIX target
- Integration with existing test infrastructure

## Future Considerations
- AIX destructors (`#[dtor]`) may need similar treatment if AIX has special requirements
- The `__sinit80000000_` prefix is the standard for AIX; variations may exist for different priority levels
- Testing on actual AIX hardware would validate the implementation