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

## 2. Conversation Management Prompts

These prompts handle automatic conversation features like title generation, summarization, and context management.

### 2.1 Conversation Title Generation

**Location:** `app/locales/en.ts:648-649`, used in `app/store/chat.ts:705`

**Purpose:** Automatically generates a concise 4-5 word title summarizing the conversation topic. This prompt instructs the AI to create a clean, unformatted title without punctuation or extra text.

**Prompt:**
```
Please generate a four to five word title summarizing our conversation without any lead-in, punctuation, quotation marks, periods, symbols, bold text, or additional text. Remove enclosing quotation marks.
```

**Implementation Notes:**
- Triggered automatically after the first user message
- Helps organize and identify conversations quickly
- Returns plain text without formatting

---

### 2.2 Conversation Summarization

**Location:** `app/locales/en.ts:650-651`, used in `app/store/chat.ts:770`

**Purpose:** Creates a concise summary of the conversation (max 200 words) to use as context for future messages. This helps compress long conversation histories while retaining important information.

**Prompt:**
```
Summarize the discussion briefly in 200 words or less to use as a prompt for future context.
```

**Implementation Notes:**
- Used for long-term memory compression
- Helps maintain context when conversation exceeds token limits
- Summary is stored and can be injected into future requests

---

### 2.3 History Context Prefix

**Location:** `app/locales/en.ts:646-647`, used in `app/store/chat.ts:536`

**Purpose:** Prefixes summarized conversation history when injecting it as context into new messages. This helps the AI understand that the following text is a recap of previous conversation.

**Prompt Template:**
```
This is a summary of the chat history as a recap: {content}
```

**Template Variables:**
- `{content}` - The summarized conversation history text

**Implementation Notes:**
- Used in conjunction with conversation summarization
- Provides clear context boundary between summary and current conversation
- Helps AI distinguish between actual history and summary

---
