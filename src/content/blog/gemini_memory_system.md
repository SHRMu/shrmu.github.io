---
title: 'Gemini CLI Memory System'
description: 'A deep dive into the layered memory system of the Gemini CLI agent.'
pubDate: '2025-08-11'
---
# Gemini CLI Memory System

This document describes the layered memory system used by the Gemini CLI agent, allowing it to handle different types of information effectively, from immediate conversational context to long-term project rules.

## 1. Short-Term Conversational Memory

This is the most basic layer. It's the history of our current conversation.

*   **What it is:** The transcript of the messages we've exchanged in this session.
*   **Purpose:** It allows the agent to understand the immediate context, follow a train of thought, and refer to things just discussed (e.g., if you say "what about that file?", it knows which file you mean).
*   **Persistence:** It's temporary and is reset when a new session starts.

## 2. Long-Term Persistent Memory

This is for information that needs to last across sessions. It's primarily managed through files and a dedicated tool.

*   **A. Global `GEMINI.md` (User-level Rules):**
    *   **Location:** A central configuration directory on the user's system (e.g., `C:\Users\username\.gemini\GEMINI.md`).
    *   **Purpose:** To store global preferences and general rules that the agent should follow for **all** projects. For example, "Always open files in my editor before changing them."
    *   **How it's updated:** The user and the agent can edit this file to establish a baseline behavior.

*   **B. Project `GEMINI.md` (Project-level Rules):**
    *   **Location:** The root directory of the project being worked on.
    *   **Purpose:** To store rules, architecture details, and conventions specific **only to that project**. This is where things like "All diagrams must use React Flow" or "This project uses `pnpm` for package management" are defined.
    *   **Hierarchy:** If a rule in the project `GEMINI.md` conflicts with a rule in the global one, the **project-specific rule always wins**.

*   **C. Saved Facts (`save_memory` tool):**
    *   **Concept:** This is a "soft" or "lightweight" method for achieving persistence. The user doesn't need to edit a file; they can simply ask the agent to remember a piece of information. The agent then uses this tool to store the fact persistently, and it's automatically loaded in future sessions.
    *   **Why it's needed:**
        *   **Convenience:** It's much faster and more conversational to say "Remember my API key is XYZ" than to manually open a `GEMINI.md` file to record it.
        *   **Scope:** It is designed for small, atomic, and context-independent facts. It is not suitable for complex, structured information like project architecture or multi-step procedures.
        *   **Separation of Concerns:** It keeps the formal project documentation (`GEMINI.md`) clean and focused on high-level rules and architecture, while small, ad-hoc personal facts are stored separately.
    *   **Persistence Mechanism:** When this tool is used, the information is saved locally in a persistent manner. The agent does not manage the file directly; the host environment that runs the agent handles the storage and retrieval. When a new session starts, the environment automatically loads these saved facts back into the agent's memory.
    *   **Example:** A user might say, "Remember that my preferred branch naming convention is `feature/ticket-123`." The agent would then use the `save_memory` tool to store that fact.

## 3. "Working Memory" (Active Context Gathering)

This isn't a file; it's the process the agent uses to understand the current state of a project for a specific task.

*   **What it is:** A temporary, in-session understanding of the codebase that the agent builds on the fly.
*   **How it works:** When given a task (e.g., "Refactor this component"), the agent uses its tools (`read_file`, `glob`, `search_file_content`) to read the relevant files, check for existing conventions, find dependencies, and analyze the code.
*   **Purpose:** This is the most critical part of the software engineering workflow. It ensures that the agent doesn't make blind assumptions and that its changes align with the project's actual structure and style. This "memory" is dynamic and is constantly being updated as the agent works on a task.

### Clarification: Conversational vs. Working Memory

The distinction between Short-Term Conversational Memory and Working Memory is crucial.

*   **Conversational Memory is the *Instructions***: It's the chat history. It's you telling the agent *what* to do.
*   **Working Memory is the *Investigation***: It's the information the agent gathers using its tools (reading files, running commands) to figure out *how* to do it.

| Aspect    | Short-Term Conversational Memory        | "Working Memory" (Active Context)             |
| :-------- | :-------------------------------------- | :-------------------------------------------- |
| **Content** | The history of our chat messages        | File contents, command outputs, search results |
| **Source**  | Our direct conversation                 | The agent's tools (`read_file`, `run_shell_command`) |
| **Purpose** | To understand the user's requests       | To understand the codebase and execute the task |

### Summary Table

| Memory Type                 | What it Stores                                      | How it's Loaded/Updated                               | Persistence          |
| --------------------------- | --------------------------------------------------- | ----------------------------------------------------- | -------------------- |
| **Conversational Memory**   | Current chat history                                | Automatically, as we talk                         | Session-only         |
| **Global `GEMINI.md`**      | User's universal rules and preferences                | By editing the file in the user's config directory     | Permanent (Global)   |
| **Project `GEMINI.md`**     | Project-specific rules and architecture             | By editing the file in the project root               | Permanent (Project)  |
| **Saved Facts**             | Small, explicit facts the user asks to remember        | Via the `save_memory` tool                            | Permanent (Global)   |
| **Working Memory**          | On-the-fly analysis of the code for a specific task | Via tools like `read_file`, `glob`, `run_shell_command` | Task-specific        |

---

### 摘要表 (中文版)

| 内存类型         | 存储内容                                   | 加载/更新方式                                           | 持久性       |
| ---------------- | ------------------------------------------ | ------------------------------------------------------- | ------------ |
| **对话内存**     | 当前的聊天记录                             | 在我们交谈时自动记录                                    | 仅限当前会话 |
| **全局 `GEMINI.md`** | 用户的通用规则和偏好                       | 通过编辑用户配置目录中的文件                            | 永久（全局） |
| **项目 `GEMINI.md`** | 项目特定的规则和架构                       | 通过编辑项目根目录中的文件                              | 永久（项目） |
| **已保存的记忆** | 用户要求记住的零散、明确信息               | 通过 `save_memory` 工具                                 | 永久（全局） |
| **工作内存**     | 针对特定任务对代码的即时分析               | 通过 `read_file`、`glob`、`run_shell_command` 等工具 | 任务特定     |