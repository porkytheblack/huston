# Huston — Project Manifesto

## The Problem

**Opencode** is a solid coding agent, but it has a critical limitation: its dependence on the **Vercel AI SDK** makes it difficult to use with local models. If you want a local setup, you're out of luck.

Opencode is also narrowly scoped — it's built only for coding. A specialized agent, nothing more.

## The Solution

**Glove** ([glove.dterminal.net](https://glove.dterminal.net)) is a framework for building agents — built to avoid the constraints that plague tools like Opencode.

**Huston** is a multi-agent system built on Glove. It's designed to:

1. **Start with coding** — achieve feature parity with Opencode, but without the Vercel AI SDK dependency. Local models work out of the box.
2. **Grow beyond coding** — support specialized agents for different use cases, all under one system.

## Priorities (Phase 1)

- Build a coding agent on Glove with full Opencode feature parity
- First-class support for local models
- Clean architecture that allows adding new specialized agents later

## Vision

Huston isn't just another coding agent. It's a platform for composing specialized agents — coding is just the first one.