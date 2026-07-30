# CODESTYLE.md

**Version 4.2**

Modern, strict, homogeneous Object Pascal (Delphi). This document is the
**source of truth** for any human or LLM writing or reviewing code in this
project. When generating code, follow these rules over statistically common
patterns found elsewhere.

The guiding principle behind every rule below: **code is read far more often
than it is written.** Optimize for the reviewer six months from now, not for
the fastest way to make it compile today.

### Why this document exists
Without an explicit style guide, an LLM defaults to the pattern most common in
its training data, not the pattern that is architecturally best. For this stack
that default skews toward long monolithic methods, controls created in code, and
literals inlined everywhere - because that is how the task most often appears in
public code. Delphi makes this worse than most: the public corpus is decades
deep and largely pre-modern, so the statistical pull runs toward the old way.
This document exists to override that default. The more concrete the rule, the
less the generated code slides back into the statistical average.

### The extraction principle
One move recurs throughout this guide under different names. Naming it once so
the later sections can point at it instead of restating it:

> **Extraction principle: when a thing is shared, lift it into its own named
> home and let the dependency flow one way.**

Its three appearances:
- Fields that travel together → lift into a record (§10).
- A generic nested inside a generic → lift the inner type into a named type
  (§10).
- Two units that reference each other → lift the shared type into a third unit
  both depend on (§12).

The failure mode it prevents is always the same: something with no home of its
own gets smeared across the places that need it.

---

## 1. Naming

### Local variables
- **PascalCase, no prefixes.**
- **No single-letter names** except a `for` loop counter.
- The name must carry meaning. A bare `Task` is acceptable only when the
  context makes it unambiguous what task it is; otherwise use two words
  (`CurrentTask`, `PendingTask`). If you have to guess later what it held, the
  name failed.

```pascal
// BAD
var
  F: TSettingsForm;   // single-letter AND masquerading as a field

// GOOD
var
  SettingsForm: TSettingsForm;
```

### Loop counters
The single exception to PascalCase. Lowercase, single letter, declared inline:

```pascal
for var i := 0 to List.Count - 1 do
```

Nested loops: `i`, `j`, `k`. When the index itself is not needed and only the
element matters, prefer `for-in` - it removes any chance of an off-by-one at
the boundary:

```pascal
for var Item in List do
  Process(Item);
```

### Fields
Prefix `F`, PascalCase:

```pascal
private
  FAgentThread: TAgentThread;
```

### Properties
Clean PascalCase, no prefix:

```pascal
property AgentThread: TAgentThread read FAgentThread;
```

### Parameters
Prefix `A`, PascalCase. The `A` prefix deliberately separates parameters from
both fields (`F`) and locals (no prefix):

```pascal
procedure SetTask(const AValue: string);
```

### Type-family prefixes
- Types: `T` - `TAgentClient`
- Interfaces: `I` - `IAgentTransport`
- Exceptions: `E` - `EAgentError`

### Constants
PascalCase. **Not** `SCREAMING_CASE` - that is a C convention and reads as
foreign in modern Pascal:

```pascal
// GOOD
MaxRetries = 3;

// BAD
MAX_RETRIES = 3;
```

No `c` prefix either (`cMaxRetries`). Whether a name is a constant or a
variable is information you need at exactly one point - when you try to assign
to it, where the compiler stops you for free - not on every read. The name
should carry the meaning (`MaxRetries` - a retry ceiling); const-ness is
metadata that does not earn a character on every line. (Contrast `F` on
fields, which *is* needed on every read, to tell object state from a local.)

### Enumeration members
Keep the classic short-prefix style:

```pascal
TAgentState = (asIdle, asRunning, asStopped);
```

### Acronyms are words, not shouting
Capitalize an acronym as a normal word: `Http`, `Url`, `Json`, `Api`, `Sql`.
Not `HTTP`, `URL`, `JSON`. All-caps acronyms collide into unreadable walls the
moment two of them meet:

```pascal
// BAD
HTTPURLParser: THTTPJSONClient;

// GOOD
HttpUrlParser: THttpJsonClient;
```

