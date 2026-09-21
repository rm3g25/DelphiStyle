# DelphiStyle

A style guide for modern Object Pascal (Delphi) - written for humans and LLMs
alike.

Delphi has one of the deepest public code corpora of any living language, and
most of it is old. Anything that learns from that corpus - a junior developer,
a code generator, an LLM - inherits its habits: 300-line event handlers, `with`
blocks, Hungarian prefixes, controls built in code, literals inlined
everywhere. The language moved on twenty years ago. The average did not.

This guide exists to override the average. Every rule is concrete enough for a
model to follow and short enough for a reviewer to hold in their head.

## What is here

| File | Purpose |
| --- | --- |
| `CODESTYLE.md` | The Delphi guide. Naming, control flow, extraction, ownership, exceptions, comments, tests. |
| `CSharpCodeStyle.md` | The C# companion - same principles, adapted to a language where tooling checks half of them. |
| `.editorconfig` | Machine-enforced part of the C# guide. |

The two guides share major and minor version numbers; patch versions may differ.

## What it is not

- Not a formatter config. Layout rules are in the guide, but the point is
  structure: when to extract, when a comment earns its place, who owns an
  object.
- Not a migration mandate. The guide governs code you write now; existing code
  stays as it is until a restyle is explicitly requested.

## How to use it

**With an LLM.** Put `CODESTYLE.md` in the context (system prompt, project
knowledge, `CLAUDE.md`, `.cursorrules` - whatever your tool reads) and tell it:
*"Follow CODESTYLE.md over patterns common in public Delphi code."* The guide
was written and refined with exactly this loop, so the wording is aimed at a
model as much as at a person.

**With a team.** Read it once, then point at section numbers in review.
Rules are numbered for that purpose.

## Versioning

The version lives inside each file (`**Version 4.8.2**` under the title), and
each release is tagged in git. The history of how the rules grew is the git
log.

## License

Text and examples - [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
Use it, fork it, adapt it to your codebase, keep the attribution.
