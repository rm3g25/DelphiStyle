# CSharpCodeStyle.md

**Version 4.8.2**

Pragmatic modern C#. This document is the **source of truth** for any human or
LLM writing or reviewing C# in this project. Version numbers are shared with
`CODESTYLE.md` (Delphi) - major and minor move together, patches may differ.

The guiding principle: **code is read far more often than it is written.**
Optimize for the reader who comes after you, not for the fastest way to make
it compile today.

Two supporting principles, applied throughout:

- **Expedience.** Exactly as much machinery as carries meaning, not one turn
  more. Neither the oldest way nor the newest - the one that pays for itself.
- **Uniformity.** A consistently applied older style beats a mix of two styles.

### Reading the examples
The guide speaks only in the label line above a block - `// BAD - ...`,
`// GOOD - ...` and their relatives. Inside a BAD block a comment may point at
the defect. Inside a GOOD block - and an unlabeled block is a GOOD block -
every comment is a real one and passes the erasure test (§5). `// ...` marks
elided code, not a comment. A GOOD example is a sample to imitate, and an LLM
imitates the comments along with the code.

### Why this document is short
Half of a style guide's traditional content - indentation, brace placement,
naming casing, `var` policy - is **enforced by `.editorconfig` and analyzers**,
checked by the build, not by a reviewer. Rules a machine verifies do not need
prose here. See the companion `.editorconfig`.

"Checked by the build" is only true with three project properties set (a
`Directory.Build.props` at the repository root does it for every project at
once): `Nullable` on, `EnforceCodeStyleInBuild` on, and an `AnalysisLevel`
that pulls in the CA rules. Without them the IDE shows squiggles and the build
says nothing; the header of `.editorconfig` lists the exact lines.

What remains is what tooling cannot check: extraction, naming *meaning*,
comments, ownership, async discipline, and a short list of constructs that
compile fine and are still wrong.

### What gets written down
A rule is written down only where the default output would otherwise be wrong -
outdated, unsafe, or grandma code. Things that come out correct without being
asked (field naming, `using` blocks, string interpolation, LINQ basics) are
deliberately absent. Absence means "no problem here", not "unregulated".

### Scope: the guide governs code you author
Every rule applies to the code being written now - new types, new methods, and
the lines a change actually touches. None of them is a mandate to bring the
surrounding code up to date.

- **Existing code that breaks a rule stays as it is** until a restyle is
  explicitly requested. Not "while I am here", not "it was only three lines".
- **A point change stays a point change.** Fix the line, not its neighbours.
  Formatting counts - and `dotnet format` on a whole file is a restyle, not a
  fix.
- **A restyle is its own change**, shipped separately from any behaviour
  change, so each diff can be read for one intent.
- **New code inside an old file follows the guide** - it is authored code -
  except where the file has a visible local convention the guide contradicts.
  There the uniformity principle above wins.

Later appearances: generated files (§13).

Why this needs saying: a diff that mixes a fix with tidying cannot be reviewed
for either, and a guide read as a licence to sweep turns every task into a
rewrite of whatever it happened to touch.

---

## 1. The extraction principle

One move recurs under different names:

> **When a thing is shared, lift it into its own named home and let the
> dependency flow one way.**

Its appearances:
- Values that travel together → lift into a `record`.
- A repeated construction shape → lift into a factory method.
- Two classes that reference each other → lift the shared contract into an
  interface or a third type both depend on.

The failure it prevents is always the same: something with no home of its own
gets smeared across the places that need it.

---

## 2. Methods and structure

### Long methods → pipelines
The trigger for extraction is not a line count:

> **If you can draw a horizontal line inside a method where a new logical phase
> begins with minimal variable overlap with the phase above, that is a candidate
> for extraction - regardless of length.**

A long method with a single phase may stay. A 20-line method with three phases
should be cut. The top-level method then reads as a pipeline - build, send,
check, parse - visible without scrolling.