The one exception is a single-word type where no collision is possible and the
RTL already established the spelling (`TJSONObject` from the RTL stays as the
RTL named it - do not rename other people's types).

### Unit names: no `u` prefix, dotted subsystem namespaces
**New projects: drop the `u` prefix.** `uMenu.pas` is a relic of flat folders,
weak IDEs, and Hungarian notation. It carries zero information - the same
letter on every file distinguishes nothing - and every original reason for it
(spotting your units in a flat list, avoiding name collisions) is solved better
by folders, the IDE, and namespaces. The RTL itself dropped it long ago:
`System.SysUtils`, `FMX.Forms`.

Group by subsystem with a dotted namespace instead:

```
// BAD
uMenu.pas
uMonsterParser.pas
uSprites.pas

// GOOD
Menu.pas
Monsters.Parser.pas
Render.Sprites.pas
```

Do **not** prepend the project name (`Moon.Monsters.Parser.pas`). It is armor
against a collision that does not happen in a project without competing
third-party units - the RTL needs it, you do not.

**Legacy projects keep their prefix.** A codebase already full of `u*.pas`
stays that way: uniformity beats modernity. Mixing the two styles in one folder
is worse than either style consistently applied.

### No double negations
`IsValid`, not `IsNotInvalid`. A name that must be mentally inverted at every
call site (`if not IsNotValid then`) is read three times instead of once.

### Symmetric verbs for symmetric operations
Pick one verb pair per axis and stick to it: `Load/Save`, `Read/Write`,
`Build/Parse`, `Open/Close`. Mixing pairs (`Load` next to `Store`, `Read` next
to `Save`) forces the reader to check whether the asymmetry means something.
It never does - it just means two authors or two moods.

---

## 2. Variables

### Declaration placement
- **Needed throughout the method** → declare in the method's `var` section.
- **Lives only inside a scope/branch** → declare inline at the point of use.

### Type inference
Use inference when the type is obvious from the right-hand side. Spell the type
out when the right-hand side does not make it clear:

```pascal
var Count := List.Count;          // obvious - Integer, inference is fine
var ContentArr: TJSONArray := ...;  // not obvious from RHS - state it
```

### `const` on parameters
Apply `const` only where it gives a real benefit - strings, arrays, large
records, interfaces (avoids a copy / refcount churn). Do **not** stamp it on
every `Integer` "for symmetry". `const` is a signal, not decoration.

---

## 3. Formatting

- **No aligned columns.** Do not pad with spaces so that colons or types line
  up under each other. That style is dead; the padding is maintenance debt with
  zero payoff.
- Colon tight to the name, one space, then the type.

```pascal
// BAD - aligned columns
var
  HTTP        : TNetHTTPClient;
  ReqBody     : TJSONObject;
  EndpointURL : string;

// GOOD
var
  Http: TNetHTTPClient;
  ReqBody: TJSONObject;
  EndpointUrl: string;
```

### Line width and indentation
- **Soft limit 100 columns.** Past 100, wrap. Under 100, use judgement - a hard
  limit produces ugly breaks to save three characters, no limit produces
  worm-length lines.
- **Two spaces per level, never tabs.** The IDE default, and what the RTL is
  written in.

The goal is not to imitate the RTL for its own sake - it is that people who read
RTL sources every day should not bleed from the eyes reading yours.

### Wrapping parameter lists
Fits on one line → one line. Does not fit → continuation indented one level:

```pascal
// GOOD
function SendRequest(const ASystemPrompt: string; AIncludeTools: Boolean;
  const ANudge: string): TJSONObject;
```

Do not put each parameter on its own line - that is a C#/Java habit, not a
Pascal one. **Five or more parameters is not a wrapping problem, it is a
signature smell**: apply the travel test from §10 and group the ones that
always move together into a record.

### Wrapping boolean conditions
A condition is wrapped only when it does not fit. Do not break a short one to
look symmetrical:

```pascal
// GOOD - fits, so one line
if (Response.StatusCode = 429) or (Response.StatusCode = 529) then

// GOOD - does not fit; operator stays at the end, continuation indented
if (Response.StatusCode = 429) or (Response.StatusCode = 529) or
  (FRetryPolicy.Mode = rmAggressive) and (FAttemptCount < FMaxAttempts) then
```

Operator at the **end** of the line, not the start. Leading operators scan
better in the abstract, but they read as foreign in Delphi - consistency with
the surrounding ecosystem wins, the same way it did for uppercase compiler
directives.

**Three or more terms in a condition is a candidate for a named function.** Even
correctly wrapped, the example above reads poorly; the real fix removes the wrap
entirely:

```pascal
if IsRetryable(Response.StatusCode) and FRetryPolicy.ShouldRetry(FAttemptCount) then
```

### File-level: line endings and encoding
These are non-negotiable and machine-checkable. Tools that generate Pascal
source get both wrong by default - LF-normalized, BOM-less - so state them
explicitly.

- **Line endings: CRLF.** RAD Studio warns on LF-terminated source
  (`Line endings are LF, but RAD Studio requires CRLF`). Every `.pas`, `.dpr`,
  `.dpk` uses CRLF.
- **Encoding: UTF-8 with BOM.** Without the BOM the IDE reads the file as ANSI
  in the system codepage, and any non-ASCII byte - a Cyrillic
  `resourcestring` value, an em dash in a comment - turns to mojibake. ASCII-only
  files survive either way, but do not rely on staying ASCII-only forever.

Do not rely on discipline alone - pin it in the repository so it holds
regardless of who or what wrote the file:

```gitattributes
*.pas text eol=crlf
*.dpr text eol=crlf
*.dpk text eol=crlf
*.dfm text eol=crlf
```

---

## 4. Control flow

### Guard clauses
Prefer early exit over a nested `if` ladder. Guards flatten the method and keep
the happy path un-indented:

```pascal
if not Assigned(AConfig) then
  Exit;
if AConfig.ApiKey = '' then
  raise EAgentError.Create(SMissingApiKey);
// ... main logic, un-nested
```

### Single statement after `if` / `for`
No `begin/end` for a single statement. Keep one-liners as one line:

```pascal
for var Text in AMessages do
  AppendUserMessage(Text);
```

### `with` - do not use it
No exceptions in new code. `with` is the one legacy construct with no case left
in its favour:

- **Silent shadowing.** `with Proxy do Port := 8080;` compiles and works. The day
  someone adds a `Port` field to the enclosing class, that same line silently
  writes somewhere else - no error, no warning. Adding a field changes the
  behaviour of code that did not move.
- **`with A, B do` resolves right to left.** Unreadable, and that is not an
  exaggeration.
- **The debugger goes blind** - identifiers inside a `with` often cannot be
  evaluated or inspected.
- **It defeats grep.** `Port := 8080` will not be found by searching for
  `Proxy.Port`. In a large legacy codebase, a symbol you cannot search for is a
  symbol you cannot maintain.

The only argument for `with` was brevity on deep access, and inline `var` does
that better:

```pascal
// BAD
with FConfig.Network.Proxy do
begin
  Host := 'localhost';
  Port := 8080;
end;

// GOOD - explicit, greppable, visible to the debugger
var Proxy := FConfig.Network.Proxy;
Proxy.Host := 'localhost';
Proxy.Port := 8080;
```

If the target is a **record**, that local is a copy - assign it back (§10,
property-copy trap). For a class instance it works as written.

Legacy code keeps its `with` blocks; do not sweep through old units rewriting
them.

---

## 5. Subprograms vs methods

Deciding whether a piece of logic becomes a **nested subprogram** or a
**private method** uses a three-step test. Apply them in order:

1. **Used in more than one method?** → private method. No further thought.
2. **Used only here, but does NOT touch the outer method's locals** (only its
   own parameters and class fields)? → still a private method, just with a
   single caller for now. It is cheaper to have it already portable than to
   dig it out of someone else's `begin...end` in two weeks.
