---
title: "Dev Containers: Rust Quick Start"
description: "How to set up a Rust development environment using Dev Containers in VS Code"
publishDate: "16 Jan 2026"
tags: ["rust", "dev-containers", "vscode"]
#pinned: true
#updatedDate: "12 Jan 2026"
---

## Introduction

Dev Containers let you use Docker to run an isolated development environment, so that your tooling and runtime stack can live “somewhere else” instead of on your host machine.
​
A `devcontainer.json` file in your repo defines how to create or connect to that development container (and which tools/extensions/settings to include).
​
Dev Containers have become pretty mainstream lately, and I’m finding them especially useful in several scenarios, for instance when I need to temporarily set up a development environment or a specific toolchain for a quick test I'll later discard, and I don't want to pollute my host machine.

## Requirements

To follow this tutorial, you need to have:

- Docker
- Visual Studio Code with the "Dev Containers" extension installed

I tested this on Ubuntu 24.04 on WSL2 (Windows 11), but it should work on any platform supported by Docker and VS Code, with minor adjustments.

## Adding the Dev Container configuration

First of all, let's create our project folder:

```bash
mkdir hello-dev-container && cd hello-dev-container
```

Then, let's create a file called `.devcontainer/devcontainer.json` in the project root with the following content:

```json
{
  "name": "Rust Hello World (Dev Container)",
  "image": "mcr.microsoft.com/devcontainers/rust:latest",
  "customizations": {
    "vscode": {
      "extensions": ["rust-lang.rust-analyzer"]
    }
  }
}
```

If you haven't done it yet, open the project root folder with VS Code:

```bash
code .
```

Then use the Command Palette (`Ctrl` + `Shift` + `P`), and select `Dev Containers: Reopen in Container`. This command may take a while the first time as Docker needs to download the base image and set up the container. Once finished, the blue bottom-left corner of VS Code should indicate that you are connected to the Dev Container. As an additional confirmation, you can open a terminal in VS Code and see that the environment looks different from your host machine (e.g., the current user is `vscode`).

At this point, you can continue working as usual; the fact that you are inside a Dev Container is mostly transparent.

## Hello World Rust Template

To test the Dev Container with Rust, we need at least a minimal project that can be compiled, run, and eventually debugged. To do so, let's open a VS Code terminal and type:

```bash
cargo init --bin --name hello-dev-container
cargo run
```

You should see the "Hello, world!" string printed to the terminal. The same could be done directly from VS Code by clicking on the buttons on top of the `main()` function. Debugging works as well.

## Testing the environment isolation

To verify that the Dev Container is indeed isolated from your host machine, let's suppose that we want to test our "Hello World" Rust application with the latest nightly toolchain. We could install it on our host machine, but maybe we won't need it beyond this quick test, and we are too lazy to uninstall it later. Instead, we can simply install it inside the Dev Container.

To force `cargo` to use a specific toolchain version, let's create a file called `rust-toolchain.toml` in the project root with the following content:

```
[toolchain]
channel = "nightly-2026-01-15"
```

VS Code should automatically detect the change and attempt to refresh the project, but the Rust Analyzer extension may fail as the specified toolchain is not yet installed inside the Dev Container.
If you open the extension log, you should see an error like this:

```
toolchain 'nightly-2026-01-15-x86_64-unknown-linux-gnu' is not installed
help: run `rustup toolchain install` to install it
```

Let's accept the suggestion and run the command in the VS Code terminal inside the Dev Container:

```bash
rustup toolchain install
```
Once finished, you may need to restart Rust Analyzer, and then the IDE is fully functional again. You can notice, for instance, that we can run or debug the application by clicking on the buttons on top of the `main()` function.

Since I prefer to verify rather than trust, let's make a quick check. From the IDE terminal, run `rustup show`. It lists the toolchains installed on the Dev Container, and should include `nightly-2026-01-15`. Now, if you have Rustup installed on the host machine, you can do the same and compare the output: that toolchain version should not appear unless you installed it on your own, regardless of this tutorial.