### One reason to change
A method or class should have **one reason to change**, not one verb in its
description. The test: how many different, unrelated bosses could ask to rewrite
this for reasons unconnected to each other? One boss with a complex request →
one method. Several from different departments → split.

Do not over-apply into false granularity: splitting `ValidateEmail` into three
methods that all change for the same reason is noise, not cleanliness.

### Call sites must read without opening the signature
Boolean literals are the worst offender:

```csharp
// BAD - what does true mean here?
SendRequest(prompt, true, nudge);

// GOOD - named argument, or an enum
SendRequest(prompt, includeTools: true, nudge);
SendRequest(prompt, ToolsMode.Include, nudge);
```

C# has named arguments - use them at the call site rather than inventing an enum
for every flag. An enum is worth it when the flag has more than two states or
appears in many calls.

### Blank lines phrase the body
A blank line marks a boundary between phases too small to deserve their own
method. The test: **does the subject change?** Constructing an object, then
mutating outer state, are two subjects.

Specific places, not everywhere:
- **Not** after `{`, **not** before `}`.
- **Not** between every statement - air everywhere is air nowhere.
- **Not** between lines sharing a subject - four property assignments on the
  same object are one phrase.
- More than four or five groups means the method wants extraction instead.

### Local function vs private method
Three questions, in order:

1. **Used from more than one method?** → private method.
2. **Used only here, but captures nothing from the enclosing method** (only its
   own parameters and fields)? → still a private method, or a `static` local
   function if it is genuinely a detail of this one caller. Either way it is
   already portable; `static` is the compiler-checked promise that it captures
   nothing.
3. **Used only here and genuinely captures a local?** → only now is a
   non-static local function justified.

A lambda assigned to a delegate variable (`Func<int, bool> isValid = x =>
...`) is a local function that lost its name and its debugger frame - write the
local function.

### Lambdas: one level, and no async where a void is expected
An expression lambda inside a LINQ chain is an argument, not a nesting level -
chain length is governed in §11. The problem starts when a lambda has a
statement body **and** contains another statement-bodied lambda:

- **A block lambda inside a block lambda - never.** Each one is a nesting
  level that arrives with no `if` or `foreach` in sight, and callback stairs
  are the one way a method blows the §3 budget without a single branch. The
  inner one becomes a named method or local function; what it needed from the
  outer scope becomes a parameter.
- **A method group beats a lambda that only forwards.** `.Select(x =>
  Transform(x))` is `.Select(Transform)`.
- **An `async` lambda passed where `Action` is expected becomes `async void`**
  (§8) - exceptions escape and crash the process. Pass it where `Func<Task>`
  is expected, or do not pass it at all.

```csharp
// BAD - two levels of capture, the logic is two indents from daylight
client.Send(request, response =>
{
    parser.Parse(response, result =>
    {
        ...
    });
});

// GOOD - one level, and the inner step has a name and a signature
client.Send(request, response => parser.Parse(response, HandleParsed));
```

---

## 3. Control flow

### Guard clauses
Prefer early return over a nested `if` ladder. Guards keep the happy path
un-indented, and every guard makes the whole body beneath it cheaper, not one
line. At a public entry point the guard is `ArgumentNullException.ThrowIfNull`
or a domain exception; inside the type, §7 already proves what is not null -
do not re-guard it.

```csharp
// GOOD - two guards, then the happy path at zero depth
if (config is null)
    return;
if (string.IsNullOrEmpty(config.ApiKey))
    throw new AgentException(MissingApiKey);
// ...
```

### Nesting budget
Every level of nesting is paid by every line beneath it: a statement three
levels deep is read with three conditions held in the head. Count the control
structures a statement sits inside - `if`, `switch`, loops. `try` and `using`
do not count: they bracket a lifetime, they do not branch.

- **Two levels**: fine. A loop with a filter inside it is everyday code.
- **Third level**: yellow card. First look for a guard or a `continue` that
  flattens it; if none applies, the body from the second level down becomes a
  method with a name.
