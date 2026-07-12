# CODESTYLE.md

Modern, strict, homogeneous Object Pascal (Delphi). This document is the
**source of truth** for any human or LLM writing or reviewing code in this
project. When generating code, follow these rules over statistically common
patterns found elsewhere.

The guiding principle behind every rule below: **code is read far more often
than it is written.** Optimize for the reviewer six months from now, not for
the fastest way to make it compile today.

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

### Enumeration members
Keep the classic short-prefix style:

```pascal
TAgentState = (asIdle, asRunning, asStopped);
```

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

---

## 5. Subprograms vs methods

Deciding whether a piece of logic becomes a **nested subprogram** or a
**private method** uses a two-step test. Apply them in order:

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
    raise Exception.CreateFmt('HTTP %d from Anthropic: %s',
                              [Response.StatusCode, RespStr]);

  Result := TJSONObject.ParseJSONValue(RespStr) as TJSONObject;
  if Result = nil then
    raise Exception.CreateFmt('Failed to parse Anthropic response: %s', [RespStr]);

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
  class by meaning. If it does not need `Self` and is not about the class as a
  concept, it does not belong inside the class. Putting `EscapeJsonString` into
  `TAgentClient` as a `class function` lies about its ownership - it is about
  strings, not about agents.

  ```pascal
  function EscapeJsonString(const AText: string): string;
  ```

- **`class function` / `class procedure`** - only when the work genuinely
  belongs to the class as a concept but needs no instance. Legitimate cases:
  factory methods, class-level state, and validation logic that truly knows
  something class-specific that nobody outside knows.

  ```pascal
  type
    TAgentClient = class
    public
      class function CreateDefault: TAgentClient;                    // factory
      class function IsValidModelName(const AName: string): Boolean; // class-bound knowledge
      class var InstanceCount: Integer;                              // shared class state
    end;
  ```

  The test is the same duck test used for subprograms, one level up: ask not
  "can I make this a `class function`" but "**does this logic depend on being on
  this particular class, or would it work for anything?**" Depends → `class
  function`. Does not → free function.

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

// GOOD - one function owns the difference; every caller reads unconditionally
procedure WriteToPlatformLog(const AText: string);
```

---

## 12. Forms and dynamic controls

The form designer exists precisely so that static UI can be laid out once and
forgotten. Use it.

> **Rule:** Put static UI in the designer (`.dfm`). Create controls in code only
> when they genuinely behave dynamically - a list of unknown length, controls
> materialized from a query result, anything that physically cannot be laid out
> ahead of time.

Building a static form entirely in code because it is "cleaner" ignores the tool
the language gives you for exactly this job.

---

## Appendix: on generation defaults

Without an explicit style guide, an LLM defaults to the pattern most common in
its training data, not the pattern that is architecturally best. For this stack
that default skews toward long monolithic methods, controls created in code, and
literals inlined everywhere - because that is how the task most often appears in
public code. This document exists to override that default. The more concrete
the rule, the less the generated code slides back into the statistical average.
