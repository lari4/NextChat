# NextChat AI Agent Pipelines Documentation

This document describes all AI agent pipelines, workflows, and data flows in the NextChat application. Each pipeline is documented with ASCII diagrams showing the sequence of operations, prompts used, and data transformations.

## Table of Contents

1. [Message Processing Pipeline](#1-message-processing-pipeline)
2. [Conversation Management Workflows](#2-conversation-management-workflows)
3. [MCP Tool Calling Pipeline](#3-mcp-tool-calling-pipeline)
4. [Image Generation Workflows](#4-image-generation-workflows)
5. [API Request/Response Cycles](#5-api-requestresponse-cycles)
6. [Error Handling Flows](#6-error-handling-flows)

---

## 1. Message Processing Pipeline

### Overview
The core pipeline for processing user messages, applying templates, assembling context, and streaming AI responses.

### Entry Point
- **File:** `app/store/chat.ts`
- **Function:** `onUserInput(content, attachImages?, isMcpResponse?)`
- **Lines:** 407-528

### ASCII Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     USER INPUT RECEIVED                          │
│                    (text + optional images)                      │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│               STEP 1: TEMPLATE APPLICATION                       │
│  File: chat.ts:416-428                                          │
│  Function: fillTemplateWith(content, modelConfig)               │
│                                                                  │
│  Prompt Used: DEFAULT_INPUT_TEMPLATE = "{{input}}"              │
│  Location: app/constant.ts:281                                  │
│                                                                  │
│  Variables Injected:                                            │
│    - {{ServiceProvider}} → "OpenAI", "Anthropic", etc.         │
│    - {{cutoff}} → Knowledge cutoff date                         │
│    - {{model}} → Current model name                             │
│    - {{time}} → Current timestamp                               │
│    - {{lang}} → User language                                   │
│    - {{input}} → User message text                              │
│                                                                  │
│  Output: Template-filled user message                           │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│            STEP 2: MESSAGE OBJECT CREATION                       │
│  File: chat.ts:430-440                                          │
│                                                                  │
│  Creates two ChatMessage objects:                               │
│                                                                  │
│  userMessage = {                                                │
│    role: "user",                                                │
│    content: processed_content,  // text or multimodal array     │
│    date: new Date().toLocaleString()                            │
│  }                                                               │
│                                                                  │
│  botMessage = {                                                 │
│    role: "assistant",                                           │
│    content: "",                                                 │
│    streaming: true,                                             │
│    date: new Date().toLocaleString()                            │
│  }                                                               │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 3: MEMORY & CONTEXT ASSEMBLY                        │
│  File: chat.ts:542-640                                          │
│  Function: getMessagesWithMemory()                              │
│                                                                  │
│  Assembles messages in order:                                   │
│                                                                  │
│  1. SYSTEM PROMPT (if enabled)                                  │
│     Prompt: DEFAULT_SYSTEM_TEMPLATE                             │
│     Location: app/constant.ts:290-297                           │
│     Content:                                                    │
│     ┌─────────────────────────────────────────────────────┐   │
│     │ You are ChatGPT, a large language model            │   │
│     │ trained by {{ServiceProvider}}.                     │   │
│     │ Knowledge cutoff: {{cutoff}}                        │   │
│     │ Current model: {{model}}                            │   │
│     │ Current time: {{time}}                              │   │
│     │ Latex inline: \(x^2\)                               │   │
│     │ Latex block: $$e=mc^2$$                             │   │
│     └─────────────────────────────────────────────────────┘   │
│                                                                  │
│  2. MCP SYSTEM PROMPT (if MCP enabled)                          │
│     Prompt: MCP_SYSTEM_TEMPLATE                                 │
│     Location: app/constant.ts:306-421                           │
│     Function: getMcpSystemPrompt() (lines 205-224)              │
│     Content: Tool descriptions + usage instructions             │
│                                                                  │
│  3. LONG-TERM MEMORY (if exists)                                │
│     Format: Locale.Store.Prompt.History(memoryPrompt)           │
│     English: "This is a summary of the chat history             │
│              as a recap: " + memoryPrompt                       │
│                                                                  │
│  4. CONTEXT PROMPTS (from mask.context)                         │
│     Predefined conversation starters from selected mask         │
│                                                                  │
│  5. RECENT MESSAGES                                             │
│     Limited by:                                                 │
│     - historyMessageCount (e.g., 32 messages)                   │
│     - max_tokens (respects token limit)                         │
│                                                                  │
│  Output: Complete messages array for API request                │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│              STEP 4: MESSAGE STORAGE                             │
│  File: chat.ts:448-457                                          │
│                                                                  │
│  Action: Appends to session.messages                            │
│    - userMessage                                                │
│    - botMessage (with streaming: true)                          │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│               STEP 5: API REQUEST                                │
│  File: chat.ts:459-527                                          │
│                                                                  │
│  getClientApi(modelConfig.providerName).llm.chat({              │
│    messages: messagesWithMemory,                                │
│    config: modelConfig,                                         │
│    onUpdate: (message) => {                                     │
│      botMessage.content = message;  // Incremental update       │
│    },                                                            │
│    onFinish: (message) => {                                     │
│      botMessage.content = message;                              │
│      onNewMessage(botMessage);  // Triggers title/summary       │
│    },                                                            │
│    onError: (error) => {                                        │
│      botMessage.content += "\n\n" + prettyObject(error);        │
│    }                                                             │
│  })                                                              │
│                                                                  │
│  Routes to: app/client/platforms/[provider].ts                  │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│              EXIT: RESPONSE STREAMING                            │
│                                                                  │
│  Three possible outcomes:                                       │
│                                                                  │
│  SUCCESS:                                                       │
│    → botMessage updated incrementally via onUpdate              │
│    → onFinish triggers conversation management workflows        │
│    → streaming: false set on completion                         │
│                                                                  │
│  ERROR:                                                         │
│    → botMessage.isError = true                                  │
│    → Error message appended to content                          │
│                                                                  │
│  TOOL CALL DETECTED:                                            │
│    → Routes to Tool Calling Pipeline (see section 3)            │
└─────────────────────────────────────────────────────────────────┘
```

### Key Data Transformations

**Input → Template Processing:**
```javascript
// Before
content = "What is 2+2?"

// After fillTemplateWith()
content = "What is 2+2?"  // Simple case, just passes through

// With custom template (if user configured)
// Template: "Query: {{input}}\nPlease answer in {{lang}}"
content = "Query: What is 2+2?\nPlease answer in English"
```

**Multimodal Content Structure:**
```javascript
// Text only
content = "Describe this image"

// With images
content = [
  { type: "text", text: "Describe this image" },
  { type: "image_url", image_url: { url: "data:image/png;base64,..." } }
]
```

### Performance Notes
- **Message Limit:** Respects `historyMessageCount` to prevent context overflow
- **Token Budget:** `getMessagesWithMemory()` drops old messages when exceeding `max_tokens`
- **Streaming:** Uses Server-Sent Events (SSE) for incremental updates

---
