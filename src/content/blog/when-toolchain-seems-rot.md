---
title: "When Toolchain Seams Rot: Rebuilding the CubeMX → PlatformIO Boundary"
description: "How small upstream changes quietly broke a once-viable STM32 workflow, and why generic toolchains decay without explicit bridges."
pubDate: 2025-12-22
heroImage: "../../assets/stm32bridge_image.jpg"
---

Toolchains rarely fail all at once.

They decay at the seams, the places where responsibility quietly shifts from “supported” to “assumed”. When those seams are implicit, undocumented, or owned by no one in particular, the workflow doesn’t explode. It just becomes fragile, manual, and increasingly expensive to maintain.

This post is about one such seam in the STM32 ecosystem. The lesson, however, is not specific to embedded systems.

## The original boundary

For a long time, working with STM32 devices meant accepting a split workflow:

- **STM32CubeMX** handled configuration: pins, clocks, HAL setup, middleware, and (optionally) FreeRTOS.
- **The development environment** handled everything else: building, debugging, editing, and iteration.

That division of labor made sense. CubeMX is very good at generating correct initialization code. Tools like PlatformIO are very good at managing build systems, dependencies, and modern editor workflows.

The seam between the two was awkward, but real, and importantly, stable enough to build workflows around.

## The two supported paths

Historically, “working with STM32 and PlatformIO” meant choosing between two distinct approaches.

One option was to stay entirely within a higher-level framework, most commonly **Arduino**. In that world, PlatformIO works well: you select an Arduino-compatible STM32 board, write firmware against the Arduino APIs, and avoid CubeMX altogether. For many projects, this is a perfectly valid and well-supported choice.

The other option was to use **STM32CubeMX (or CubeIDE)** for what it does best: configuring clocks, pins, HAL, and middleware; and then migrate the generated output into PlatformIO to regain a modern development workflow.

The seam this post focuses on exists only in the second case.

Arduino-based workflows never depended on it. But they also come with different constraints: limited access to low-level configuration, less predictable HAL integration, and friction once you move beyond the abstractions Arduino provides.

For teams and projects that needed CubeMX-level control *and* PlatformIO-level ergonomics, that seam mattered.

## What broke (quietly)

Nothing dramatic happened. No loud deprecations. No explicit announcement that the workflow was no longer viable.

Instead, two small changes accumulated:

- STM32CubeMX removed the “Other Toolchains (GPDSC)” export option.
- CubeMX launched from within STM32CubeIDE often locks the Toolchain/IDE selector to CubeIDE-only generation.

Each change is locally reasonable. Together, they invalidate an entire class of workflows.

If you rely on CubeMX for configuration and PlatformIO for development, you’re pushed into one of three options:

- manually copying files after every regeneration,
- maintaining parallel projects and keeping them in sync,
- or slowly accepting that the workflow is brittle and error-prone.

No one explicitly broke the seam, it simply stopped being maintained.

## The real problem isn’t export formats

It’s tempting to frame this as “GPDSC was removed” or “the Toolchain selector is locked”.

Those are symptoms, not the cause.

The deeper issue is **regeneration combined with structural mismatch**.

CubeMX will regenerate code. That’s unavoidable.
PlatformIO expects a different project structure. That’s also reasonable.

When the mapping between those structures is implicit and manual, you end up with:

- include paths that drift,
- build filters that accrete exceptions,
- middleware versions that quietly diverge,
- and a workflow that breaks every time the `.ioc` changes.

The cost isn’t just time, it’s confidence. You stop trusting regeneration. You hesitate to change configuration. You avoid necessary evolution.

At that point, the seam has already failed.

## A general pattern: bridges, not glue

When a boundary between tools stops being supported, the only sustainable response is to **formalize it**.

Not with ad-hoc scripts.
Not with copy-paste checklists.
But with something that encodes assumptions *once* and makes them explicit.

A bridge does three things:

- it defines what flows across the boundary,
- it makes regeneration safe and repeatable,
- and it turns tribal knowledge into a deterministic step.

This pattern shows up everywhere: ingestion pipelines, schema evolution, deployment tooling. Toolchains are no different.

## stm32bridge as an instance of the pattern

`stm32bridge` is one concrete implementation of this idea for the CubeMX → PlatformIO seam.

It doesn’t try to replace either tool. It accepts their roles as they are, and formalizes the boundary between them.

At a high level, it:

- normalizes CubeMX output into a structure PlatformIO can build,
- generates a correct `platformio.ini` (flags, include paths, filters),
- preserves the `.ioc` so regeneration remains possible,
- avoids GPDSC entirely,
- and optionally generates custom PlatformIO board definitions when needed.

The goal isn’t automation for its own sake.
The goal is to make the workflow *boring* again.

## A minimal demonstration

Given a CubeMX-generated project, you can inspect what actually matters:

```bash
stm32bridge analyze ./example_project
```

Then migrate it into a fresh PlatformIO project and verify the build:

```bash
stm32bridge migrate ./example_project ./migrated_project \
  --board generic_stm32l433 \
  --board-file ./generic_stm32l433.json \
  --build \
  --disable-freertos
```

If the build succeeds, the seam is restored. You can iterate in PlatformIO as if the project had been created there from day one, while still treating CubeMX as the source of truth for configuration.

That’s the entire point.

## Trade-offs and limits

This isn’t magic, and it’s not trying to be.

Some constraints are intentional:

- PlatformIO uses its framework HAL drivers, not whatever CubeMX copied into the project. That avoids drift, but custom HAL modifications need to be handled consciously.
- CubeMX regeneration is preserved, but only if you treat CubeMX as a configuration tool, not as the place where application logic lives.
- Board auto-detection isn’t always possible. When it fails, that’s a signal that the metadata needs to be made explicit.

Bridges don’t eliminate decisions. They surface them.

## The broader lesson

Generic workflows don’t usually fail loudly.

They decay quietly: one broken assumption at a time, until manual work becomes normalized and no one remembers why the friction exists.

At that point, the choice is simple: accept the decay, or rebuild the bridge explicitly.

Toolchains are no exception.

>If this maps to a problem you’re dealing with, feel free to reach out at me@jobenas.com. I’m more interested in discussing real systems than debating abstractions.