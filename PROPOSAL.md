# Parcelator — Implementation Proposal

The _Parcelator_ is a source-level translator that processes C source files containing
[_Parcel_](https://github.com/nbyoung/parcel) language declarations and generates the
`export/` and `import/` include files that realise the Parcel module semantics in
standard C. It operates as a _pre-preprocessor_: it runs before the C preprocessor
and writes generated files into a `_parcel/` subdirectory, leaving the compiler
toolchain otherwise unchanged.

This document proposes the design and implementation of the Parcelator. For the
language semantics it must implement see
[SEMANTICS.md](https://github.com/nbyoung/parcel/blob/main/SEMANTICS.md); for the
diagnostics it must emit see
[TRANSLATION.md](https://github.com/nbyoung/parcel/blob/main/TRANSLATION.md).

---

## Contents

1. [Pipeline overview](#1-pipeline-overview)
2. [Stage 1 — Discovery](#2-stage-1--discovery)
3. [Stage 2 — Declaration parsing](#3-stage-2--declaration-parsing)
4. [Stage 3 — Type graph construction](#4-stage-3--type-graph-construction)
5. [Stage 4 — Type expansion](#5-stage-4--type-expansion)
6. [Stage 5 — Validation](#6-stage-5--validation)
7. [Stage 6 — Code generation](#7-stage-6--code-generation)
8. [Build integration](#8-build-integration)
9. [Implementation language](#9-implementation-language)
10. [Key data structures](#10-key-data-structures)
11. [Testing strategy](#11-testing-strategy)
12. [Implementation phases](#12-implementation-phases)

---

## 1. Pipeline overview

```
 C source files (.c)
        │
        ▼
┌───────────────┐
│  1. Discovery │  Locate #pragma parcel, export includes, import includes
└───────┬───────┘
        │  parcel declarations, export paths, import directives
        ▼
┌───────────────────────┐
│  2. Declaration       │  For each exported identifier, extract its C
│     parsing           │  declaration and classify its kind and type
└───────┬───────────────┘
        │  typed identifier records
        ▼
┌───────────────────────┐
│  3. Type graph        │  Build a directed graph of typedef-name
│     construction      │  dependencies within and across parcels
└───────┬───────────────┘
        │  typedef dependency graph
        ▼
┌───────────────────────┐
│  4. Type expansion    │  Resolve all typedef names to primitive forms
│                       │  (primitive types, struct/union/enum tags)
└───────┬───────────────┘
        │  fully-expanded type specifiers
        ▼
┌───────────────────────┐
│  5. Validation        │  Emit errors and warnings per TRANSLATION.md
└───────┬───────────────┘
        │  validated, expanded parcel model
        ▼
┌───────────────────────┐
│  6. Code generation   │  Write _parcel/export/... and _parcel/import/...
└───────────────────────┘
```

All six stages process the entire source tree before generating any output, so that
cross-file information (import stems, canonical name collisions) is available during
validation.

---

## 2. Stage 1 — Discovery

### Inputs

All `.c` files reachable from the project root (or from an explicitly specified set
of source roots).

### What to find

The scanner performs a single linear pass over each source file looking for three
directive patterns, ignoring all other content:

**Parcel declaration**

```
#pragma parcel <name> { <id> ... }
```

Records: source file path, parcel name, ordered list of exported identifier names.

**Export include**

```
#include "export/<path>"
```

Associates all preceding `#pragma parcel <name>` statements in the same file whose `<name>` matches the
trailing segment of `<path>` with that export path. The leading segments of `<path>` are the namespace
path. Records: canonical name (`<path_underscored>_<name>`).

**Import include**

```
#include "import/<path>/<name>.<stem>"
```

Records: importing source file, namespace path, parcel name, stem.

### Key constraint

The Parcel language requires the export include to appear _after_ all definitions the
parcel exports. The scanner does not enforce ordering — that is a Stage 5 concern —
but it does record the line number of each directive to support ordering validation.

### Output

A project-level catalogue:

- For each source file: zero or more parcel declaration records, each with its export
  path and the line numbers of both the pragma and the export include. Multiple
  declarations with the same parcel name are merged into a single record whose
  identifier list is the cumulative union and whose export position is that of the
  last pragma (see TRANSLATION.md _Cumulative parcel identifiers_).
- A list of all import directives project-wide, keyed by `(namespace path, parcel
  name)` and stem.

---

## 3. Stage 2 — Declaration parsing

### Goal

For each identifier listed in a `#pragma parcel` declaration, find its C declaration
in the same file, determine its _kind_ (typedef, variable, constant, function), and
record its type information.

### Kind classification

| Kind | Syntactic form |
|------|---------------|
| `typedef` | `typedef <T> <Id> ;` |
| `variable` | `<T> <id> [ = <init> ] ;` |
| `constant` | `const <T> <id> [ = <init> ] ;` or `<T> const <id> …` |
| `function` | `<R> <f> ( <params> ) { … }` or `<R> <f> ( <params> ) ;` |

### Parsing strategy

A full C parser is not required. The parcelator uses a _targeted lexer_ approach:

1. **Tokenise** the source file into a flat token stream: identifiers, keywords,
   punctuation, string literals, numeric literals, and comments (discarded). Block
   contents (brace-delimited bodies) are consumed as opaque tokens — their internal
   structure is not parsed.

2. **Scan forward** from the top of the token stream for any token matching one of
   the exported identifier names. On a match, attempt to parse the surrounding token
   sequence as a declaration using the grammar rules below.

3. **Record** the result as a typed identifier record. If no declaration is found
   for a listed identifier, record an _undefined identifier_ error (Stage 5).

### Declaration grammar (simplified)

The parser recognises the following top-level declaration forms. All other
constructs are skipped.

**Simple typedef**
```
typedef  <type-specifier>  <identifier>  ;
```
Type specifier: everything between `typedef` and the final `<identifier> ;`.
Declarator identifier: last identifier before `;`.

**Pointer typedef**
```
typedef  <type-specifier>  * [ const ]  <identifier>  ;
```

**Function-pointer typedef**
```
typedef  <type-specifier>  ( * <identifier> )  ( <param-list> )  ;
```
The identifier is embedded between `(*` and `)`. The return type is the
specifier preceding `(*`, expanded recursively if it contains typedef names.

**Struct typedef**
```
typedef  struct  { <member-list> }  <identifier>  ;
```
Member types are recorded but not fully expanded at this stage.

**Variable declaration**
```
<type-specifier>  <identifier>  [ = <initialiser> ]  ;
```
`const` absent from the type specifier (after stripping leading `static`).

**Constant declaration**
```
const  <type-specifier>  <identifier>  [ = <initialiser> ]  ;
```
or `<type-specifier>  const  <identifier>  …`

**Function definition or prototype**
```
[ static ]  <return-type-specifier>  <identifier>  ( <param-list> )  ( ; | { … } )
```

### Declarator reconstruction

For import file generation, the parcelator must be able to reconstruct the declarator
with the exported identifier name replaced by a stem-prefixed form. The original
declarator is stored verbatim alongside its identifier position, so that substitution
can be applied without re-parsing.

---

## 4. Stage 3 — Type graph construction

### Purpose

Type specifier expansion (Stage 4) requires knowing, for each typedef name used in a
type specifier, what that name expands to. This stage builds the dependency graph that
makes that computation possible.

### Graph structure

The type graph is a directed graph where each node represents a typedef name defined
in some source file, and each edge `T → U` means "the type specifier of `T` contains
the typedef name `U`".

The graph is built per source file during the same pass as Stage 2. Typedef names
from imported parcels are added as nodes with their expansions taken from the import
record (which carries the fully-expanded type from its source parcel).

### Cycle detection

A cycle in the type graph represents a recursive typedef, which is not valid in C
except through struct indirection. If a cycle is detected (DFS back-edge), the
Parcelator reports an error and skips expansion for the affected names.

---

## 5. Stage 4 — Type expansion

### Algorithm

Type specifier expansion proceeds by post-order DFS over the type graph:

1. For each leaf node (a primitive type, struct tag, union tag, or enum tag), the
   expanded form is the node itself.
2. For each interior node (a typedef name `T` with specifier containing typedef
   names `U₁ … Uₙ`), the expanded form of `T` is the specifier with each `Uᵢ`
   replaced by `expand(Uᵢ)`.

### Scope rules

Typedef names are resolved in the following order of precedence:

1. Names defined in the same translation unit (before the point of use).
2. Names imported from other parcels via `#include "import/..."` directives that
   precede the point of use.
3. Primitive C types (`int`, `char`, `void`, `float`, …) — always available.
4. `struct`, `union`, `enum` tags — treated as primitive (not expanded further).

A typedef name that cannot be resolved in any of these scopes causes a type expansion
error.

### Same-parcel exception for import files

When generating an import file that contains multiple typedef declarations from the
same parcel, later declarations may reference earlier ones by their stem-prefixed form
without further expansion. This mirrors the behaviour of the original source file and
keeps the import file self-contained.

### Struct member expansion

Type specifier expansion applies to struct member types in both export and import
files (see SEMANTICS.md). If a struct member type contains a typedef name that is not
available in the scope of the generated file, it must be expanded to its primitive
form, potentially as an anonymous struct type.

---

## 6. Stage 5 — Validation

The following diagnostics are emitted exactly as specified in
[TRANSLATION.md](https://github.com/nbyoung/parcel/blob/main/TRANSLATION.md).

### Errors (halt generation for affected files)

| Condition | Detection point |
|-----------|----------------|
| Undefined identifier | Stage 2: identifier listed in pragma has no declaration in same file |
| Orphan export | Stage 1: export include present with no matching pragma in same file |
| Undefined export | Stage 1: import include present with no corresponding export include for that name in any scanned file |
| Path-name collision | Stage 1: two distinct `(path, name)` pairs produce the same canonical prefix |
| Stem collision | Stage 1: two import includes in the same file apply the same stem |

### Warnings (generation proceeds)

| Condition | Detection point |
|-----------|----------------|
| Unexported parcel | Stage 1: pragma present with no corresponding export include in same file |
| Duplicate interface identifier | Stage 1/2: same identifier listed more than once in the same parcel, including across cumulative same-name declarations |
| Exported static identifier | Stage 2: exported identifier is declared `static` |
| Canonical name conflict | Stage 4: a generated canonical name matches a user-defined identifier in the same file |
| Unused import | Stage 1 + Stage 2: import stem present but no corresponding stem-qualified identifiers appear in the file |

All diagnostics report the source file path, line number, and a descriptive message.
The exit code is non-zero if any errors were emitted.

---

## 7. Stage 6 — Code generation

Generated files are written into `_parcel/` relative to the project root (or a
configurable output root). The directory structure mirrors the namespace paths.

### Export file — `_parcel/export/<path>/<name>`

**Typedef-only parcel** (all exported identifiers are typedefs):

```
(empty file)
```

**Parcel with variables, constants, or functions**:

```c
struct <canonical> {
    <xT> * const <id>;           /* variable */
    const <xT> * const <id>;     /* constant */
    <xR> (* const <f>)(<xP>);    /* function */
} const <canonical> = { &<id>, &<id>, <f> };
```

Where `<xT>`, `<xR>`, `<xP>` are the fully expanded type specifiers from Stage 4.

The export file is included at the end of the exporting source file (after all
definitions), so all identifier names it references are in scope without forward
declaration.

### Import file — `_parcel/import/<path>/<name>.<stem>`

```c
typedef <xT> <stem>_<Id>;    /* one per exported typedef */

struct <canonical> {
    <xT> * const <id>;
    const <xT> * const <id>;
    <xR> (* const <f>)(<xP>);
};
extern const struct <canonical> <canonical>;
const struct <canonical> *<stem> = &<canonical>;
```

For the typedef declarations:
- The identifier is substituted: `<Id>` → `<stem>_<Id>`, placed at the same
  syntactic position in the declarator.
- The type specifier is fully expanded (`<T>` → `<xT>`), with the same-parcel
  exception applied for self-referential typedefs within the same import file.

For the canonical struct re-declaration, member type specifiers are fully expanded
(possibly to anonymous struct form) so the import file is self-contained and does not
depend on the object parcel's import being in scope.

### Idempotency

Generated files are written only if their content has changed. This avoids spurious
recompilation when the parcelator is re-run without changes to the source.

---

## 8. Build integration

### Compiler include path

The C compiler must be told to search `_parcel/` for include files, so that
`#include "export/..."` and `#include "import/..."` resolve correctly:

```sh
cc -I _parcel [sources] ...
```

### Makefile integration

```make
PARCELATOR = parcelator
PARCEL_OUT = _parcel

.PHONY: parcel
parcel:
	$(PARCELATOR) .

# All object files depend on parcelation completing first
%.o: %.c | parcel
	$(CC) -I$(PARCEL_OUT) $(CFLAGS) -c $< -o $@
```

For incremental builds, the parcelator can emit a `.d`-style dependency file listing
which source files each generated `_parcel/` file depends on, allowing `make` to
re-run only the affected generations.

### Invocation modes

| Mode | Command | Use case |
|------|---------|---------|
| Whole project | `parcelator <root>` | Full build or CI |
| Single file | `parcelator <root> --file <src.c>` | Editor integration, fast iteration |
| Dry run | `parcelator <root> --dry-run` | Validation without writing files |
| Dependency output | `parcelator <root> --deps <out.d>` | Makefile incremental builds |
| Watch mode | `parcelator <root> --watch` | Interactive development |

---

## 9. Implementation language

### Recommendation: Python (Phase 1–2), Rust (Phase 3+)

**Python** is recommended for the initial implementation:

- The targeted lexer and pattern-matching approach requires modest parsing complexity
  that Python handles well with `re` and simple token-list processing.
- Rapid iteration is valuable while the semantics are still being refined.
- No compilation step simplifies distribution during the prototype phase.
- The existing `parcel/examples/` test fixtures can drive development immediately.

**Rust** is recommended for the production implementation:

- Strong type system makes the typed identifier records, type graph, and code
  generation pipeline easy to express correctly.
- Parser combinator libraries (`nom`, `pest`) provide a solid foundation for the
  C declaration parser.
- Compiled binary with no runtime dependency is better for toolchain integration.
- The Python prototype serves as a behavioural specification for the Rust port.

### Alternative: self-hosting in C with Parcel

A longer-term goal could be to implement the Parcelator in C using Parcel modules
for its own internal structure. This would serve as a demonstration of the language's
viability at scale, but is deferred until the semantics and tooling are stable.

---

## 10. Key data structures

```
ProjectModel
├── source_files: [SourceFile]
└── import_directives: [ImportDirective]

SourceFile
├── path: Path
├── parcels: [ParcelDeclaration]
└── imports: [ImportDirective]

ParcelDeclaration
├── name: str                   # parcel name, e.g. "stdout"
├── export_path: str            # e.g. "output/stdout"
├── canonical: str              # e.g. "output_stdout"
├── pragma_line: int
├── export_include_line: int    # -1 if absent (warning)
└── identifiers: [Identifier]

Identifier
├── name: str
├── kind: typedef | variable | constant | function
├── type_specifier: str         # as written in source
├── declarator: str             # verbatim declarator text
├── id_offset: int              # byte offset of name within declarator
└── expanded_type_specifier: str  # populated by Stage 4

ImportDirective
├── source_file: Path
├── path: str                   # namespace path
├── parcel_name: str
├── stem: str
└── line: int

TypeGraph
├── nodes: {name: TypeNode}
└── edges: {name: [name]}       # dependency edges

TypeNode
├── name: str
├── specifier: str              # as written
├── expanded: str               # populated after expansion
└── source_parcel: ParcelDeclaration | None
```

---

## 11. Testing strategy

### Unit tests

- **Lexer**: token stream correctness for representative C fragments including
  function-pointer typedefs, struct definitions, and multi-line declarations.
- **Declaration parser**: correct kind classification and type extraction for each
  declaration form in the grammar.
- **Type expansion**: expansion of chains of typedef names, anonymous struct
  expansion, same-parcel exception, cycle detection.
- **Code generation**: generated export and import file text matches expected output
  for each identifier kind.
- **Diagnostics**: each error and warning condition triggers the correct message and
  exit code.

### Integration tests — golden output

The existing hand-written `_parcel/` files in the example projects serve as golden
output for integration testing:

| Source | Expected output |
|--------|----------------|
| `parcel/examples/hello_world/` | `_parcel/export/output/_` (empty), `_parcel/export/output/stdout`, `_parcel/export/output/null`, `_parcel/import/output/_.out`, `_parcel/import/output/stdout.std`, `_parcel/import/output/null.null` |
| `parcelon/examples/hello_world/` | same structure, PARCELON types |

The integration test runs the parcelator against the source files, diffs the output
against the golden `_parcel/` files, and fails on any difference.

### Error case tests

The defects identified in the `paracee/parcel/` source analysis provide a ready-made
error test suite:

| Defect | Expected diagnostic |
|--------|-------------------|
| Export include before definitions | ordering error |
| Missing `lib/` prefix in import/export paths | undefined export |
| `IGetter` struct member named `put` instead of `get` | (no parcel error; C semantic issue only) |

### Regression suite

Each resolved diagnostic case is added to a permanent regression suite to prevent
reintroduction.

---

## 12. Implementation phases

### Phase 1 — Typedef parcels (MVP)

**Scope**: typedef-only parcels; no variables, constants, or functions; no type
expansion (all types are primitive).

**Deliverable**: the parcelator correctly processes `parcel/examples/hello_world/`
— it produces the empty export file for `output/_` and the two typedef declarations
in `import/output/_.out`, and the program compiles and runs.

**Validates**: discovery, typedef declaration parsing, simple import file generation,
compiler include path integration.

### Phase 2 — Variables, constants, functions

**Scope**: all identifier kinds; canonical struct generation in export files;
canonical struct re-declaration in import files; type expansion for primitive types.

**Deliverable**: both `hello_world` examples compile and run after parcelator
processing, with generated output identical to the hand-written golden files.

**Validates**: full export file generation, full import file generation, type
expansion for single-file typedef chains.

### Phase 3 — Cross-file type expansion

**Scope**: type specifier expansion for struct member types that reference typedef
names from imported parcels; anonymous struct inlining in import files; full
TRANSLATION.md diagnostics.

**Deliverable**: the `parcelon/examples/hello_world/` example compiles — this
requires correct expansion of the `out_Output` member type in the import file for
the `stdout` and `null` parcels.

**Validates**: cross-file type graph construction, anonymous struct generation,
complete diagnostic coverage.

### Phase 4 — Tooling and integration

**Scope**: incremental build support (dependency files, idempotent output, single-file
mode); watch mode; CMake module; IDE language-server hints.

**Deliverable**: a `parcelator` binary suitable for inclusion in a project's build
system with no manual `_parcel/` management required.
