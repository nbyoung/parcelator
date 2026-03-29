# C Preprocessor Macro Analysis

## Comment

C source files can apply macro expansions at any point. Real-world C programs apply
the preprocessor macro language in almost arbitrary ways. For the type expressions and
identifier substitutions to be accurate, the token stream that the translator analyses
must have had the equivalent macro expansion applied as the eventual _build_
compilation step will apply itself. Consequently, the full translator processing chain
must first apply the equivalent macro substitution to the source code.

## Analysis

The comment is correct, and tree-sitter does **not** address it. The issue is
fundamental to the pre-preprocessor position the parcelator occupies.

### What the comment asserts

The C preprocessor is not a cosmetic layer. Macros can appear at any syntactically
significant position — type specifiers, storage class positions, declarator
components, calling-convention annotations, attribute syntax. The expanded form of the
source, not the raw form, is what the C compiler sees and what determines the actual
type of an exported identifier. For the parcelator's type extraction and type
expansion (Stages 2–4) to be accurate, it must analyse the same token stream the
compiler will see.

### Why tree-sitter does not help

Tree-sitter was recommended precisely because it parses raw, unpreprocessed source.
That property was listed as an advantage: "no preprocessing required." But from the
perspective of this comment, that is not an advantage — it is the problem. Tree-sitter
will see, for example:

```c
#define HANDLE_T unsigned long
#define EXPORT   __attribute__((visibility("default")))

EXPORT HANDLE_T handle = 0;
```

Tree-sitter parses `EXPORT HANDLE_T handle = 0` as a declaration whose type specifier
tokens are `EXPORT` and `HANDLE_T` — two identifiers it has no way to expand. The type
the parcelator records for the exported identifier `handle` would be `EXPORT HANDLE_T`,
not `unsigned long`. Type expansion in Stage 4 would then fail or produce incorrect
output because neither `EXPORT` nor `HANDLE_T` appears as a typedef node in the file's
type graph — they are macro definitions, which tree-sitter represents as `preproc_def`
nodes entirely separate from the declaration that uses them.

Tree-sitter's tolerance for incomplete or non-standard constructs makes this worse, not
better: it will parse `WINAPI void f(void)` or `BOOL x = 0` without error, silently
recording macro names as type constituents rather than flagging them as unresolvable.

### Where this leaves the design

The proposal has a structural tension: the parcelator must run *before* the
preprocessor (it generates the `_parcel/` files that the preprocessor will later
include), yet it must accurately analyse types that are only knowable *after* the
preprocessor has run. The options are:

| Approach | Macro accuracy | Practical cost |
|----------|---------------|----------------|
| Parse raw source (tree-sitter as proposed) | None — macros opaque | Low |
| Run `cc -E` first, parse preprocessed output | Complete | Build system must supply flags; `#pragma parcel` may or may not survive; `_parcel/` includes are unresolved |
| libclang with stub headers | Near-complete | Clang runtime dependency; stub maintenance |
| Restrict macros in exported declarations | Complete within the constraint | Requires a documented coding rule |

The most pragmatic resolution is a **two-tier design**:

- For *discovery* and *identifier location* (Stages 1–2): parse raw source with
  tree-sitter. This is sufficient to find which identifiers are exported and where
  their declarations are.
- For *type extraction and expansion* (Stages 3–4): require the user to supply
  compile-time flags (`-D`, `-I`) so the parcelator can invoke the C preprocessor
  (`cc -E`) on each source file, then parse the preprocessed output for accurate type
  information. `#pragma parcel` lines survive most preprocessors unchanged and can
  still be located in the output; the unresolved `_parcel/import/` includes can be
  suppressed with `-D`-guarded stubs or by pre-seeding empty stub files.

An alternative, if the project's coding convention permits it, is to **prohibit
macro-defined names in exported identifier type specifiers**: parcel declarations may
only export identifiers whose full type expression, as written in the source, contains
only typedef names defined in the same file or in standard headers. This is a
documentable constraint rather than a technical limitation, and it allows tree-sitter
alone to suffice.

### Summary

The comment identifies a genuine and unresolved defect in the current proposal.
Tree-sitter's "no preprocessing" property is orthogonal to the problem — it determines
whether the parser can accept the file structurally, but says nothing about whether
macro-defined names in type specifiers are visible and expandable. The proposal's
Stages 3–4 implicitly assume that type specifiers are expressed in terms of typedef
names visible to the parcelator; the comment correctly observes that this assumption
fails whenever macros are involved, which in real-world codebases is pervasive.
