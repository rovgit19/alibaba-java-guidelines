# Alibaba Java Coding Guidelines Skill (P3C)

阿里巴巴Java开发手册（黄山版）+ Modern Java 17+ 补充规范。

适用于 Claude Code / Claude.ai / Gemini CLI / Codex CLI / Cursor 等 AI coding agent。

## Install

### 通过 npx skills CLI（推荐，支持所有 agent）

```bash
npx skills add rovgit19/alibaba-java-guidelines --skill alibaba-java-guidelines
```

### Claude Code 手动安装

```bash
# 全局
cp -r skills/alibaba-java-guidelines ~/.claude/skills/

# 项目级
cp -r skills/alibaba-java-guidelines .claude/skills/
```

### Claude.ai 网页端

下载仓库 zip → Settings → Customize → Skills → Upload

### Gemini CLI / Codex CLI

```bash
cp -r skills/alibaba-java-guidelines ~/.gemini/skills/
cp -r skills/alibaba-java-guidelines ~/.codex/skills/
```

## 内容覆盖

### P3C 核心规约 (`references/guidelines.md`)

| 章节 | 内容 |
|------|------|
| 📝 编程规约 | 命名、常量、格式、OOP、集合、并发、流程控制、注释 |
| ⚠️ 异常日志 | 异常处理、SLF4J 日志规范 |
| 🗄️ MySQL 规约 | 建表、索引、SQL、ORM |
| 🏗️ 工程结构 | 分层架构、GAV 规范、服务器配置 |
| 🔒 安全规约 | 鉴权、SQL 注入、XSS、CSRF |

### Modern Java 补充 (`references/modern-java.md`)

| 主题 | 要点 |
|------|------|
| ✨ Record & 不可变性 | DTO 用 record，何时不适用；Spring Boot 2.x/3.x 兼容说明 |
| 🔍 Optional | 链式调用，禁止用作字段/参数 |
| 🌊 Stream | pipeline 长度控制，何时退回 for 循环 |
| 🛡️ Null 分层防御 | Bean Validation → Optional → requireNonNull → @Nullable |
| 💥 异常体系设计 | DomainException / RepositoryException / ServiceException + @RestControllerAdvice |
| 🧪 测试规范 | JUnit 5 + AssertJ + Mockito（@Mock/@InjectMocks）；@WebMvcTest/@DataJpaTest 切片 |
| 📋 结构化日志 | key=value 格式 + MDC traceId |
| 🍃 Spring 最佳实践 | 构造器注入、@ConfigurationProperties、事务传播、@Async 自定义线程池 |
| ☕ Java 17 特性 | sealed classes、pattern matching、text blocks、switch expressions |
| 📖 Effective Java 补充 | Builder 模式、组合优于继承、EnumMap/EnumSet、返回空集合而非 null、异常转译 |

## 工作方式

Skill 触发后会自动读取 `references/` 下的两个文件，保证规则覆盖完整。每次会话只加载一次，无需重复触发。

- **日常代码生成**：SKILL.md 内置 Quick Reference，覆盖高频规则
- **代码审查 / 合规检查**：自动读取完整 references，按 [Mandatory] → [Recommended] 顺序逐项检查
- **规范查询**：直接引用具体规则条目及严重级别

## 规则分级

- **[Mandatory]** — 必须遵守，违反可能导致线上故障
- **[Recommended]** — 建议遵守，除非有特殊原因
- **[For Reference]** — 参考建议

## License

MIT
