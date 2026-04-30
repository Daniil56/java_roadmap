# Roadmap: Java Backend Developer v2.0

## Your Path to Java Developer in 4-6 Months

**Commitment:** 30-40 hours per week
**Author:** [daniil_it_engineer](https://t.me/daniil_it_engineer)

**Language:** [English](README.md) | [Русский](../README.MD)

---

## Table of Contents

- [Introduction](#introduction)
- [How to Read This Roadmap](#how-to-read-this-roadmap)
- [How the Path Is Structured](#how-the-path-is-structured)
- [How to Learn with AI and AI Tooling](#how-to-learn-with-ai-and-ai-tooling)
- [How to Work with Requirements](#how-to-work-with-requirements)
- [How to Estimate Time](#how-to-estimate-time)
- [Sources and Tools](#sources-and-tools)
- [Modules](#modules)
- [Conclusion](#conclusion)
- [Contacts](#contacts)

---

## Introduction

This roadmap shows the path from your first steps in Java to the level where you can confidently pursue backend development and keep growing.

It is for:

- those starting from scratch;
- those who already have the basics but lack a systematic approach;
- those who have studied but are stuck between "I know something" and "I can build a real project";
- those entering the market in tougher conditions and want to learn following a clear route rather than random topics.

The previous version of the roadmap is still relevant in terms of the core technology stack and can be used as reference material: [version 1.1](../version/v.1.1/README.MD). Its limitation is not an "outdated stack" but a narrower scope: less attention to requirements, system runtime behavior, observability, AI tooling, and the engineering context of decision-making. Version 2.0 does not replace it as "the only right way" — it expands and deepens the route.

Version 2.0 preserves the goal of the first version: to help you get into Java development through a clear sequence of topics and practice. But the market has changed, and knowing Java syntax and a couple of Spring annotations is no longer enough. A backend developer is now expected to:

- **build and verify projects:** Gradle, tests, CI, coverage, style checks;
- **work with data and infrastructure:** PostgreSQL, migrations, Docker, Kubernetes, configuration;
- **design APIs and services:** REST, OpenAPI, microservices along domain boundaries, distributed operations;
- **ensure reliability:** metrics, traces, logs, SLI/SLO, Circuit Breaker, idempotency;
- **design safe concurrency:** maintain request/message context isolation, safely parallelize independent I/O scenarios, and account for runtime constraints, connection pools, and external services;
- **use AI as a tool:** AI is not a replacement for the developer but an accelerator. The ability to set a task for an AI agent, verify the result, describe project rules (CLAUDE.md, AGENTS.md, MCP), and take responsibility for the code is an expected competence on the market in 2026.

And not only that. A developer who understands the value of what they are building is more in demand than one who simply writes code to a specification. Business context is not an abstraction — it manifests concretely in how system requirements are formulated: functional requirements grow from business needs ("the system registers a participant" is not CRUD, but an answer to "how does the organizer manage registrations without manual tracking?"), non-functional requirements grow from business expectations ("p95 latency < 200ms" is not a metric for its own sake but "the user does not wait too long"; test coverage is not about the percentage but ensuring registration does not break after refactoring).

The more business context a developer brings into decisions — why this data structure was chosen, why this index, why a Saga here instead of a synchronous call — the higher their market value. This roadmap teaches you to connect technical decisions with business questions from the very first module.

---

## How to Read This Roadmap

The roadmap is divided into modules. Each module answers three questions:

1. What you need to learn.
2. What should appear in the project after this step.
3. How to understand that the module is truly completed.

The modules evolve **one project** from simple to complex — this is the recommended path. But each module can also be completed **independently**, applying its requirements to your own project or independently reproducing the results of previous modules. Each module description has a "Who is this for" section that clarifies what you need to know before starting.

It is best to go through the roadmap in order. If time is limited, some things can be studied in parallel, but the main technical line should not be broken: Java basics → application → database and transactions → Spring and runtime HTTP requests → microservices and fan-out → observability → events and Kafka → career track.

---

## How the Path Is Structured

The roadmap is built around a single project that evolves from simple to complex.

```text
Console application + Java Core + Gradle + local checks + Git and PR workflow
→ more mature Java logic + GitHub Actions + JaCoCo + recommended AI tools
→ containerization and local launch in Docker
→ PostgreSQL + JDBC + transactions and concurrent registrations
→ REST API on Spring Boot + request-context and safe shared state
→ microservice architecture + parallel calls to independent services
→ Kubernetes and observability + trace/metrics for fan-out scenarios
→ Kafka + saga/outbox + consumer parallelism, message-local context and DLQ
→ job search preparation
```

This format is closer to real work. You do not start a new learning project each time — you improve an existing one: first you build a console application, master Java and basic tools, and then gradually add more mature quality practices, AI tools, and the next technical layers.

---

## How to Learn with AI and AI Tooling

AI is needed not to replace the developer but to accelerate development and learning.

The core principle: **AI does not replace responsibility.** AI is not accountable to anyone. Only the developer is accountable.

How to use AI throughout the roadmap:

- **Module 1:** a chat (DeepSeek or Qwen Chat), the official guide, and basic IDE work. This is enough to work through Java Core, build errors, Git, Gradle, and your first tests.
- **Starting from Module 2:** you can connect more advanced AI tools: a CLI agent (Qwen Code, OpenCode, or Kilo), Continue, and MCP, where you need to work not only with a question but with the codebase, documentation, and configurations. This is a recommended layer, not a mandatory barrier for completing the roadmap.

Basic rules:

- **For chat:** ask questions with context, ask for an explanation of the reason, and do not copy the answer into the project without verification.
- **For an agent (AI Pair Programming):** starting from Module 2, the CLI agent becomes a pair programmer — it generates tests, refactors code, helps with boilerplate. The person verifies the result through coverage metrics, tests, and the build. If the generated code is unclear — stop and ask the agent to describe what it is doing. This is a concept from Extreme Programming: two developers working on one task, one leads, the other helps.
- If the question depends on a library version, plugin, configuration, or external service, the answer should be verified through MCP or the official source.
- The result is always verified through code, build, tests, launch, and logs.

---

## How to Work with Requirements

Each module has two layers of outcomes.

**Functional requirements** answer the question: what should the system be able to do.
**Non-functional requirements** answer the question: how well and reliably it should work.

A simple example:

- functionally, the system must register a user or create an event;
- non-functionally, the project must build, pass tests, not break on style, and be verifiable.

Therefore, a module cannot be considered complete just because "the feature seems to work." If the code does not pass verification, tests poorly, or does not reproduce on another machine, the work is not done.

---

## How to Estimate Time

Task estimation is part of a developer's job, not a skill for "someday later." On one project, tasks may already be estimated; on another, estimation is delegated to you. In any case, it is important to understand how the estimate was derived, to be able to adjust it, and to ask questions if something does not add up. This roadmap provides a training ground — I recommend starting to estimate tasks from the first module, even if the estimates are rough at first.

To avoid getting lost, estimate the module not as a whole but in parts:

- analyzing requirements and business context: what exactly needs to be done and why;
- studying the topic;
- setting up the environment;
- writing code;
- debugging;
- verifying the result.

A normal estimation pattern:

```text
module time = requirements analysis + reading + practice + fixes + verification + 30% buffer
```

In short:

- start estimating from the first module — even a rough estimate is better than no estimate;
- after the first real step, adjust the estimate;
- do not plan the module "with no room to spare";
- always leave time for errors, returning to documentation, and rework.

---

## Sources and Tools

Below are the basic external sources and tools useful throughout the roadmap.

### For Task Estimation and Planning

To make estimation not a "guess," it is convenient to separate two things:

- **estimation method**: how to understand the size of a task;
- **tracking tool**: where to store the plan, notes, and deadlines.

#### What to Read About Estimation

- [Habr: Task Estimation in Story Points](https://habr.com/ru/articles/489500/)
  A good Russian-language introduction to story points, relative estimation, and the connection between estimation and actual team work.

- [Habr: Planning Poker](https://habr.com/ru/articles/748180/)
  A short and clear explanation of planning poker if you want to understand the mechanics of collective estimation.

- [Atlassian: Story points and estimation](https://www.atlassian.com/agile/project-management/estimation)
  The official guide on relative estimation, planning poker, comparing tasks, and revising past estimates.

- [Atlassian Support: Estimate a work item](https://support.atlassian.com/jira-software-cloud/docs/estimate-an-issue/)
  Useful if you want to understand how estimates live in a real task tracker.

- [Atlassian Support: Time estimates](https://support.atlassian.com/jira-software-cloud/docs/what-are-time-estimates-days-hours-minutes/)
  A good reference when it is more convenient to estimate in time rather than relative units.

#### Where to Track Estimates and Plans

| Tool | When to Use | Why It Works |
|---|---|---|
| [Obsidian](https://obsidian.md/) | If you want a local note-taking system | Great for module notes, task estimation, progress journal, all stored locally in Markdown |
| [Obsidian Help](https://help.obsidian.md/) | If you are starting from scratch in Obsidian | Official documentation on vaults, notes, links, and basic organization |
| [Notion](https://www.notion.so/) | If you need a web-based tool | Suitable for tables, notes, calendars, lists, and team collaboration |
| [Google Calendar: Tasks](https://support.google.com/calendar/answer/9901136?co=GENIE.Platform%3DDesktop) | If you need to distribute tasks across time slots | Convenient for setting real calendar slots instead of keeping estimates only in your head |

### Methodologies and Practices

Starting from Module 2, the roadmap relies on the practice of **pair programming** from Extreme Programming (XP): two developers work together on one task, one leads (the human), the other helps (AI). The CLI agent becomes a pair programmer — generating tests, refactoring code, helping with boilerplate, while the human verifies the result through coverage metrics, tests, and the build.

- [Kent Beck: Extreme Programming Explained](https://www.oreilly.com/library/view/extreme-programming-explained/0201616416/) — the classic book describing pair programming and other XP practices (TDD, continuous integration, small releases)
- [Martin Fowler: Pair Programming](https://martinfowler.com/articles/onPairProgramming.html) — an overview of the pair programming practice

### AI Tools and Plugins

**Core (free, widely available):**

- [DeepSeek](https://chat.deepseek.com/) — universal chat for analyzing topics, errors, and quick questions. Free, works without VPN.
- [Qwen Code](https://github.com/QwenLM/qwen-code) — open-source AI agent for the terminal. The most powerful free CLI: Qwen model support, Skills, SubAgents, generous free tier. Integration with VS Code, Zed, and JetBrains. [Documentation](https://qwenlm.github.io/qwen-code-docs/).
- [OpenCode](https://opencode.ai/) — open-source agent for the terminal, IDE, and desktop. Free, 75+ models.
- [Kilo](https://kilo.ai/) — open-source AI coding agent for VS Code, JetBrains, and CLI.
- [Continue](https://www.continue.dev/) — AI assistant and agent platform for IDE, terminal, and CI. [Documentation](https://docs.continue.dev/).
- [Qwen Chat](https://chat.qwen.ai/) — chat alternative for quick explanations and code analysis.

**Paid (optional):**

- [ChatGPT](https://chatgpt.com/) — paid chat with broad capabilities. [Codex](https://openai.com/codex/) — agent tool for working with code.
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) — AI agent for the terminal with access to the codebase, files, and shell.
- [GLM-5 by Z.AI](https://docs.z.ai/guides/llm/glm-5) — a model for agentic engineering and long agent tasks.

If you need the simplest setup, this combination is enough: **Obsidian for notes + Google Calendar for time**. If you need a visual web format like Notion, you can use **Notion**.

You can also use **Anki** separately for terms and concepts that you regularly encounter and forget.

### Voice to Text

I regularly dictate notes by voice — task estimates, lines of thought, architecture ideas. I use voice-to-text both for communicating with AI chats and for populating my knowledge base. It is faster than typing, and most importantly — you see the text immediately. Unlike voice messages, there is no need to re-listen: read it, correct it, save it. The entire thought process is in front of you, and you can return to it and recall why you made a particular decision.

| Tool | Platform | What It Does |
|---|---|---|
| [Wispr Flow](https://wisprflow.ai/r?DANIIL290/) | macOS, Windows | Dictation into any application, 100+ languages including Russian, automatic punctuation. Paid, but the most convenient I have tried |
| [MacWhisper](https://macwhisper.io/) | macOS | Transcription based on OpenAI Whisper, works offline, Russian supported. Free version with base model |
| [Whisper Notes](https://whispernotes.app/) | iOS, macOS | Offline transcription via Whisper, no internet or subscription required. Free on Mac |
| [Google Docs Voice Typing](https://support.google.com/docs/answer/4492226) | Browser (Chrome) | Free voice input directly in Google Docs, Russian supported |
| Built-in Dictation | macOS, Windows, iOS, Android | System feature — press a keyboard shortcut and dictate into any input field. Lower quality than specialized tools, but sufficient for quick notes |

---

## Modules

| # | Module | Key Topics | Why It Matters |
|---|---|---|---|
| 1 | [Java: Basics and Tools](../modules/module-1-java-core-base.md) | Java Core, Gradle, Git, basic tests, Checkstyle, console I/O, chat and guide | Establish a working foundation: write code, build the project, run it, and don't get overwhelmed by tools at the start |
| 2 | [Java Core: Advanced](../modules/module-2-java-core-advanced.md) | OOP, collections, Stream API, `HashMap`, exceptions, GitHub Actions, JaCoCo, recommended AI tools | Make the code more mature and connect a more advanced workflow: CI, coverage, and mindful work with documentation |
| 3 | [Containerization (Docker)](../modules/module-3-docker.md) | Docker, Dockerfile, `docker pull`, `docker run`, `docker exec`, Docker Compose, volumes | Learn to run the project reproducibly, understand the container foundation, and set up a local environment for the next step |
| 4 | [Database](../modules/module-4-database.md) | PostgreSQL, SQL, JDBC, Liquibase, Testcontainers, transactions, concurrent registrations | Move the project from in-memory to real data storage and understand how the database maintains invariants under concurrent writes |
| 5 | [Spring Ecosystem](../modules/module-5-spring.md) | Spring Boot, Spring Web, REST API, JPA, `@Transactional`, MockMvc, OpenAPI, request-context, shared state | Turn the project into a Spring backend service and maintain correctness of runtime HTTP requests |
| 6 | [Microservice Architecture](../modules/module-6-microservices.md) | DDD bounded contexts, API Gateway, Database per Service, Feign, Resilience4j, Outbox, WireMock, Vault, fan-out | Split the monolith into microservices along domain boundaries and learn to safely parallelize independent downstream calls |
| 7 | [Kubernetes and Observability](../modules/module-7-k8s-observability.md) | minikube, Helm umbrella chart, Health Probes, Prometheus + Grafana, Jaeger, SLI/SLO/SLA, fan-out observability, K8s RBAC, HPA | Move the microservice stack into Kubernetes and make fan-out scenarios observable through traces, metrics, and diagnostics |
| 8 | [Events and Kafka](../modules/module-8-kafka.md) | Kafka, producer/consumer, Saga Pattern (choreography), Transactional Outbox (Debezium CDC), Kafka Streams, idempotent consumer, consumer parallelism, message-local context, DLQ | Add an async pipeline and learn to manage message processing concurrency without violating ordering and idempotency |
| 9 | [Career Track](../modules/module-9-career-track.md) | CV and achievements, engineering case studies, code review, AI-assisted workflow, project presentation, systems thinking | Package your experience into a resume, learn to present your project, and prepare for interviews |

---

## Conclusion

If you complete the roadmap in full, you will not simply "study topics." You will have a project that you have carried hands-on from a console jar file to microservices with Kafka — through Gradle and tests, Docker and PostgreSQL, Spring Boot and REST API, Kubernetes and Prometheus, events and outbox. Every step is not an abstract exercise but an answer to a specific question: why a transaction here, why this index, what problem the Saga solves, how Circuit Breaker helps.

I built this roadmap so that technical decisions would not hang in the air. A Saga is needed not because it is a "microservices pattern" but because in a distributed operation you cannot leave the system in a half-broken state. Circuit Breaker is not for a checkbox in Resilience4j but so that a failure in one service does not drag down the entire user scenario. An idempotent consumer is not for "proper Kafka" but so that a redelivered event does not create a duplicate registration.

Beyond code and architecture, you will gain:

- **an engineering foundation:** build, CI, migrations, containerization, observability — what distinguishes a project from "it works on my machine";
- **experience working with AI:** from a chat for debugging errors to a CLI agent as a pair programmer — AI does not replace responsibility but accelerates routine work;
- **task estimation skills:** from the first module through the career track.

This is a foundation that lets you confidently go into interviews and keep growing.

---

## Contacts

- Telegram channel: [@mentee_power](https://t.me/mentee_power)
- Personal Telegram: [@daniil_it_engineer](https://t.me/daniil_it_engineer)