- **Fourth level**: not written. Extract before it exists.

```csharp
// YELLOW - three deep, and the real work sits at the bottom of the stairs
foreach (var item in items)
    if (item.Enabled)
        switch (item.Kind)
        {
            ...
        }

// GOOD - a guard flattens the filter, a method owns the decision
foreach (var item in items)
{
    if (!item.Enabled)
        continue;
    ProcessItem(item);
}
```

### Mixed `&&` / `||` in one condition: name the parts
A chain of one operator reads as a list - `a && b && c` is three things that
must all hold, however long it gets. The moment `&&` and `||` meet in one
expression the reader is evaluating precedence (`&&` binds tighter), not
intent.

> **Rule:** `&&` and `||` mixed in one expression → give each homogeneous
> part a name. A local when the fact is local to the method; a method when it
> is needed a second time.

```csharp
// BAD - the reader is doing precedence, and the parentheses admit it
if (owner == _userId && active || role is Role.Admin or Role.Root)

// GOOD - two facts, one decision
var isOwner = owner == _userId && active;
var isAdmin = role is Role.Admin or Role.Root;
if (isOwner || isAdmin)
```

`!` over a bracketed group is the same problem in a hat - name the group.

### `switch` over an `else if` ladder
Three or more branches that all test the **same value** are a `switch`, not an
`if / else if` ladder. A `switch` compares one value against a list of
patterns and is taken in at a glance; a ladder may compare anything against
anything on every rung, so every rung has to be read to confirm it still tests
the same thing. Type tests are the same case: a ladder of `is` checks is a
`switch` with type patterns.

- **Every arm produces a value** → switch expression, not a `switch`
  statement with a `return` per arm.
- **The value is a string of your own vocabulary** (a kind, a mode, a state)
  → `switch` can take the string, but the string should have been an enum:
  the compiler then knows the set is closed and warns on a missing arm.
- **The value is a protocol string** (JSON field, command line) → decode it
  into an enum once at the boundary (§6 keeps the spellings as constants), and
  `switch` on the enum everywhere else. One place knows the spelling.

```csharp
// BAD - grows a rung per kind, and each rung re-reads kind
if (kind == "file") ...
else if (kind == "dir") ...
else if (kind == "link") ...

// GOOD - the string is decoded once, the logic reads the enum
var handler = ParseItemKind(kind) switch
{
    ItemKind.File => HandleFile,
    ItemKind.Dir => HandleDir,
    ItemKind.Link => HandleLink,
    _ => throw new ArgumentOutOfRangeException(nameof(kind)),
};
```

Two branches stay an `if / else`. A ladder whose rungs test **different**
values is not a `switch` candidate - it is a decision table, and usually a sign
the branches want to be separate methods.

---

## 4. Naming that tools cannot check

`.editorconfig` enforces casing. What it cannot enforce:

- **Names carry meaning.** No single letters except loop counters and lambda
  parameters in short expressions. A bare `task` or `data` passes only when the
  context makes it unambiguous.
- **No double negations.** `IsValid`, never `IsNotInvalid`.
- **Symmetric verbs for symmetric operations.** `Load/Save`, `Read/Write`,
  `Open/Close`, `Serialize/Deserialize`. Do not mix pairs.
- **`Async` suffix on every method returning `Task`/`Task<T>`.** This is a
  contract signal, not decoration - it tells the caller to await.
- **`Try` prefix for the non-throwing variant**, returning `bool` with an `out`
  parameter, mirroring `int.TryParse`. Ship **one** of the pair unless both are
  genuinely called: choose `Try*` when failure is routine at the call site, and
  the throwing form when failure means the input is corrupt.

---

## 5. Comments

The code should read as a story on its own; a comment is a rare insert where
the story falls silent.