3. **Used only here AND genuinely captures the outer method's locals?** → only
   now is a nested subprogram justified.

> **Rule:** Extract to a private method if the subprogram does not capture the
> outer method's local variables - regardless of how many callers it currently
> has. Keep as a nested subprogram only what physically cannot live without the
> parent's context.

```pascal
// PROMOTE to a method: reaches only FModelName (a field), captures no local.
function TAgentClient.BuildRequestBody: TJSONObject;
begin
  Result := TJSONObject.Create;
  Result.AddPair('model', FModelName);
end;

// KEEP nested: AppendUserMessage captures History, a local of the outer method.
// Promoting it would mean passing History as a parameter - noise for nothing.
procedure TAgentClient.RunConversation(const AMessages: TArray<string>);
var
  History: TJSONArray;

  procedure AppendUserMessage(const AText: string);
  begin
    var Entry := TJSONObject.Create;
    Entry.AddPair('role', 'user');
    Entry.AddPair('content', AText);
    History.Add(Entry);
  end;

begin
  History := TJSONArray.Create;
  try
    for var Text in AMessages do
      AppendUserMessage(Text);
    SendHistory(History);
  finally
    History.Free;
  end;
end;
```

### Nesting budget
Indentation is a speedometer, not a price. Two spaces on a three-line helper is
nothing. Two spaces that grow their **own** two spaces inside is the signal:
you have built a program inside a program - unwind it into methods.

- **One level of nesting**: fine.
- **Second level**: yellow card.
- When the main `begin` disappears under a mountain of nested declarations and
  you have to hunt for the entry point like a hidden stash - extract to methods.
  The main `begin` should read like a table of contents, not a treasure map.

### Naming subprograms
Verb phrases, **no prefix**: `BuildRequestBody`, `AppendUserMessage`,
`ParseResponse`. The only prefix reservation is `Do`-methods bound to events
(`DoChange` and relatives from the RTL) - those are virtual class methods and
have nothing to do with nested subprograms.

---

## 6. Methods and classes

