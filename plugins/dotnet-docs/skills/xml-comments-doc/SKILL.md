---
name: xml-comments-doc
description: >-
  Guides the agent in writing accurate and complete XML documentation comments
  for .NET C# code. Use this skill when adding or improving <summary>, <param>,
  <returns>, <exception>, <remarks>, <example>, or other XML doc tags on public
  or internal APIs, types, and members.
license: MIT
metadata:
  author: Antonello Provenzano
  version: "1.0"
  compatibility:
    - github-copilot
    - claude-code
    - openai-codex
    - opencode
---

# XML Documentation Comments

## Purpose

This skill helps agents write clear, consistent, and complete XML documentation comments (`///`) for C# .NET code. It covers every standard XML doc tag, explains what to include in each section, and establishes quality criteria so that generated or reviewed documentation is useful to consumers of APIs, IDEs IntelliSense, and documentation-generation tools such as DocFX or Sandcastle.

## When to Use

- Adding XML documentation comments to public or internal types and members
- Reviewing and improving existing XML doc comments for correctness, completeness, or clarity
- Documenting method parameters, return values, thrown exceptions, and generic type parameters
- Writing `<remarks>` or `<example>` sections that explain non-obvious behavior or usage
- Applying `<inheritdoc>` correctly to overridden or interface-implementing members
- Ensuring documentation is compatible with DocFX, Visual Studio IntelliSense, or NuGet package publishing

## When Not to Use

- Writing Markdown documentation files for the solution (use the **markdown-solution-doc** skill instead)
- Generating API reference sites or configuring DocFX pipelines
- Writing code comments that are not XML doc comments (e.g., inline `//` comments)
- Documenting non-.NET codebases

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Target type or member | Yes | The class, struct, interface, enum, method, property, field, or event to document |
| Existing code | Yes | The C# source or signature to annotate |
| Visibility scope | No | Whether to document only `public`, or also `internal` and `private` members |
| Documentation style preference | No | Tone (e.g., imperative vs. declarative sentence style for `<summary>`) |
| Tool compatibility | No | DocFX, Sandcastle, or plain IntelliSense — affects tag selection and formatting |

## Workflow

### Step 1: Identify what needs documentation

Scan the provided code and determine:

- Which types (classes, structs, interfaces, enums, delegates) lack or have incomplete XML doc comments
- Which members (constructors, methods, properties, fields, events) are public or internal and undocumented
- Whether the target is part of a NuGet package or internal library — public NuGet APIs must be documented exhaustively
- Whether `<InternalVisibleTo>` or explicit internal docs are expected

Prioritize in this order: public API surface → internal contracts → protected members → private members (document private members only if they encode non-obvious invariants).

### Step 2: Choose the right tags for each member

Use this tag selection guide:

| Member kind | Required tags | Conditional tags |
|-------------|--------------|------------------|
| Class / struct / interface | `<summary>` | `<remarks>`, `<typeparam>`, `<seealso>` |
| Enum | `<summary>` | `<remarks>` |
| Enum value | `<summary>` | — |
| Constructor | `<summary>` | `<param>`, `<exception>`, `<remarks>` |
| Method | `<summary>`, `<returns>` (if non-void) | `<param>`, `<typeparam>`, `<exception>`, `<remarks>`, `<example>` |
| Property | `<summary>`, `<value>` | `<remarks>` |
| Field | `<summary>` | `<remarks>` |
| Event | `<summary>` | `<remarks>` |
| Delegate | `<summary>` | `<param>`, `<returns>`, `<typeparam>` |

#### Tag writing rules

- **`<summary>`**: Write a single sentence that states what the member is or does. Use imperative mood for methods (`Gets the user by identifier.`). Do not start with "This method" or "This class".
- **`<param name="...">`**: Describe what the parameter represents, not its type (the type is already in the signature). Keep it concise.
- **`<returns>`**: Describe what the method returns and under which condition, especially for nullable or collection returns.
- **`<exception cref="...">`**: List every exception the method can throw, including those from called methods when they propagate unchanged. Use the fully qualified or resolvable type name in `cref`.
- **`<typeparam name="...">`**: Describe the type parameter's purpose and any constraints not expressed by a `where` clause.
- **`<value>`**: Use on properties only. Describe what the property value represents, not how to set it.
- **`<remarks>`**: Provide context, design rationale, thread-safety guarantees, performance notes, or behavioral edge cases not obvious from the signature.
- **`<example>`**: Include a runnable or illustrative code snippet inside a `<code>` block when the usage is not immediately obvious.
- **`<see cref="...">`**: Reference related types or members inline. Always use `cref=` so tools can validate and hyperlink the reference.
- **`<seealso cref="...">`**: List related members at the end of a comment when the relationship is worth surfacing but not part of the main description.
- **`<inheritdoc>`**: Apply to members that override a base class or implement an interface. Use `<inheritdoc cref="..."/>` when inheriting from a specific base rather than the immediate parent.

