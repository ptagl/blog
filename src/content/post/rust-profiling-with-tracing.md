---
title: "Profiling Rust applications with `tracing`"
description: "Using `tracing` to generate profiling traces"
publishDate: "15 Mar 2026"
tags: ["rust", "profiling"]
#pinned: true
#updatedDate: "15 Mar 2026"
---

## Introduction

When I think about **profiling**, I do not think about optimization first. I think about visibility and **understanding**.

Profiling is obviously useful when **performance** matters, but that is only half of the story. In practice, it is also one of the best ways to understand how a system actually **behaves**: a function may be called more often than expected, an initialization path may run twice, or a step you assumed was negligible may end up dominating the whole execution.

In this post I will use `profiling` and `tracing` in Rust to generate a **JSON trace** that I can open in **Perfetto UI**. Perfetto can visualize external traces in Chrome JSON format, which makes it a very convenient way to inspect the output produced by `tracing_chrome`.


## A quick introduction to the `tracing` and `profiling` crates

`tracing` is widely used in Rust for **logging** and diagnostics, but its scope is broader than traditional logging. It provides structured **events** and **spans**, which makes it useful not only for recording what happened, but also for tracking execution flow, preserving context, and collecting timing information for **profiling**.

`profiling`, on the other hand, is a thin **abstraction layer** that lets you instrument code in a generic way, without tying your application directly to a specific **profiling backend**. In this post I use it together with `tracing`, but it can work with other profilers as well.


## Creating custom spans

The most direct way to instrument a specific part of your code is to create a **custom span** around it.

```
fn main() {
    // Some code
    ...

    {
        profiling::scope!("Main Application Scope");

        // Code to be tracked
        ...
    }

    // Some other code
    ...
}

```

It is simple, but also surprisingly effective.


## Instrumenting functions

Of course, real world applications are more complex than this dummy `main` function, and it would be inconvenient to call this macro in tens or hundreds of places around the codebase. Luckily, there is a macro that can be used to instrument whole **functions** very quickly:

```
#[profiling::function]
fn very_important_function() {
    ...
}
```

Marking functions with this attribute will make the profiler keep track of every call to it.


## Instrumenting implementations

Even this approach could become annoying if it needs to be replicated for several functions to be instrumented.

There is another attribute that comes in handy to simplify our job:

```

struct VeryImportantStruct {}

#[profiling::all_functions]
impl VeryImportantStruct {
    fn function1() {
        ...
    }

    fn function2() {
        ...
    }

    fn function3() {
        ...
    }
}
```

This tells `profiling` to instrument **every function** found in the implementation body at once, making it way more concise.


## Initializing the profiling and exporting the trace

Most of the job is already done, but we need one more step before being able to collect data from our application.

We need to **initialize** the profiling, specifying the format we need (JSON file compatible with Chrome tracing in this case) and decide when to start and stop collecting data.

It is typically done early in the `main` function or in some other initialization function, but it can also be started later or even on-demand if needed:

```
fn main() {
    let _guard = {
        use tracing_chrome::ChromeLayerBuilder;
        use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

        let (chrome_layer, guard) = ChromeLayerBuilder::new().build();
        tracing_subscriber::registry().with(chrome_layer).init();
        guard
    };

    // The rest of the application logic
    ...
}
```

`tracing_chrome` provides a layer for tracing-subscriber that writes traces in Chrome trace format. Profiling starts immediately after the `init()` call, and the trace file is finalized when the guard returned by the builder is dropped, so it is important to keep it alive for the whole duration of the profiling session.

If it's **dropped earlier** for any reason (e.g. the guard is placed in a dedicated initialization function without being returned), the profiling **stops**. The result of such an error is typically the creation of **empty** JSON traces!


## Analyzing the trace

The **full example** of instrumented Rust application can be found on my GitHub profile. Once everything is set up, run the application and wait for it to stop. At that point, you should see a new **JSON file** generated in the current working directory with a name like `trace-1773521455925975.json`.

There are several ways to open this trace, for instance with the **viewer** built into Chrome (`chrome://tracing`) or with Perfetto UI.

Personally, I prefer [Perfetto](https://ui.perfetto.dev/) because it provides a modern, clean, and fast UI.

Once you open the trace in Perfetto UI, you get a **timeline** view of the recorded spans, which gives you an immediate overview of what happened over the lifetime of the application.

![Perfetto - Timeline Chart](/blog/images/posts/battery-historian/perfetto-trace.png)

By **selecting** spans or time ranges, you can **inspect** durations, nesting, timestamps, and the relationships between different parts of the execution.

That is the kind of detail that is easy to miss in logs and **easy** to spot in a **trace**.
