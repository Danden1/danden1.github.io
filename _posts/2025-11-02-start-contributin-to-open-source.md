---
title: Start contributing to open source
description: The Growth Journey of a Junior Developer
author: cotes
date: 2025-11-02 11:33:00 +0900
categories: [Develop, Study]
tags: [java,open source,kestra, good first issue]
math: true
# mermaid: true
---

## Why I started Contributing to open source?

I'm a backend developer with 2 years of experience.

At my company, I work as a backend developer - but in reality, I often find myself writing doc(excel...) mor than actual code.

At some point, I started to feel stuck as a developer. To grow, I thought I should either start a side project or open source.

In the end, I decided to do both.

### In open soruce,

- How to test
  - devcontainer, unit test, class design for better testability.
- How to write better code.
  - I've realized that programmers are a bit like writers - we write code for other developers to read.
  - Writing clean, readable, and maintainnable code is a kind of communication.
  - And performance, data consistency and so on is very important


### In side project,

- I can collaborate with designers, planners, and other developers.
- I can improve **my service** and gain a broader perspective.
- Somtimes, I can even earn a bit of income. 😊


## How to Conritube?


To get started, I searched for issues labled 'good first issues' - designed to helop beginners start contributing.

```yaml
label:"good first issue" updated:>2024 language:Java  state:open no:assignee
```

I searched this query in github.


And I find Kestra project.

I knew this project. I used n8n for team 생산성 향상을 위해 회사에서 사용해봤다.

This is familar with code more than a n8n.(n8n보다 코드 친화적이다.)


## My Contribution

[My PR Link](https://github.com/kestra-io/kestra/pull/12591)


I contirubted by modifying **a plugin icon in Kestra**.

It was a simple task - there was no need to change the code itself. However, I was able to take a closer look at the class design, structure, and other parts of the project.

I also learned how to test my changes using a devcontainer.


### What I Found Interesting in the Structure

This project consists of backend / frontend.

The backend provides plugin images to frontend.

Theses image are located on `core.plugin.resources.icon` package and are served throught the `webserver.controllers.api.PluginController`


The Icons are loaded below code.


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

I fountthis design interesting because it allows icons to be modified easility - thanks to this structure, I didn't have to change the icon code itself.

The backend is built with Micronaut, which was completely new to me since I had only used Spring before.


### For Testing

I didn't know about devcontainer. If there hadn't been a devcontatiner, I would have spent a lot of time for testing my change.

In particular, I tried to build the backend using gradle, but It failed. - even with help from AI(chatgpt, copilot) and Stack Overflow.


I didn't fully understand how it worked, but thanks to the devcontainer, I didn't need to install npm, java and pip.


## Summary

To grow as a developer, I decided to learn from side projects and open source.
Through a small contribution, I discovered many interesting things —
such as class design, internal logic, and the use of devcontainer for testing.

Even though it was a simple change, it gave me a chance to explore how a large project is organized.
When I have more time, I plan to dig deeper into the codebase — especially its test code, overall structure, performance, and data consistency.


