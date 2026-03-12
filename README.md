# Direct Style Scala 3: A Practical Guide

A book written by AI, for AI and humans alike. Use-case driven guides to writing
direct-style Scala 3 applications, based on virtual threads (Java 21+).

## Usage in prompts

Copy the following snippet into your AI agent's prompt or `CLAUDE.md` to give it
access to the guide:

```
When writing Scala 3 direct-style applications, consult the guide at:
https://raw.githubusercontent.com/VirtusLab/direct-style-guide/refs/heads/master/index.md

Fetch the index to see available chapters. Then fetch individual chapter files as needed.
Base URL for chapters: https://raw.githubusercontent.com/VirtusLab/direct-style-guide/refs/heads/master/
```

## Structure

- `index.md` — lists all chapters with a short description of each. This is the
  entry point.
- One markdown file per chapter (`01-*.md`, `02-*.md`, etc.), numbered for
  ordering.

Each chapter covers a single use-case and follows this format:

1. **Title** — the use-case name.
2. **Dependencies** — SBT coordinates (`groupId %% artifactId` or `groupId %
   artifactId`) without versions, each with a short explanation.
3. **Content** — `##` sections with prose and code examples.

## Reference project

All code examples are based on
[Bootzooka](https://github.com/softwaremill/bootzooka), a production-ready
starter project using Tapir, Ox, sttp, and PostgreSQL. Clone it to study the
patterns before writing.

Additional documentation sources:
- [Tapir](https://tapir.softwaremill.com)
- [Ox](https://ox.softwaremill.com)
- [sttp client](https://sttp.softwaremill.com)

## Instructions for adding a new chapter

1. **Clone Bootzooka** to a temporary directory and explore the relevant source
   files. Read the actual code — don't guess at APIs or patterns.

2. **Identify the use-case.** Each chapter is a single, focused topic (e.g.,
   "authentication", "testing HTTP endpoints", "observability"). Don't combine
   multiple unrelated concerns.

3. **Create the chapter file** as `NN-slug.md` where `NN` is the next number and
   `slug` is a short kebab-case name.

4. **Write the chapter** following the format:
   - Start with a `#` title and a description paragraph.
   - List dependencies as SBT coordinates without versions.
   - Write `##` sections. Each section should explain one concept, then show the
     code from Bootzooka that implements it. Show real code from the project,
     not invented examples.
   - Explain Scala 3 / direct-style specifics where they appear naturally
     (context functions, `either` blocks, `supervised` scopes, virtual threads).
     Don't force explanations of features that aren't relevant to the section.

5. **Update `index.md`** — add the new chapter to the list with a description.

### Writing guidelines

- **Use real code.** Every code block should come from Bootzooka (or be a
  minimal adaptation of it). If the project doesn't have an example of what you
  need, say so rather than inventing one.
- **Explain the why, not just the what.** Don't just show code — explain the
  design decision behind it. Why is `Auth` generic over `T`? Why is the tracing
  interceptor prepended? Why use `transactEither` instead of `transact`?
- **Keep it direct.** No filler, no motivational paragraphs about why
  observability matters. The reader chose this chapter because they want to
  implement the thing.
- **Dependencies are library coordinates only.** No version numbers — they go
  stale. The reader will use the versions from Bootzooka's `build.sbt` or the
  library's latest release.
- **Don't duplicate content across chapters.** If authentication is covered in
  chapter 1, chapter 2 can reference it rather than re-explaining the `Fail` ADT
  or `secureEndpoint`.
