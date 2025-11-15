# NextChat AI Prompts Documentation

This document provides a comprehensive catalog of all AI prompts used in the NextChat application, organized by functional categories.

## Table of Contents

1. [System Templates and Configuration](#1-system-templates-and-configuration)
2. [Conversation Management Prompts](#2-conversation-management-prompts)
3. [Built-in Assistant Masks - Programming & Development](#3-built-in-assistant-masks---programming--development)
4. [Built-in Assistant Masks - Content Creation & Writing](#4-built-in-assistant-masks---content-creation--writing)
5. [Built-in Assistant Masks - Professional & Educational](#5-built-in-assistant-masks---professional--educational)
6. [Image Generation Prompts](#6-image-generation-prompts)
7. [MCP (Model Context Protocol) Tool Integration](#7-mcp-model-context-protocol-tool-integration)
8. [External Community Prompts](#8-external-community-prompts)

---

## 1. System Templates and Configuration

These are core system prompts that define the basic behavior and context for AI interactions.

### 1.1 Default System Template

**Location:** `app/constant.ts:290-297`

**Purpose:** Default system prompt injected for GPT models to provide context about the model, knowledge cutoff, current time, and LaTeX formatting support.

**Prompt:**
```
You are ChatGPT, a large language model trained by {{ServiceProvider}}.
Knowledge cutoff: {{cutoff}}
Current model: {{model}}
Current time: {{time}}
Latex inline: \(x^2\)
Latex block: $$e=mc^2$$
```

**Template Variables:**
- `{{ServiceProvider}}` - The AI service provider (e.g., "OpenAI")
- `{{cutoff}}` - Knowledge cutoff date
- `{{model}}` - Current model identifier
- `{{time}}` - Current timestamp

---

### 1.2 Default Input Template

**Location:** `app/constant.ts:281`

**Purpose:** Template for processing and formatting user input before sending to the AI model.

**Prompt:**
```
{{input}}
```

**Template Variables:**
- `{{input}}` - Raw user input text

---

### 1.3 Bot Hello Message

**Location:** `app/locales/en.ts:643`

**Purpose:** Default greeting message displayed when starting a new conversation.

**Prompt:**
```
Hello! How can I assist you today?
```

---

### 1.4 Default Conversation Topic

**Location:** `app/locales/en.ts:642`

**Purpose:** Default title for new conversations before auto-generation.

**Prompt:**
```
New Conversation
```

---