### Step 3: Write the documentation

Apply these quality rules:

1. **Completeness**: Every public parameter, return value, and documented exception has a corresponding tag.
2. **Accuracy**: The description matches the actual behavior. Do not copy-paste misleading descriptions from similar members.
3. **Conciseness**: `<summary>` fits in one sentence. Use `<remarks>` for explanatory prose.
4. **Cref validity**: Every `cref=` attribute references a type or member that exists and is resolvable. Prefer fully qualified names when the referenced type is not in scope.
5. **Null semantics**: When a parameter or return can be `null`, document the null contract explicitly in `<param>` or `<returns>`. When `Nullable` annotations are enabled, the documentation must be consistent with the nullability annotations.
6. **Exception propagation**: Document all exceptions that are observable by callers, including `ArgumentNullException`, `ArgumentOutOfRangeException`, `InvalidOperationException`, and any domain-specific exceptions.
7. **Async and Task**: For `Task`-returning methods, describe what the task represents in `<returns>`. Note cancellation behavior when a `CancellationToken` parameter is present.

### Step 4: Apply `<inheritdoc>` correctly

Use `<inheritdoc/>` when:

- A class implements an interface and the method's behavior matches the interface contract exactly
- A subclass overrides a virtual method without changing its observable semantics

Use `<inheritdoc cref="IMyInterface.MyMethod"/>` (explicit cref) when:

- The class implements multiple interfaces and the documentation should clearly resolve to one
- The override diverges slightly from the base and explicit attribution avoids confusion

Do **not** use `<inheritdoc/>` blindly when:

- The override changes preconditions, postconditions, or thrown exceptions
- The base documentation is known to be missing or inaccurate

For members that override and extend the base behavior, write a full comment and include a `<remarks>` note referencing the base or interface via `<see cref="..."/>`.

### Step 5: Validate documentation coverage

After writing or updating documentation:

- Confirm that every public member has at least a `<summary>`
- Verify all `cref` references compile without warning (`CS1574`, `CS1584`, `CS1591`)
- Check that no parameter in the signature is missing a `<param>` tag
- Ensure `<exception>` tags match all documented throw sites in the method body
- Confirm `<returns>` is present for non-`void` and non-`Task` methods
- Validate that `<inheritdoc/>` is not applied to members where behavior diverges from the base

Enable `<GenerateDocumentationFile>true</GenerateDocumentationFile>` in the project file and treat `CS1591` (missing XML comment) as a warning or error to enforce coverage.

## Validation

- [ ] Every public type and member has at least a `<summary>` comment
- [ ] Every parameter has a matching `<param>` tag
- [ ] Non-void methods have a `<returns>` tag that describes the result and null conditions
- [ ] Every observable exception is documented with `<exception cref="...">`
- [ ] `cref` attributes reference valid, resolvable types or members
- [ ] `<inheritdoc>` is used only where the base documentation is accurate and behavior is unchanged
- [ ] `<remarks>` and `<example>` are present for non-obvious APIs
- [ ] `<value>` is present for properties where the meaning of the value is not obvious from the name
- [ ] Nullable return values and parameters are documented consistently with nullability annotations
- [ ] `CancellationToken` parameter cancellation behavior is described in `<remarks>` or `<param>`
- [ ] `Task`-returning methods describe the task outcome in `<returns>`
- [ ] No `CS1574`, `CS1584`, or `CS1591` warnings are produced when documentation generation is enabled

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| `<summary>` repeats the member name verbatim | Describe purpose, not name: `Gets the user.` not `Gets user.` |
| Missing `<exception>` for observable exceptions | Enumerate all exceptions, including `ArgumentNullException` for non-nullable params |
| `cref` attribute references a type in a different assembly without full qualification | Use the fully qualified name or add the assembly XML reference |
| `<inheritdoc/>` applied to an override that changes behavior | Write a full comment and add a `<remarks>` note about the deviation |
| `<returns>` omitted for nullable return types | Always describe null conditions: "Returns `null` if no user is found." |
| `<param>` describes the type instead of the value | Describe what the argument represents, not its type |
| `<example>` contains code that does not compile | Validate all example code before committing |
| Internal members not documented despite being part of a visible API boundary | Document internal members exposed via `InternalsVisibleTo` as if they were public |
| `<summary>` written in passive voice ("Is used to…") | Use active or imperative phrasing ("Initializes…", "Returns…", "Represents…") |
| Async method `<returns>` does not mention cancellation | Add a `<param name="cancellationToken">` and note cancellation in `<returns>` or `<remarks>` |

