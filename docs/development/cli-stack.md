# CLI Plugin System Architecture & Stack

This document outlines the chosen technology stack for building a modern, extensible, and high-performance Command-Line Interface (CLI) application with a robust plugin system. The stack prioritizes speed, TypeScript support, seamless developer experience, and robust environment configuration, leveraging the best tools from the [UnJS](https://unjs.io/) ecosystem and complementary modern libraries.

---

## 🧰 Core Stack Overview

| Tool | Purpose | Ecosystem |
| :--- | :--- | :--- |
| **[dotenv](https://github.com/motdotla/dotenv)** | Environment variables and `.env` file loading | Community Standard |
| **[Citty](https://github.com/unjs/citty)** | Argument parsing and command routing | UnJS |
| **[Consola](https://github.com/unjs/consola)** | Beautiful logging, spinners, and error formatting | UnJS |
| **[@clack/prompts](https://github.com/bombshell-dev/clack)** | Interactive data input and wizards | Community |
| **[giget](https://github.com/unjs/giget)** | Reliable downloading from GitHub and npm | UnJS |
| **[hookable](https://github.com/unjs/hookable)** | Event interception and plugin lifecycle management | UnJS |
| **[C12](https://github.com/unjs/c12)** | Smart configuration loading from user projects | UnJS |

---

## 📚 Tool Justification & Details

### 1. [dotenv](https://github.com/motdotla/dotenv) – Environment Variables & `.env` Management
**Role:** Early-stage loading of environment variables from `.env` files.
**Why it was chosen:**
- **Early Bootstrapping:** CLI applications often need environment variables (e.g., API tokens for GitHub/npm, debug flags, proxy settings) *before* any config files or commands are parsed. `dotenv` allows synchronous, immediate loading at the very entry point of the application.
- **Universal Standard:** It is the most battle-tested, widely recognized solution for `.env` parsing in Node.js, supporting multiple environments (`.env`, `.env.local`, `.env.production`).
- **Synergy with C12:** While [C12](https://github.com/unjs/c12) has built-in `.env` support for config resolution, explicitly using `dotenv` at the entry point ensures that *all* subsequent tools (including Citty, Consola, and Giget) have immediate access to `process.env`.

### 2. [Citty](https://github.com/unjs/citty) – Command Routing & Parsing
**Role:** The foundational CLI framework.
**Why it was chosen:**
- **Declarative & Type-Safe:** Commands, subcommands, and arguments are defined as typed objects, ensuring perfect TypeScript inference.
- **Zero Boilerplate:** Automatically generates comprehensive `--help` output and handles version flags.
- **UnJS Native:** Seamlessly integrates with other UnJS tools, ensuring consistent behavior and minimal bundle size.

### 3. [Consola](https://github.com/unjs/consola) – Logging & UI Feedback
**Role:** The visual output engine for the CLI.
**Why it was chosen:**
- **Rich Formatting:** Provides out-of-the-box support for colored logs, boxed messages, and animated spinners (e.g., `consola.start()`, `consola.success()`).
- **Cross-Platform:** Works flawlessly across Node.js, Deno, Bun, and even browser environments.
- **Standardization:** It is the de facto logging standard in the UnJS ecosystem (used by Nuxt, Unbuild, etc.), guaranteeing long-term maintenance.

### 4. [@clack/prompts](https://github.com/bombshell-dev/clack) – Interactive Input
**Role:** Handling interactive user prompts when CLI arguments are omitted.
**Why it was chosen:**
- **Modern UI:** Delivers a clean, macOS/Unix-native aesthetic that feels premium and responsive.
- **First-Class ESM & TS:** Designed from the ground up for modern JavaScript environments, avoiding the legacy CommonJS pitfalls of older tools like `Inquirer.js`.
- **Graceful Cancellation:** Built-in, elegant handling of `Ctrl+C` via the `onCancel` callback, preventing messy stack traces.

### 5. [giget](https://github.com/unjs/giget) – Plugin Downloading Engine
**Role:** Fetching plugin source code from remote registries.
**Why it was chosen:**
- **Monorepo Support:** Natively understands syntax like `gh:user/repo/packages/my-plugin`, downloading *only* the specified subdirectory without fetching the entire repository history.
- **Multi-Source:** Uniformly handles `npm:package-name`, `gh:user/repo`, and `gl:user/repo` (GitLab) with a single API.
- **Isolation:** Downloads plugins into a dedicated directory (e.g., `.myapp_plugins/`), keeping them isolated from the host application's `node_modules`.

### 6. [hookable](https://github.com/unjs/hookable) – Architectural Glue
**Role:** Event-driven communication between the CLI core and plugins.
**Why it was chosen:**
- **Async-First:** All hooks natively support `async/await`, which is critical for I/O operations like file reading or network requests.
- **Decoupling:** Allows plugins to intercept lifecycle events (e.g., `plugin:before-install`, `cli:command:run`) and mutate context data without requiring any modifications to the core CLI codebase.
- **Lightweight:** Weighs less than 1KB, adding zero overhead to the application bundle.

### 7. [C12](https://github.com/unjs/c12) – Configuration Management
**Role:** Loading and merging user-defined configuration files.
**Why it was chosen:**
- **Smart Resolution:** Automatically searches for config files (e.g., `myapp.config.ts`, `.myapprc.json`) starting from the current working directory up to the root.
- **TypeScript Support:** Uses `jiti` under the hood to seamlessly load `.ts` and `.mts` configuration files without requiring the user to pre-compile them.
- **Layered Merging:** Effortlessly merges default configs, user configs, and environment-specific overrides (and natively respects `.env` variables during this resolution).

---

## 🔄 Architectural Flow Example

When a user runs `mycli install gh:org/plugin`:

1. **dotenv** loads `.env` and `.env.local` files at the very entry point, populating `process.env` with necessary tokens or debug flags.
2. **Citty** parses the `install` command and the `gh:org/plugin` positional argument.
3. **C12** loads the user's `myapp.config.ts` to check for any default plugin registries, respecting the environment variables loaded in step 1.
4. **hookable** triggers the `plugin:before-install` hook, allowing registered core modules or existing plugins to modify the download URL or validate the request.
5. **giget** downloads the plugin source into `.myapp_plugins/org-plugin` and runs `npm install` inside that directory if a `package.json` is present.
6. **Consola** provides real-time visual feedback (spinners, success messages) throughout the process.
7. **hookable** triggers the `plugin:after-install` hook, allowing the newly installed plugin to register its own commands or perform initialization tasks.

If the user runs `mycli install` without arguments, **@clack/prompts** intercepts the flow, displays an interactive text input, and feeds the result back into step 4.
