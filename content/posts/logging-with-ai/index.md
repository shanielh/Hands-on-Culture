---
title: "How I used AI to modify hundreds of logs messages"
date: 2026-02-09
draft: false
tags: ["AI", "Scala", "Logging"]
---

## Background

As part of Upsolver joining Qlik, we had to make our logs comply with stricter standards. Sensitive data and metadata could no longer appear in production logs.  

The problem is that debugging without that information is significantly harder. So we maintained two versions of each log message: one containing sensitive information, and one compliant version without it.

Upsolver services are deployed inside the customer’s cloud environment, which means customers can still access detailed logs locally, clean sensitive information if needed, and share them with Customer Operations when investigating issues that are otherwise hard to reproduce or solve.

Today, one of my colleagues told me that we have an issue with our logs, they are _either_ saved as compliant
or not, defined by an environment variable; This worked in development, but we can't enable it in production, since it would
also send sensitive data to our servers. Production issues will happen, and when they do, we need those
logs if the customers are willing to give them to us. As we're approaching General Availability, 
it became clear that this must be solved as soon as possible. 

It was obvious what the answer had to be:

> Challenge Accepted

## The Key Idea

Instead of maintaining two versions of every log message, I wanted the compiler
to enforce compliance for me.

A log message should only compile if every interpolated value explicitly declares
how it should be handled:

- hidden
- visible
- hash-coded
- hash-coded path

Instead of relying on developers to remember logging rules, I moved logging compliance 
into the type system, **making the safe thing the only thing that compiles**.

## Changing the logging mechanism

First, I had to change the logging mechanism, instead of defining two versions of logs for each log message:

```scala
logger.compliant.info(s"Creating table: ${tableName}", "Creating table");
```

I wanted the users (My users, the developers) to write logs like this: 

```scala
logger.info(c"Creating table: ${tableName.hidden}")
```

One of the following methods must be used after each argument in order to compile:

- `hidden` - Hides the variable value and replaces it with `<HIDDEN:variable_name>`
- `visible` - Shows the variable value
- `hashCoded` - Shows the hash code of the variable value as hex
- `hashCodedPath` - Split the value with `/` and hashes each part of the path, unless i'ts not sensitive data

So, in order to do that, I created a special "String Interpolation" in Scala. In Scala, even the smallest
constructs of the language are not bound to the compiler. I created `c` (stands for Compliant) string 
interpolation which what it does, it only accepts an Interface (`LogValue`) which has one of the four implementations 
(`hidden`, `visible`, `hashCoded`, and `hashCodedPath`). Each implementation has a `toString` implementation 
that does what it should do, in order to hide sensitive information.

The String Interpolation gives me two arrays:

  - String Parts - The constant parts that come before and after each `${...}`
  - Values - An Array of `LogValue`s that holds the actual values of the variables (and possibly their names!)

This effectively turns a log line into a structured template plus typed arguments. For example, for the log above, the template will be:

```
Creating table: {}
```

And the values will be used as the `args: Any*` of the log message.

## Messing with Log Appenders

We're using [Logback](https://logback.qos.ch) as our Logging Infrastructure, 
together with [Scala-Logging](https://github.com/lightbend-labs/scala-logging).

I asked Copilot (with Claude Sonnet 4.5 as the model) to write me a Decorator Appender
that extracts the real values out of the `LogValue`, call it `CompliantValueExtractorAppender`, it did a great job,
now, all I needed to do "Decorate" the "FileAppender" with it, for example, If I had :

```xml
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logs/${sun.java.command}.log</file>
    <!-- More to come here -->
</appender>
```

I should have now:

```xml
<appender name="FILE" class="com.upsolver.common.logging.CompliantValueExtractorAppender">
    <appender-ref ref="INNER_FILE"/>
</appender>
<appender name="INNER_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logs/${sun.java.command}.log</file>
    <!-- More to come here -->
</appender>
```

Now, each message that is sent to the `FILE` appender, will be extracted with the real values,
and the sensitive information _will_ be saved there. 

While other loggers, that are not decorated with this new appender, will hide sensitive data.

A performance bonus we get by using Scala-Logging is that log messages that goes to `/dev/null`
due to logger levels are actually not causing any performance penalty. My new methods
are marked as `@inline`, which hints the compiler to inline the methods if they're small enough; And
Scala-Logging uses a Macro in Scala that actually wraps the line with an `if` statement.

So actually, writing: `logger.info(c"Hello ${world.visible}")`, turns to:

```scala
if (logger.underlying.isInfoEnabled) {
    logger.underlying.info(c"Hello ${world.visible}");
}
```

## Modifying Existing Log Messages

Going back to the first part of the solution, I knew we had to "Migrate" all the log lines:

```scala
// OLD 
logger.compliant.info(s"Creating table: ${tableName}", "Creating table");
// NEW
logger.info(c"Creating table: ${tableName.hidden}")
```

I knew that I could use Copilot w/ Claude to do so, but I thought to myself, AI is great
for getting to know code, but it isn't so great for repetitive tasks. That's where [Scalafix](https://scalacenter.github.io/scalafix/)
comes in.

### Scalafix

Scalafix is a refactoring and linting tool for Scala, it extracts Scala's Abstract Syntax Tree (therefore: AST)
and allows you to:

- Create a custom Linting Rules - that adds warnings
- Create a custom Refactoring Rules - that patch existing code with new AST

Pretty cool, huh?

Ok, now I can use AI to create a new Rule that modifies old log lines with new one, the logic is pretty simple:

- Arguments that are shown in non-compliant message and in compliant should be suffixed with `.visible`
- Arguments that are shown in non-compliant message but doesn't show in compliant should be suffixed with `.hidden`
- Arguments that are shown in non-compliant message and hashed in compliant should be suffixed with `.hashCoded`
- Arguments that are shown in non-compliant message and are paths should be suffixed with `.hashCodedPath`

AI did that job and wrote the code in a few seconds, *it didn't have to go over all log messages*, only on the code
of the old compliant logger, the new logger extension method, and the new `c` String Interpolation.

I had to tell it to fix the code a few times before running the new Rule, unfortunately, we don't have Plan mode 
in IntelliJ yet.

The rest of the job was done by running the new Scalafix rule. As a bonus, the rule doubles as a linting rule — any log line that doesn’t use the new compliant string interpolation fails CI, preventing regressions going forward.

## Conclusion

AI is great, it changed the way we seek for information, it changed the way we write documents, 
and it challenges the way we write code, but it has its own drawbacks. For example, 
ask AI to modify a variable name that is widely used and wait forever, use Refactor Tool in the
IDE and it will take less than a second 
(Hi Github - please add tools that use the IDE and LSP capabilities!)

I used AI to write a code that rewrites code - I read the "Rule" code, it was pretty obvious what it does.
I believe that telling Copilot to modify all the log lines would have taken much more time, 
and it would probably start hallucinate at some point in the middle of the processing, since the context window would be vast.

**For me, this showcases our new role as engineers: to design, to show the path, and let AI do the rest.**

At the end of the day, I'm thankful for choosing Scala (❤️) again as our primary coding language, 
such a language with a small, but _thoughtful_ ecosystem that allowed me to accomplish these changes
in less than a few hours. 

Achievement unlocked 🔓.