### Single Responsibility - one reason to change
A method or class should have **one reason to change**, not one verb in its
description. The distinction matters: a method can technically "do one thing"
from the outside (send a request, return a response) while having four
independent internal reasons to be edited (message format, retry policy, cache
scheme, log format).

The practical test: **how many different, unrelated bosses could walk in and
ask to rewrite this for reasons that have nothing to do with each other?**
- One boss with a complex request → one method, even if the logic is large.
- Several bosses from different departments → split, even if each asks for one
  line.

Do not over-apply this into false granularity. Splitting `ValidateEmail` into
`ValidateEmailFormat`, `ValidateEmailLength`, `ValidateEmailDomain` when they
all change for the same reason ("the email rules changed") is noise, not
cleanliness - three files to open where the rule is one.

### Long methods → pipelines
Long methods are discouraged. The trigger for extraction is not a line count:

> **Rule:** If you can draw a horizontal line inside a method where a new
> logical phase begins with minimal variable overlap with the phase above, that
> is a candidate for extraction - regardless of the method's length in lines.
> Length is a consequence, not the cause.

A 200-line method with a single phase (e.g. an honest hand-written parser going
line by line) may stay as is. A 20-line method with three phases should be cut.
The top-level method should then read as a pipeline - build, log, send, check,
parse - visible top to bottom without scrolling.

```pascal
function TAgentThread.SendRequestAnthropic(const ASystemPrompt: string;
  AIncludeTools: Boolean; const ANudge: string): TJSONObject;
var
  ReqBody: TJSONObject;
  ReqStr: string;
begin
  ReqBody := BuildAnthropicRequestBody(ASystemPrompt, AIncludeTools, ANudge);
  try
    ReqStr := ReqBody.ToString;
    LogRequest(ReqStr);
  finally
    ReqBody.Free;
  end;

  var Response := PostAnthropicWithRetry(BuildAnthropicEndpointUrl, ReqStr);
  var RespStr := Response.ContentAsString(TEncoding.UTF8);
  LogResponse(Response.StatusCode, RespStr);

  if Response.StatusCode <> 200 then
    raise EAgentError.CreateFmt('HTTP %d from Anthropic: %s',
                              [Response.StatusCode, RespStr]);

  Result := TJSONObject.ParseJSONValue(RespStr) as TJSONObject;
  if Result = nil then
    raise EAgentError.CreateFmt('Failed to parse Anthropic response: %s', [RespStr]);

  LogAnthropicUsage(Result);
end;
```

### Call sites must read without opening the signature
Do not pass arguments whose meaning is invisible at the call site. Boolean
literals are the worst offender:

```pascal
// BAD - what does True mean here? Go find the signature to learn.
SendRequest(SystemPrompt, True, Nudge);

// GOOD - the call site explains itself.
type
  TToolsMode = (tmWithTools, tmNoTools);
SendRequest(SystemPrompt, tmWithTools, Nudge);
```

Delphi has no named arguments, so the fix is an enum or a named constant. The
same applies to magic sentinel values (`DoWork(-1)`) - if the reader must open
the signature to decode an argument, the argument is misnamed or mistyped.

### Reusable construction helpers
When the same structure is built more than once (e.g. five tool schemas that
differ only in name/description/params), extract the shared shape into a helper
and let the top level declare only what differs. The duplication test (used in
more than one place → extract) applies to a repeated *template*, not just to a
repeated *fragment*.

```pascal
function MakeTool(const AName, ADesc: string;
                  AProps: TJSONObject; AReq: TJSONArray): TJSONObject;
var
  Params: TJSONObject;
  Func: TJSONObject;
begin
  Params := TJSONObject.Create;
  Params.AddPair('type', 'object');
  Params.AddPair('properties', AProps);
  Params.AddPair('required', AReq);

  Func := TJSONObject.Create;
  Func.AddPair('name', AName);
  Func.AddPair('description', ADesc);
  Func.AddPair('parameters', Params);

  Result := TJSONObject.Create;
  Result.AddPair('type', 'function');
  Result.AddPair('function', Func);
end;
```

### Format over concatenation
Prefer `Format` over string concatenation for anything beyond a trivial join.

### Obvious first, fast later
Write the obvious version first. Optimize only after profiling or benchmarking
identifies a real bottleneck - not when a hypothetical one is imagined. An
optimization without a proven bottleneck is complexity paid up front for a
benefit that may never arrive.

---

## 7. Constants and resource strings

### Constant vs literal in place
Turn a literal into a constant when it is **either** used more than once **or**
carries a hidden meaning the literal itself does not explain. Otherwise leave it
inline.

