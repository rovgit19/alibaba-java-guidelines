---
name: alibaba-java-guidelines
description: >
  Alibaba Java Coding Guidelines (阿里巴巴Java开发手册/P3C) for writing production-quality Java code.
  Use this skill whenever writing, reviewing, or refactoring Java code — including Spring Boot services,
  MyBatis/MyBatis-Plus mappers, POJO/DTO/VO definitions, MySQL schema design, REST APIs, or any Java
  backend work. Also trigger when the user asks about Java coding standards, naming conventions, 
  concurrency best practices, collection pitfalls, exception handling patterns, SQL/ORM rules, or 
  mentions "P3C", "阿里规约", "阿里巴巴开发手册", or "coding guidelines". Even if the user doesn't 
  explicitly ask for standards compliance, apply these rules when generating or reviewing Java code.
---

# Alibaba Java Coding Guidelines Skill

## Purpose

Enforce Alibaba Java Coding Guidelines (P3C, 黄山版) when writing or reviewing Java code.
These guidelines cover: naming, formatting, OOP, collections, concurrency, flow control,
comments, exceptions, logging, MySQL schema/index/SQL/ORM, project structure, and security.

## When to Apply

Apply these guidelines **automatically** whenever you:
- Write new Java classes, methods, or SQL
- Review or refactor existing Java code
- Design MySQL table schemas or indexes
- Define POJOs (DO/DTO/VO/BO)
- Configure thread pools, locks, or concurrent data structures
- Write MyBatis XML mappings or ORM code

## Quick Reference — Most Critical Rules

Before reading the full reference, internalize these high-impact rules:

### Naming
- Class: `UpperCamelCase` (except DO/DTO/VO). Method/var: `lowerCamelCase`. Constant: `UPPER_SNAKE_CASE`
- Boolean fields: **no** `is` prefix (`success` not `isSuccess`)
- Service/DAO method prefixes: `get`/`list`/`count`/`save`/`remove`/`update`
- Domain models: `*DO`, `*DTO`, `*VO`, `*BO`. Never `*POJO`

### OOP & Collections
- Always `@Override`. Use `"constant".equals(obj)` or `Objects.equals()`
- Wrapper classes for POJO members and RPC params. Primitives for local vars
- No `Executors.newXxx()` — use `new ThreadPoolExecutor(...)` directly
- `ThreadLocal.remove()` in finally. `SimpleDateFormat` is not thread-safe
- Don't modify collections in foreach — use `Iterator`

### MySQL
- Boolean → `is_xxx` unsigned tinyint. PK → `id` bigint unsigned auto_increment
- Every table: `gmt_create` + `gmt_modified` datetime columns
- No `SELECT *`. Use `#{}` not `${}` in MyBatis. No join on 3+ tables
- `count(*)` is correct, not `count(column_name)`

### Exception & Logging
- Pre-check > catch RuntimeException. Try-with-resources for closeable
- SLF4J facade only. Placeholder `{}` for log params. Never string concat in log
- `logger.error(context + "_" + e.getMessage(), e)` — always include stack trace

### Modern Java (17+)
- Prefer `record` for DTOs/VOs (immutable, auto toString/equals)
- Return `Optional` from query methods. Never use as field or parameter
- Streams: keep pipelines ≤3-4 ops. Use `toList()` (Java 16+). Complex logic → use loop
- Bean Validation (`@Valid` + `@NotNull`/`@NotBlank`) on Controller inputs
- Domain exception hierarchy + `@RestControllerAdvice` global handler
- Testing: JUnit 5 + AssertJ + Mockito. Name: `method_scenario_expected`
- Structured logging: `key=value` format + MDC for traceId/userId

## Full Reference

**Core rules (P3C 阿里巴巴Java开发手册):**
→ `references/guidelines.md`

**Modern Java supplement (Record/Optional/Stream/Testing/Exception design):**
→ `references/modern-java.md`

## How to Use This Skill

1. **When writing code**: Apply rules from the Quick Reference above by default.
   Consult `references/guidelines.md` for specific sections when uncertain.

2. **When reviewing code**: Check against [Mandatory] rules first. Flag violations
   with the rule reference (e.g. "Naming Rule #8: Boolean fields should not use `is` prefix").

3. **When the user asks about standards**: Cite specific rules from the guidelines
   with examples. Mention severity level ([Mandatory] vs [Recommended] vs [For Reference]).

4. **Priority**: [Mandatory] rules must always be followed. [Recommended] rules should
   be followed unless there's a specific reason not to. [For Reference] rules are suggestions.