### The addressee is the reader five years from now
Not today's reviewer. A comment that explains a **change** - why something was
replaced or removed, why the new way beats the old - answers a question nobody
asks a year later: the file holds no "before". Its home is the commit message
or the PR. Generated code likewise: an explanation for the reviewer goes in the
reply or the commit, not into the method body.

### The erasure test
> **Erase the comment. Would the next conscientious developer, making a
> routine edit, "fix" the code into a bug? Yes → it stays, as short as
> possible. No → it goes.**

"Would understand it slower" is a naming problem, not a break. Three kinds pass,
all about what code cannot say:

- **Why not the obvious alternative** - the rejected option looks like an
  improvement.
- **External fact** - an API limit, a protocol constraint, the origin of a
  number (§6).
- **Trap** - an order that carries meaning, a sync-over-async bridge that
  exists on purpose (§8). Fallback, not first resort: first ask whether the
  structure can make the wrong edit impossible.

One fact, one comment: what is said at the declaration is not repeated at the
use.

### Before a comment, try a name - but a name is not free
A name replaces *what*, never *why*. Climb the cost ladder, stop at the first
rung that works: rename what exists → a local or a constant → an enum → a
method, and the method only if it passes §2 on its own merits (a phase line or
a second caller). Nothing fits → a blank line marks the phase. Extraction is
not a way to give three lines a heading.

Litmus for the name: longer than four or five words, or with `And` inside - it
is a comment in PascalCase. `ValidateInputAndLogFailureUnlessQuiet` is not a
method, it is a confession.

### Two exceptions to the erasure test
- **TODO / FIXME with a real reference.** It is a debt receipt, not an
  explanation, so it lives only with a tracker reference. `// TODO: fix this`
  is litter.
- **XML docs on public API** - below.

### A comment explains the code, not how the code came to be
No references to this guide, no names, no "as agreed", no "requested by", no
edit history ("was X", "fixed the crash").

```csharp
// BAD - cites the guide; rots on renumbering, helps no reader
// Static helpers - no instance state (guide, 9)

// BAD - attributes the decision to a person
// Ilia's decision: keep the timeout at 30 seconds

// GOOD - states the reason, which is what the reader needs
// HttpClient is per-thread here; created in the handler, not the constructor
```

Three reasons: git already stores authorship and history and stores them
correctly; provenance is not the knowledge the reader needs; and a name turns a
technical decision into a question of authority - while the comment says "the
gateway drops at 35s", anyone can verify and change it, but once it says whose
decision it was, disputing the decision means disputing the person.

### XML docs on public API only
`/// <summary>` on public members of a library or shared contract. Do not
generate doc comments that restate the signature (`/// <summary>Gets the
name.</summary>` on `Name`) - that is a *what* comment with extra ceremony.

---

## 6. Magic values

**Numbers are magic; text usually is not.** A bare number never explains itself -
why 20 and not 15? A string often is its own meaning: `"app.log"` reads as what
it is.

- Magic number → named constant, with a comment on the *origin* if it comes from
  outside (an API limit, a protocol constraint).
- One-off string used once → leave it inline.
- Protocol keys, fixed vocabularies, names read in two or more places → constants,
  because a typo in a literal is never caught by the compiler.

```csharp
// API rejects requests with more tool blocks per turn
private const int MaxToolCallsPerTurn = 20;
```

---

## 7. Nullability

**Enable `<Nullable>enable</Nullable>` project-wide** and treat its warnings as
real. This is the single largest difference between modern C# and grandma C#:
the compiler tracks what can be null, if you let it.

**`<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` project-wide**, alongside
the properties listed above. A nullable warning that does not stop the build is
noise, and noise is ignored by the second hundred lines - the compiler's null
tracking is worth exactly as much as the attention it gets. Obsolete-API
warnings (CS0618, CS0612) are the one exception: list them in
`WarningsNotAsErrors` so a framework deprecation does not stop work, and fix
them as they appear.