**The sharper form of the same rule: numbers are magic, text usually is not.**
A bare number never explains itself - why 20 and not 15 or 30? It needs a name
to carry its meaning. A string often *is* its own meaning: `'legacygrep.log'`
reads exactly as what it is. So a magic number is almost always a constant; a
one-off string usually is not.

- **Protocol keys / fixed-vocabulary values** (JSON keys, schema type values,
  tool names read in two or more places across the system) → **constants**. A
  typo in a string literal is never caught by the compiler; a typo in a
  constant name is caught immediately.
- **Parameter keys local to one tool**, human prose descriptions used exactly
  once in exactly one place → **leave inline**. Wrapping them in constants
  scatters what belongs together and buys nothing.

```pascal
// GOOD - magic number becomes a named constant near its origin,
// with a comment explaining WHERE the number comes from (not what it does).
const
  MaxToolCallsPerTurn = 20; // API limit: rejects requests with more tool_use blocks

if ToolCalls.Count > MaxToolCallsPerTurn then
  raise EAgentError.Create(STooManyToolCalls);
```

### Constant placement - three levels
1. **Shared constants unit** (project / subsystem) - used across multiple modules.
2. **Top of the module** - used only in this module, but in several places
   (this also covers a constant needed by several methods of one class - it is
   "more than one method", so it goes here, not duplicated per method).
3. **`const` section in the method declaration**, before `var` - used only by
   that one method.

