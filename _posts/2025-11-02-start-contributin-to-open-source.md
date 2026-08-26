---
title: My First Open-Source Contribution
description: What I learned from making my first open-source contribution
date: 2025-11-02 11:33:00 +0900
categories: [Develop, Study]
tags: [java,open source,kestra, good first issue]
math: true
# mermaid: true
---

## Why I Started Contributing to Open Source

I'm a backend developer with 2 years of experience.

At my company, I work as a backend developer—but in reality, I often find myself writing documentation, mostly Excel files, more often than actual code.

At some point, I started to feel stuck as a developer. To grow, I thought I should either start a side project or contribute to open source.

In the end, I decided to do both.

### Through Open-Source Contributions

- How to test
  - Devcontainers, unit tests, and class design for better testability.
- How to write better code.
  - I've realized that programmers are a bit like writers—we write code for other developers to read.
  - Writing clean, readable, and maintainable code is a kind of communication.
  - Performance and data consistency also matter.


### Through Side Projects

- I can collaborate with designers, planners, and other developers.
- I can build and improve **a service of my own** and gain a broader perspective.
- Sometimes, I can even earn a little extra income.


## How to Contribute


To get started, I searched for issues labeled `good first issue`, which are designed to help beginners start contributing.

```yaml
label:"good first issue" updated:>2024 language:Java  state:open no:assignee
```

I used this query on GitHub.


That was how I found the Kestra project.

I already knew about this project — I once used n8n at work to improve team productivity.
Kestra feels more code-oriented than n8n.

## My Contribution

[My PR Link](https://github.com/kestra-io/kestra/pull/12591)


I contributed by modifying **a plugin icon in Kestra**.

It was a simple task—there was no need to change the code itself. However, I was able to take a closer look at the class design, structure, and other parts of the project.

I also learned how to test my changes using a devcontainer.


### What I Found Interesting in the Structure

This project consists of a backend and a frontend.

The backend provides plugin images to the frontend.

These images are located in the `core.plugin.resources.icon` package and are served through `webserver.controllers.api.PluginController`.


The icons are loaded by the code below.


```java
package io.kestra.core.plugins;


@SneakyThrows
public String icon(Class<?> cls) {
    InputStream resourceAsStream = Stream
        .of(
            this.getClassLoader().getResourceAsStream("icons/" + cls.getName() + ".svg"),
            this.getClassLoader().getResourceAsStream("icons/" + cls.getPackageName() + ".svg")
        )
        .filter(Objects::nonNull)
        .findFirst()
        .orElse(null);

    if (resourceAsStream != null) {
        return Base64.getEncoder().encodeToString(
            IOUtils.toString(resourceAsStream, StandardCharsets.UTF_8).getBytes(StandardCharsets.UTF_8)
        );
    }
    return null;
}
```

I found this design interesting because it allows icons to be modified easily. Thanks to this structure, I didn't have to change the icon-handling code itself.

The backend is built with Micronaut, which was completely new to me since I had only used Spring before.


### Testing

I didn't know about devcontainers. Without the devcontainer, I would have spent a lot of time testing my change.

In particular, I tried to build the backend using Gradle, but it failed—even with help from AI tools such as ChatGPT and Copilot, as well as Stack Overflow.


I didn't fully understand how it worked, but thanks to the devcontainer, I didn't need to install npm, Java, or pip.


## Summary

To grow as a developer, I decided to learn through side projects and open-source contributions.
Through a small contribution, I discovered several interesting things, including class design, internal logic, and the use of a devcontainer for testing.

Even though it was a simple change, it gave me a chance to explore how a large project is organized.
When I have more time, I plan to dig deeper into the codebase — especially its test code, overall structure, performance, and data consistency.
