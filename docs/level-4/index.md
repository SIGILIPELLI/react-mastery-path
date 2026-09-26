---
description: "Level 4 of React Mastery Path: Server Components, concurrency, profiling, design systems, large-app architecture, security, testing, deployment and i18n."
---

# Level 4 · Master <span class="level-badge">Architecture</span>

Goal: make the decisions a senior or lead React engineer makes. At this level the
questions are less "how do I use this hook?" and more "where should this code run?",
"why is this interaction slow and how do I prove it?", "how do twelve people work in
this codebase without stepping on each other?", and "what could an attacker do with
this component?".

## Modules

1. [Server Components & Frameworks](01-server-components.md) — the server/client split, `'use client'`, Server Functions, and choosing a framework
2. [Concurrent Rendering](02-concurrent-rendering.md) — `useTransition`, `useDeferredValue`, and how interruptible rendering works
3. [Performance Profiling](03-performance-profiling.md) — Core Web Vitals, the React Profiler, the Performance panel, and a repeatable method
4. [Design Systems & Component Libraries](04-design-systems.md) — tokens, headless primitives, variants, and documentation
5. [Architecture for Large Apps](05-architecture.md) — feature-based structure, boundaries, and dependency rules
6. [Security in React Apps](06-security.md) — XSS, `dangerouslySetInnerHTML`, URLs, tokens, and supply chain
7. [Testing Strategy](07-testing-strategy.md) — what to test at which level, end-to-end tests, and keeping suites fast and trustworthy
8. [Building & Deploying](08-deployment.md) — static hosting, environment variables, caching, and CI
9. [Internationalization](09-internationalization.md) — translations, plurals, `Intl` formatting and right-to-left layouts
10. [Capstone — Team Dashboard](10-capstone.md) — a production-grade app assembled from every level

## Before you start

You should be fluent with everything in Levels 1–3. Several topics here depend on tools
outside React itself — a meta-framework for Server Components, a hosting provider for
deployment, a monitoring service for real-user metrics. Where something can't be fully
run on a laptop alone, the lesson says so and shows what you *can* practise locally.

React's server-side features have been evolving quickly. This level teaches the
concepts that hold across frameworks and points you to each framework's documentation
for exact file conventions and configuration.