`private const` inside a class is reserved for genuinely class-owned values
(an ORM class's table name, a protocol version) - things that have no meaning
without that class. It is not for "a number two methods happen to touch"; that
goes to the top of the module.

### `resourcestring` vs `const`
- **`resourcestring`** - text a **human** reads and that may need translation
  (error messages surfaced to the user, `EAgentError` message text). These land
  in a separate resource section and can change or be localized without
  recompiling logic.
- **`const`** - protocol strings a **machine** reads (JSON keys, fixed
  vocabulary, identifiers). Never a resource string - a protocol cannot be
  translated; it would break.

```pascal
resourcestring
  STooManyToolCalls = 'Too many tool calls in single turn';
```

---

## 8. Comments

A comment is not a sin in itself. It becomes one when it compensates for
something that should have lived in a name or in the structure. Four categories:

1. **What the code does** → delete it. Rename or restructure instead. If a
   comment is needed to explain *what* the code does, the code is written badly.
   ```pascal
   // BAD
   Inc(RetryCount); // increment retry counter
   ```
2. **Why** → keep, if not self-evident. The code honestly says *what*; it cannot
   say *where a magic value came from* or *why a decision was made*.
   ```pascal
   // API rejects requests with more than 20 tool_use blocks per turn
   ```
3. **Warning about fragile / non-obvious order** → keep, but as a fallback,
   not a first resort. First ask whether the architecture can make the wrong
   order impossible (encapsulate the teardown in one method, transfer
   ownership so only one owner frees). When enforcement would cost more than
   it protects - e.g. wrapping two adjacent `Free` calls in machinery - the
   warning comment stays legitimate. No variable name can express that two
   adjacent lines carry order-dependent meaning.
   ```pascal
   // Order matters: FHttpClient must be freed before FConnection,
   // otherwise pending requests crash on the dangling connection
   FHttpClient.Free;
   FConnection.Free;
   ```
4. **TODO / FIXME with context** → keep, but with a real reference, not a
   hieroglyph. `// TODO: fix this` is litter buried in code.
   ```pascal
   // TODO: switch to streaming once the SDK supports SSE (tracked: issue #47)
   ```

### `//` vs `{ }`
- **`//`** - all real comments, always. This is the default for every category
  above.
- **`{ }`** - only for temporarily disabling a block of code during debugging,
  or a block header at the very top of a file/unit (license, authorship, unit
  overview). Used to explain logic mid-method, `{ }` reads as leftover debug
  litter - it is visually indistinguishable from commented-out code. That usage
  is a relic of pre-`//` Turbo Pascal.

---

## 9. Free functions vs class methods

- **Free function in a unit** - the default for anything not bound to a specific
  class by meaning. Putting `EscapeJsonString` into `TAgentClient` as a
  `class function` lies about ownership: it is about strings, not about agents.

  ```pascal
  function EscapeJsonString(const AText: string): string;
  ```

- **`class function` / `class procedure`** - only when the work belongs to the
  class as a concept but needs no instance: factory methods, class-level state,
  validation that knows something class-specific.

  ```pascal
  type
    TAgentClient = class
    public
      class function CreateDefault: TAgentClient;                    // factory
      class function IsValidModelName(const AName: string): Boolean; // class-bound knowledge
      class var InstanceCount: Integer;                              // shared class state
    end;
  ```

  The test: **does this logic depend on being on this particular class, or would
  it work for anything?** Depends → `class function`. Does not → free function.

- **Separate module** - when the function is used in more than one place in the
  project. Not before. Do not spawn a `Utils.pas` for a single function used in
  one place.

---

## 10. Grouping related data into records

Two tests - either one firing is enough to group:

1. **The prefix test.** Several fields sharing a name prefix are a record
   asking to be born. The prefix already *is* the record's name, just smeared
   across four lines:

   ```pascal
   // BAD - a prefix herd
   FProxyHost: string;
   FProxyPort: Integer;
   FProxyUser: string;
   FProxyPassword: string;

   // GOOD
   type
     TProxySettings = record
       Host: string;
       Port: Integer;
       User: string;
       Password: string;
       function Enabled: Boolean; // small derived helpers are fine
     end;
   // ...
   FProxy: TProxySettings;
   ```

2. **The travel test.** Variables that always move together - passed together
   as parameters, saved together, validated together - belong in one record,
   even without a shared prefix.

**Record vs class:** a record (value semantics, no lifetime, nothing to free)
for passive data bundles - settings, coordinates, operation results. A class
when the thing has identity, lifetime, or behavior beyond storing. Advanced
records may carry small derived helpers (`Enabled`, a `Default` factory) but
no business logic - a record that starts making HTTP calls is a class in a
trench coat.

**The property-copy trap.** A property of record type returns a **copy**.
Mutating a field through the property either fails to compile or silently
edits a temporary:

```pascal
// TRAP - edits a copy (or does not compile)
Client.Proxy.Port := 8080;

// CORRECT - take the whole record, change it, assign it back
var Proxy := Client.Proxy;
Proxy.Port := 8080;
Client.Proxy := Proxy;
```

Inside the owning class, access the record **field** (`FProxy.Port := 8080`)
directly - the trap only exists on the property path.

### Generics: depth by expedience, not by fashion
A generic earns its place when it removes a cast or a duplication **and** its
signature reads in one pass. `TList<TMonster>` instead of `TList` with `as
TMonster` on every access - yes: safer and shorter at the point of use.
`TDictionary<string, TMovementKind>` - yes, one level, read at a glance.

The rule is the principle of expedience applied to type machinery: exactly as
much as carries meaning, not one turn more.

> **Rule:** Type-parameter nesting deeper than one level → give the inner type
> a name. `TDictionary<string, TList<TMonster>>` is the boundary;
> `TObjectDictionary<string, TList<TPair<Integer, TMonster>>>` is a puzzle, not
> a type. The cure is not "avoid generics" - it is the extraction principle
> (see intro): name the inner type.

```pascal
// BAD - a clause inside a clause; read to the end, forget the start
FGroups: TObjectDictionary<string, TList<TPair<Integer, TMonster>>>;

// GOOD - the inner type gets a name, the outer generic is one level again
type
  TMonsterGroup = class ... end;   // wraps the inner TList<TPair<...>>
// ...
FGroups: TObjectDictionary<string, TMonsterGroup>;
```

A one-level generic that still feels unfamiliar is a matter of mileage, not
bad code - write it, get used to it. A nested one is objectively hard for
everyone, author included - do not learn to read it, learn not to write it.

---

## 11. Conditional compilation

- **Directives and symbols in uppercase**: `{$IFDEF DEBUG}` ... `{$ENDIF}`,
  symbols like `MSWINDOWS`, `CONSOLE`. This is the convention of the RTL/VCL
  sources and virtually all modern code; lowercase `{$ifdef}` reads as ported
  FPC.
- **Compound conditions** use the modern `$IF` form with `Defined` spelled as
  the intrinsic function it is:
  `{$IF Defined(MSWINDOWS) and not Defined(CONSOLE)}`. Close with `{$ENDIF}`
  (not the legacy `{$IFEND}`).
- When the closing `{$ENDIF}` sits far from its opening, add a trailing
  comment: `{$ENDIF} // MSWINDOWS`.
- **The real rule: an `{$IFDEF}` scattered mid-logic is a smell.** Each symbol
  should be tested in one place - behind a function, a constant, or a unit -
  so that calling code reads unconditionally. Ifdef confetti through a method
  body is code that reads differently depending on which platform you hold in
  your head, which means it does not read at all.

```pascal
// BAD - the logic is interleaved with platform noise
procedure TAgent.Log(const AText: string);
begin
  {$IFDEF MSWINDOWS}
  OutputDebugString(PChar(AText));
  {$ENDIF}
  {$IFDEF CONSOLE}
  Writeln(AText);
  {$ENDIF}
end;

// GOOD - one function owns the platform difference; every caller reads
// unconditionally. The ifdefs live in exactly one place, not in the logic.
procedure WriteToPlatformLog(const AText: string);
begin
  {$IF Defined(MSWINDOWS)}
  OutputDebugString(PChar(AText));
  {$ELSEIF Defined(CONSOLE)}
  Writeln(AText);
  {$ELSE}
  // no platform sink available - deliberately silent
  {$ENDIF}
end;

procedure TAgent.Log(const AText: string);
begin
  WriteToPlatformLog(AText); // reads the same on every platform
end;
```

---

## 12. Unit structure: `uses` placement

Split the `uses` clause by contract, not by habit.

- **`interface uses`** - only modules whose types appear in the `interface`
  section: method-parameter types, field types, ancestors, return types -
  anything visible in a public signature. If a type shows outside the unit, its
  module belongs here, or callers will not compile.
- **`implementation uses`** - everything needed only by method bodies. If a
  module is used internally but never surfaces in any signature, it goes below
  `implementation`. This is the **preferred default**, not a mere option: keep
  the interface thin.

The one-line test: **is the type visible in the `interface` section?** → up.
**Visible only in method bodies?** → down.

Why thin-interface is the default:
- **Transparent contract.** `interface uses` should read as what the unit is
  coupled to *as a contract*, not as every incidental helper it calls inside.
- **Build cost.** `interface uses` leaks transitively - everyone who uses your
  unit inherits your interface dependencies. `implementation uses` does not.
- A module in `implementation` is then an honest signal - "internal detail, not
  part of my contract" - rather than a hiding place.

### Units included for side effects need a comment
Some units are listed not for any symbol you reference, but for what their
`initialization` section does - registering a driver, a codec, a factory. Their
names appear nowhere in the code, so the next person to "clean up unused uses"
will delete them and the program will fail at runtime, not at compile time.
Mark them:

```pascal
uses
  // link-only: registration happens in their initialization sections.
  // Referenced by no symbol - do not "clean up".
  FireDAC.Phys.SQLite,
  FireDAC.Stan.Async,
  FireDAC.DApt;
```

### Circular dependencies: fix by architecture, not by pushing `uses` down
Delphi rejects two units that reference each other through `interface uses`
("circular unit reference"). Moving the `uses` to `implementation` makes the
compiler accept it - but that is treating a symptom. A cycle usually means the
boundary is drawn in the wrong place: the two units share something that belongs
to neither.

The fix is not to bury the dependency downward so it compiles - it is the
extraction principle (see intro): lift the shared type or interface into a third
unit both depend on, dependency flowing one way. Pushing `uses` down to tolerate
a cycle is a last resort for legacy knots, not a design tool.

---

## 13. Forms and dynamic controls

The form designer exists precisely so that static UI can be laid out once and
forgotten. Use it.

> **Rule:** Put static UI in the designer (`.dfm`). Create controls in code only
> when they genuinely behave dynamically - a list of unknown length, controls
> materialized from a query result, anything that physically cannot be laid out
> ahead of time.

Building a static form entirely in code because it is "cleaner" ignores the tool
the language gives you for exactly this job.

---

## 14. Memory management and ownership

Delphi memory is manual and its mistakes are silent - a leak or a
use-after-free compiles cleanly and fails later. Ownership must be explicit, not
inferred at the point of freeing.

### Explicit ownership
Create in the constructor, free in the matching destructor. The create/destroy
pair is symmetric like `Load/Save` - one owner, one lifetime. If ownership is
clear, no method should ever have to check whether it is allowed to free
something.

### `Free` vs `FreeAndNil` - choose by lifetime, not by ritual
- A local that dies at method end → plain `Free`. Nil-ing it is pointless; it
  is about to go out of scope.
- A field that may be accessed or recreated later → `FreeAndNil`, so a later
  access fails loudly on `nil` instead of reading freed memory (a heisenbug).

What to avoid is `FreeAndNil` - or `if Assigned(X) then X.Free` - used as a
talisman against not knowing who owns the object. If you genuinely do not know
whether you own it, the ownership model is broken; fix that, do not paper over
it with a guard.

Legitimate exceptions where the nil check is **not** a smell:
- a destructor running after a constructor that raised halfway through (fields
  past the failure point are still `nil`);
- a field being deliberately torn down and rebuilt (`FreeAndNil(FConn);
  FConn := TConn.Create(...)`), where `nil` between the two lines is a valid
  transient state.

### `nil` as a value vs `nil` as confusion
Same syntax, opposite meaning - the canon must not confuse them:
- `nil` meaning "optional / not found / not set" is a **valid state**. Guard it
  with `if X = nil then Exit` as much as you like - not a smell.
- `nil` used to **guess** whether something still needs freeing is the smell.

---

## 15. Exceptions

### Raise your own type, never bare `Exception`
A caller cannot distinguish `Exception` from any other disaster, so it cannot
handle it selectively. Define an exception per domain (`EAgentError`,
`EMonsterDefError`) and raise that.

```pascal
// BAD - indistinguishable from out-of-memory
raise Exception.CreateFmt('HTTP %d: %s', [Code, Body]);

// GOOD
raise EAgentError.CreateFmt('HTTP %d: %s', [Code, Body]);
```

### `Create` goes before `try`, not inside it
`try/finally` is an ownership tool, not error handling. If the constructor
raises inside the `try`, `finally` runs against an uninitialized variable.

```pascal
// GOOD
Http := TNetHttpClient.Create(nil);
try
  ...
finally
  Http.Free;
end;
```

For several objects in one scope, either nest the blocks, or nil them all before
a single `try`:

```pascal
Http := nil;
Body := nil;
try
  Http := TNetHttpClient.Create(nil);
  Body := TJSONObject.Create;
  ...
finally
  Body.Free;
  Http.Free;
end;
```

This is a **named exception to §14**: here the pre-nil is the mechanism that
makes a single `finally` correct, not a talisman against unknown ownership.

### Catch narrowly - with two deliberate exceptions
Catch the type you can actually act on (`on E: EMonsterDefError`). A blanket
`on E: Exception` also swallows `EOutOfMemory` and `EAccessViolation`, which you
cannot meaningfully handle. An empty `except end` is the worst construct in the
language.

Two places where a broad catch is **correct**, not sloppy:
- the top of a thread's `Execute` - otherwise an exception vanishes silently;
- a plugin/script boundary that must not take the host down.

### `try/except` is not a talisman
Wrapping a method in `except` so that "it does not crash" hides the breakage
instead of handling it - the same disease as a defensive `FreeAndNil` (§14).
Catch only where you have something to do with what you caught.

### Exceptions are not control flow
"Not found" and "did not parse" are results, not catastrophes - return `Boolean`
with an `out` parameter instead of raising.

This is where the RTL `Try` convention applies: `Try*` returns `Boolean` and
does not raise (`TryStrToInt`, `TryGetValue`, `TryEncodeDate`); the plain name
raises. The payoff is at the call site - `if TryParseX(S, Value) then` announces
that failure is routine, while `Value := ParseX(S)` announces that failure needs
a handler.

Provide **one** of the pair, not both, unless both are genuinely called - the
RTL ships both because it serves a million callers; your unit serves one
scenario. Choose by the nature of the failure: recoverable at the call site →
`Try`; a sign of corrupt input → raise with context. A bad `movement.kind` in a
monster definition means the data file is broken, so `ParseMovementKind` should
raise `EMonsterDefError` naming the monster and the field - a `Try` variant there
would invite silently defaulting to `mkStatic` and losing an afternoon to it.

---

## 16. Tests

Everything above applies to test code unchanged - naming, `uses` placement,
ownership, no aligned columns. Test-only helper classes (mocks, builders) are
implementation details: declare them in `implementation`, not `interface`.

Four rules are specific to tests:

### Name the behavior, not the method
A test name is read when it goes red. `TestAcquire` tells you nothing;
`TestAcquireReturnsActive` tells you what broke before you open the file.

```pascal
// BAD - names the method under test
procedure TestIncFail;

// GOOD - names the expected behavior
procedure TestIncFailRaisesCountAndStampsTime;
```

Pick one convention across the suite and hold it - mixed styles cost more than
either style.

### DRY is weaker in tests than in production code
This is the one place the guide reverses itself. A test must be readable **in
place**: if understanding what is being verified requires jumping to three
helpers, the extraction traded the wrong thing.

- Duplicated **test data** (the same host and port in five tests) - acceptable.
- Duplicated **setup logic** (building the same fixture graph by hand five
  times) - extract.

The test is documentation that happens to execute; documentation you have to
chase across the file has failed at its main job.

### Every test must be able to fail
If you cannot name the breakage it would catch, it is not a test.

```pascal
// BAD - cannot fail; the constructor cannot return nil
procedure TProxyCheckerTests.TestCheckerCreatedWithUrl;
begin
  Assert.IsNotNull(FChecker);
end;
```

A test that passes no matter what is worse than no test: it costs maintenance
and pays in false confidence.

Multiple asserts in one test are fine when they describe **one** behavior
(initial state, a state transition). One-assert-per-test is dogma, not a rule.

### Generated files are out of scope
DUnitX project files, IDE scaffolding, designer output - do not restyle them.
Lowercase locals in a generated `.dpr` are not a violation to fix; the next
regeneration erases the edit anyway. The guide governs code you author.
