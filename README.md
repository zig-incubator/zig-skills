# Zig Agent Skill

An [Agent Skill](https://agentskills.io/) for the Zig programming language, currently targeting version 0.16.0.

## Why This Exists

LLMs struggle with Zig more than languages like TypeScript or Swift. The language is still evolving, there's less training data available, and models frequently use deprecated or removed syntax. This skill provides up-to-date context to guide models toward correct, idiomatic Zig code.

## What's Included

- **SKILL.md**: Core patterns and critical breaking changes for Zig 0.16.0
- **references/**: 50+ files of comprehensive documentation
  - Standard library module documentation (std.Io, std.heap, std.fs, etc.)
  - Idiomatic patterns extracted from the Zig standard library
  - Build system guidance (build.zig, modules, dependencies)
  - Language reference content (comptime, builtins, error handling)
  - Code review checklist with migration examples

## How It Was Made

Generated using Claude Opus 4.5 with the Zig standard library source code as context. Each module was documented by feeding the model the actual source files and using the [skill-creator](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md) to structure the output.

The methodology prioritizes grounding the model in real source code rather than hallucinated documentation.

**Updated for 0.16.0**: The documentation has been updated to reflect major breaking changes including:
- Complete rewrite of `std.Io` interface with async/await support
- Thread-safe ArenaAllocator (ThreadSafe wrapper removed)
- Removed types: GenericReader, AnyReader, FixedBufferStream, Thread.Pool
- New concurrent operations via `std.Io.concurrent()`

See https://austinrude.com/blog/making-a-zig-agent-skill/ for details.

## Usage

### Install as a Plugin

After Claude Code is installed, you can install this skill as a plugin:

```bash
/plugin marketplace add zig-incubator/zig-skills
/plugin install zig-skills@zig-skills
```

### Manual Installation

Alternatively, install the `zig` directory into `~/.claude/skills/` per [Claude Code skill documentation](https://docs.anthropic.com/en/docs/claude-code/skills).

### Example Prompts

```
Using the zig skill, review @src/main.zig for idiomatic patterns
```

```
Help me set up a build.zig for a project with multiple modules
```

```
Explain how to use the new std.Io async interface in 0.16.0
```

```
Review my allocator usage and suggest improvements
```

## What This Skill Helps With

- Avoiding deprecated syntax and removed features from older Zig versions
- Writing idiomatic Zig code following standard library patterns
- Understanding the new `std.Io` interface and async/await patterns
- Proper allocator selection and naming conventions
- Build system configuration with modules and dependencies
- Error handling best practices (try/catch/errdefer)
- Comptime metaprogramming and type reflection

## Feedback

Please [open an issue](https://github.com/zig-incubator/zig-skills/issues) if you find any hallucinations, inaccuracies, or outdated patterns. No need to write a detailed report with a fix—just knowing about the problem helps improve the skill.

## Attribution

This skill is a fork of the original [zig-skill](https://github.com/rudedogg/zig-skill) by [Austin Rude](https://github.com/rudedogg). The initial documentation was generated using his methodology with the Zig standard library source code as context.

This fork is maintained independently and has been updated for Zig 0.16.0.

## Specification

This skill follows the [Agent Skills Specification](https://agentskills.io/).

## License

MIT