- **Do not sprinkle `!` (null-forgiving) to silence a warning.** It is the exact
  analog of a defensive null check that hides not knowing the invariant. Use it
  only where you can state why the compiler is wrong - and prefer restructuring
  so it is not wrong.
- Nullable annotations are a contract: `string?` in a signature means callers
  must handle null, and `string` means they must not pass it.
- Guard public entry points; do not re-guard internal calls that the type system
  already proves non-null.

```csharp
// BAD - silences the compiler, keeps the bug
var name = user!.Profile!.DisplayName!;

// GOOD - handle it once, at the boundary
if (user?.Profile is not { } profile)
    return DefaultName;
var name = profile.DisplayName;
```

---

## 8. Async

The largest source of code that compiles and is still wrong.

### Async all the way down
Do not bridge sync and async by blocking. `.Result`, `.Wait()`, and
`GetAwaiter().GetResult()` deadlock in any context with a synchronization
context (WinForms, WPF) and starve the thread pool everywhere else.

```csharp
// BAD - deadlock in UI code, thread-pool starvation on a server
var data = FetchAsync(url).Result;

// GOOD
var data = await FetchAsync(url);
```

If a synchronous entry point genuinely cannot be avoided (a `Main` before C# 7.1,
a legacy interface), isolate that bridge in exactly one place and comment why.

### `async void` only for event handlers
`async void` cannot be awaited and its exceptions cannot be caught by the caller -
they crash the process. The only legitimate use is an event handler, whose
signature forces it. Everywhere else return `Task`.

```csharp
// GOOD - forced by the event signature; wrap the body so nothing escapes
private async void OnSaveClicked(object sender, EventArgs e)
{
    try
    {
        await SaveAsync();
    }
    catch (IOException ex)
    {
        ShowError(ex.Message);
    }
}
```

### Flow `CancellationToken` through the whole stack
Every async method that can be waited on takes a `CancellationToken` and passes
it down. Omitting it produces work that cannot be stopped - a defect that only
shows up under load or on shutdown.

```csharp
public async Task<Report> BuildAsync(int id, CancellationToken ct)
{
    var rows = await _db.QueryAsync(id, ct);
    return await RenderAsync(rows, ct);
}
```

### `Task.Run` is for CPU-bound work only
Wrapping synchronous I/O in `Task.Run` does not make it async - it moves the
blocking to a pool thread and calls it progress. Use a genuinely async API
instead. `Task.Run` is correct for pushing real computation off a UI thread.

### `ConfigureAwait(false)` in libraries
In library code that must not depend on the caller's context, append
`ConfigureAwait(false)` to awaits. In ASP.NET Core application code it is
unnecessary (no synchronization context) and adds noise - do not add it there
reflexively.

---

## 9. Exceptions

### Throw your own type
A caller cannot handle `Exception` selectively. Define an exception per domain
and throw that. Derive from `Exception`; do not derive from `ApplicationException`
(a historical dead end).

### Catch narrowly - with two deliberate exceptions
Catch the type you can act on. A blanket `catch (Exception)` also swallows what
you cannot handle.

Two places where a broad catch is **correct**:
- the top of a background task or thread, so failures are logged rather than lost;
- a plugin or request boundary that must not take the host down.

An empty `catch { }` is never correct.

### Rethrow with `throw;`
`throw ex;` resets the stack trace and you lose the origin. Inside a `catch`,
plain `throw;` preserves it.

### `try/catch` is not a talisman
Wrapping a method so that "it does not crash" hides the breakage. Catch only
where you have something to do with what you caught.

### Exceptions are not control flow
"Not found" and "did not parse" are results, not catastrophes. Return a `bool`
with `out`, a nullable, or a result type - see the `Try` convention in §4.

---

## 10. Ownership and disposal

GC handles memory; it does not handle **resources**. Ownership of anything
`IDisposable` must be explicit.

- **The creator disposes.** Wrap it in a `using` declaration at the point of
  creation.
