# Direct Style Scala 3: A Practical Guide

A book written by AI, for AI and humans alike. Use-case driven guides to writing
direct-style Scala 3 applications, based on virtual threads (Java 21+).

## Usage in prompts

Copy the following snippet into your AI agent's prompt or `CLAUDE.md` to give it
access to the guide:

```
BEFORE writing any code that uses Tapir, Ox, sttp, or Kafka, you MUST fetch the
guide index and the relevant chapter(s), then follow the patterns shown there:
https://raw.githubusercontent.com/VirtusLab/direct-style-guide/refs/heads/master/index.md

The index lists self-contained chapters by use-case (error handling,
authentication, testing, observability, persistence, configuration, etc.). Fetch
the chapter relevant to your current task for implementation patterns and code
examples. This is not optional — code that ignores guide patterns will be
rejected in review.

Base URL for chapters:
https://raw.githubusercontent.com/VirtusLab/direct-style-guide/refs/heads/master/
```

## Structure

- `index.md` — lists all chapters with a short description of each. This is the
  entry point.
- One markdown file per chapter, numbered by group (`100-*.md`, `110-*.md`,
  `200-*.md`, etc.). Groups start at 100, 200, 300, etc.; chapters within a
  group increment by 10.

Each chapter covers a single use-case and follows this format:

1. **Title** — the use-case name.
2. **Dependencies** — SBT coordinates (`groupId %% artifactId` or `groupId %
   artifactId`) without versions, each with a short explanation.
3. **Content** — `##` sections with prose and code examples.

## Sources

Chapters are based on patterns from reference projects and library
documentation. Clone/read the relevant sources before writing a chapter.

Reference projects:
- [Bootzooka](https://github.com/softwaremill/bootzooka) — production-ready
  starter using Tapir, Ox, sttp, and PostgreSQL

Library documentation:
- [Tapir](https://tapir.softwaremill.com)
- [Ox](https://ox.softwaremill.com)
- [sttp client](https://sttp.softwaremill.com)

## Instructions for adding a new chapter

1. **Read the sources.** Clone the relevant reference project and/or read the
   library documentation. Don't guess at APIs or patterns.

2. **Identify the use-case.** Each chapter is a single, focused topic (e.g.,
   "authentication", "testing HTTP endpoints", "observability"). Don't combine
   multiple unrelated concerns.

3. **Create the chapter file** as `NNN-slug.md` where `NNN` is the next number
   in the appropriate group (see Structure above) and `slug` is a short
   kebab-case name.

4. **Write the chapter** following the format:
   - Start with a `#` title and a description paragraph.
   - List dependencies as SBT coordinates without versions.
   - Write `##` sections. Each section should explain one concept with code
     examples based on the reference projects or library documentation.
   - Explain Scala 3 / direct-style specifics where they appear naturally
     (context functions, `either` blocks, `supervised` scopes, virtual threads).
     Don't force explanations of features that aren't relevant to the section.

5. **Update `index.md`** — add the new chapter to the list with a description.

### Writing guidelines

- **Base code on real sources.** Code examples should be drawn from the
  reference projects or library documentation, adapted to show only the
  relevant pattern.
- **Explain the why, not just the what.** Don't just show code — explain the
  design decision behind it. Why is `Auth` generic over `T`? Why is the tracing
  interceptor prepended? Why use `transactEither` instead of `transact`?
- **Keep it direct.** No filler, no motivational paragraphs about why
  observability matters. The reader chose this chapter because they want to
  implement the thing.
- **Dependencies are library coordinates only.** No version numbers — they go
  stale. The reader will use the library's latest release.
- **Show only the essence.** Each code example should illustrate exactly the
  pattern being discussed. Strip out unrelated features, domain details, and
  infrastructure that blur the point — the reader can find the full version in
  the reference project.
- **Don't duplicate content across chapters.** If authentication is covered in
  chapter 1, chapter 2 can reference it rather than re-explaining the `Fail` ADT
  or `secureEndpoint`.