- **Do not dispose what was injected.** A dependency handed to you by the DI
  container is owned by the container. Disposing an injected `HttpClient`,
  `DbContext`, or logger is a bug that surfaces later, in another request.
- **A class holding disposable fields is itself disposable.** Implement
  `IDisposable` and dispose them; do not leave the decision to whoever reads the
  class later.
- Async resources implement `IAsyncDisposable` - `await using`.

```csharp
// GOOD - created here, so disposed here
await using var stream = File.OpenRead(path);

// GOOD - injected, so NOT disposed here
public sealed class ReportService(HttpClient client)
```

---

## 11. Types: record, class, struct

- **`record`** - immutable data carriers: DTOs, value objects, messages,
  configuration. Value equality and `with`-copies come free, and the declaration
  is one line instead of thirty of boilerplate.
- **`class`** - anything with identity, lifetime, or behavior beyond holding
  values.
- **`struct`** - small and **immutable** only. Mutable structs produce silent
  copy bugs: mutating one through a property or a collection edits a temporary
  and the change vanishes. Do not write them. When a struct is genuinely
  wanted, `readonly record struct` says immutable and gives value equality in
  the same line.

```csharp
// GOOD
public record ProxySettings(string Host, int Port, string? User);

// BAD - mutable struct; edits silently apply to copies
public struct Counter { public int Value; }
```

### Group values that travel together
Several parameters or fields that always move as a set are a `record` asking to
be born - the same prefix (`proxyHost`, `proxyPort`, `proxyUser`) is the record's
name smeared across three names. **Five or more parameters is a signature smell**,
not a wrapping problem.

### Generic depth
Generics are ordinary in C# - `Dictionary<string, List<Order>>` is a normal
signature, not a puzzle, and LINQ returns generic types constantly. The threshold
here is looser than in older languages: worry when a type argument needs a
type argument of its own *and* the whole thing no longer reads in one pass, and
then name the inner type.

The analogous smell is **chain length**: a LINQ pipeline of eight operations is a
clause inside a clause. Break it into named intermediate variables, each stating
what that stage produced.

---

## 12. Constructs to avoid

Each of these compiles and is still wrong.

- **`#region`.** It hides bulk instead of reducing it; a class needing regions to
  be navigable is a class asking to be split. Collapsed regions also hide code
  from review.
- **Static mutable state as a default.** A `static` manager or singleton holding
  application state is the classic grandma pattern: untestable, and a race
  waiting for a second thread. Prefer dependency injection with an explicitly
  chosen lifetime. `static` is fine for pure functions and true constants.
- **`dynamic`.** Only at an interop boundary (COM, a genuinely shapeless JSON
  payload) and converted to a real type immediately. Beyond that it discards the
  entire benefit of the language.
- **Fake async** - see §8.
- **Reflection to reach private members** outside of tests and frameworks. It
  compiles today and breaks silently on a rename that no compiler will catch.

---

## 13. Tests

Everything above applies to test code unchanged. Four rules are test-specific:

### Name the behavior, not the method
A test name is read when it goes red. `AcquireTest` tells you nothing;
`Acquire_ReturnsActiveProxy_WhenPoolNotEmpty` tells you what broke before you
open the file. Pick one convention and hold it.

### DRY is weaker in tests than in production code
The one place this guide reverses itself. A test must be readable **in place**:
if understanding what is verified requires jumping through three helpers, the
extraction traded the wrong thing.

- Duplicated **test data** - acceptable.
- Duplicated **setup logic** - extract.

### Every test must be able to fail
If you cannot name the breakage it would catch, it is not a test.

```csharp
// BAD - cannot fail; the constructor cannot return null
Assert.NotNull(_checker);
```

Multiple asserts are fine when they describe **one** behavior. One-assert-per-test
is dogma, not a rule.

### Generated files are out of scope
Scaffolding, designer output, generated clients - do not restyle them. The next
regeneration erases the edit (scope rule, intro).
