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

## 2. Conversation Management Workflows

### Overview
Automated workflows for generating conversation titles and creating memory summaries to compress long conversations.

### 2.1 Title Generation Workflow

#### Entry Point
- **File:** `app/store/chat.ts`
- **Function:** `summarizeSession()`
- **Lines:** 661-797
- **Trigger:** Called by `onNewMessage()` after successful response

#### Trigger Conditions
```javascript
// Lines 687-692
(config.enableAutoGenerateTitle === true) AND
(session.topic === DEFAULT_TOPIC) AND  // Still using default "New Conversation"
(countMessages(messages) >= 50)        // Enough context for title

OR

(refreshTitle === true)  // User manually requests title regeneration
```

#### ASCII Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│            TRIGGER: onNewMessage(botMessage)                     │
│            After successful AI response                          │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         CHECK: Should Generate Title?                            │
│  File: chat.ts:687-692                                          │
│                                                                  │
│  Conditions:                                                    │
│    ✓ Auto-generate enabled in config                           │
│    ✓ Topic is still default ("New Conversation")               │
│    ✓ Message count >= 50                                       │
│  OR                                                              │
│    ✓ Manual refresh requested                                   │
└────────────────────┬────────────────────────────────────────────┘
                     │ YES
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 1: MESSAGE SELECTION                                │
│  File: chat.ts:693-701                                          │
│                                                                  │
│  Process:                                                       │
│    1. Take last historyMessageCount messages                    │
│    2. Append topic generation prompt                            │
│                                                                  │
│  Messages Array:                                                │
│    [                                                            │
│      { role: "user", content: "..." },                         │
│      { role: "assistant", content: "..." },                    │
│      { role: "user", content: "..." },                         │
│      ...last N messages...                                     │
│      { role: "user", content: TOPIC_PROMPT }  ← Added           │
│    ]                                                             │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 2: PROMPT INJECTION                                 │
│  Prompt Used: Locale.Store.Prompt.Topic                         │
│  Location: app/locales/en.ts:648-649                            │
│                                                                  │
│  English Prompt:                                                │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Please generate a four to five word title summarizing    │ │
│  │ our conversation without any lead-in, punctuation,       │ │
│  │ quotation marks, periods, symbols, bold text, or         │ │
│  │ additional text. Remove enclosing quotation marks.       │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Chinese Prompt (translated):                                   │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Return a 4-5 word topic summary directly without         │ │
│  │ explanation, punctuation, or extra text. If no topic,    │ │
│  │ return "Chat"                                             │ │
│  └───────────────────────────────────────────────────────────┘ │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 3: MODEL SELECTION                                  │
│  File: chat.ts:154-171                                          │
│  Function: getSummarizeModel(currentModel)                      │
│                                                                  │
│  Logic:                                                         │
│    IF currentModel.startsWith("gpt")                            │
│      → Use SUMMARIZE_MODEL = "gpt-4o-mini"                      │
│    ELSE IF currentModel.startsWith("gemini")                    │
│      → Use GEMINI_SUMMARIZE_MODEL = "gemini-pro"                │
│    ELSE IF currentModel.startsWith("deepseek")                  │
│      → Use DEEPSEEK_SUMMARIZE_MODEL = "deepseek-chat"           │
│    ELSE                                                          │
│      → Use currentModel                                         │
│                                                                  │
│  Reason: Use cheaper, faster model for simple title generation  │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 4: API REQUEST                                      │
│  File: chat.ts:708-725                                          │
│                                                                  │
│  Configuration:                                                 │
│    {                                                            │
│      messages: [...recentMessages, topicPrompt],               │
│      config: {                                                  │
│        model: summarizeModel,                                   │
│        stream: false,  ← Non-streaming for simplicity          │
│        ...otherSettings                                         │
│      },                                                          │
│      onUpdate: null,  // Not needed for non-streaming          │
│      onFinish: (message) => {                                   │
│        session.topic = trimTopic(message);                      │
│      }                                                           │
│    }                                                             │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 5: RESPONSE PROCESSING                              │
│  File: chat.ts:708-725 (onFinish callback)                     │
│                                                                  │
│  Function: trimTopic(message)                                   │
│    - Removes quotation marks                                    │
│    - Trims whitespace                                           │
│    - Limits length if necessary                                 │
│                                                                  │
│  Updates: session.topic = cleaned title                         │
│                                                                  │
│  Example:                                                       │
│    Raw response: "\"Python Programming Help\""                  │
│    After trim:   "Python Programming Help"                      │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         EXIT: TITLE UPDATED                                      │
│                                                                  │
│  Result: Conversation title displayed in sidebar                │
│  Storage: Saved to session.topic                                │
│  UI Update: Sidebar re-renders with new title                   │
└─────────────────────────────────────────────────────────────────┘
```

#### Example Flow

**Input Conversation:**
```
User: How do I sort a list in Python?
Assistant: You can use the sorted() function or the .sort() method...
User: What's the difference between them?
Assistant: sorted() creates a new list, while .sort() modifies in-place...
[...more messages...]
```

**Title Generation Request:**
```
Messages: [last 32 messages] + "Please generate a four to five word title..."
Model: gpt-4o-mini
```

**Response:**
```
"Python List Sorting Methods"
```

**Result:**
```
session.topic = "Python List Sorting Methods"
```

---

### 2.2 Summarization Workflow

#### Overview
Compresses long conversation history into concise summaries to stay within token limits while preserving context.

#### Entry Point
- **File:** `app/store/chat.ts`
- **Function:** `summarizeSession()`
- **Lines:** 727-796
- **Trigger:** Same as title generation, runs after title is complete

#### Trigger Conditions
```javascript
// Lines 758-760
(historyMsgLength > modelConfig.compressMessageLengthThreshold) AND
(modelConfig.sendMemory === true)
```

#### ASCII Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│         CHECK: Should Summarize?                                 │
│  File: chat.ts:758-760                                          │
│                                                                  │
│  Conditions:                                                    │
│    ✓ History length > compressMessageLengthThreshold           │
│      (default: 1000 characters)                                 │
│    ✓ sendMemory is enabled in model config                     │
└────────────────────┬────────────────────────────────────────────┘
                     │ YES
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 1: MESSAGE COLLECTION                               │
│  File: chat.ts:727-747                                          │
│                                                                  │
│  Process:                                                       │
│    1. Determine start index:                                    │
│       startIndex = MAX(                                         │
│         session.lastSummarizeIndex,                             │
│         session.clearContextIndex                               │
│       )                                                          │
│                                                                  │
│    2. Collect messages from startIndex to current               │
│                                                                  │
│    3. Filter out error messages:                                │
│       messages.filter(m => !m.isError)                          │
│                                                                  │
│    4. Limit to historyMessageCount if exceeds max_tokens        │
│                                                                  │
│    5. Prepend existing memoryPrompt if exists:                  │
│       [{ role: "system", content: memoryPrompt }, ...msgs]      │
│                                                                  │
│  Output: Messages array ready for summarization                 │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 2: PROMPT INJECTION                                 │
│  Prompt Used: Locale.Store.Prompt.Summarize                     │
│  Location: app/locales/en.ts:650-651                            │
│                                                                  │
│  English Prompt:                                                │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Summarize the discussion briefly in 200 words or less     │ │
│  │ to use as a prompt for future context.                    │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Chinese Prompt (translated):                                   │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Briefly summarize the conversation for use as context     │ │
│  │ prompt, within 200 words                                  │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Appended to: messages array                                    │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 3: API REQUEST (STREAMING)                          │
│  File: chat.ts:766-795                                          │
│                                                                  │
│  Configuration:                                                 │
│    {                                                            │
│      messages: [...collectedMessages, summarizePrompt],        │
│      config: {                                                  │
│        model: summarizeModel,  // gpt-4o-mini or equivalent    │
│        stream: true  ← Streaming for real-time display         │
│      },                                                          │
│      onUpdate: (message) => {                                   │
│        session.memoryPrompt = message;  // Live update          │
│      },                                                          │
│      onFinish: (message) => {                                   │
│        session.memoryPrompt = message;                          │
│        session.lastSummarizeIndex = session.messages.length;    │
│      },                                                          │
│      onError: (error) => { /* Handle error */ }                 │
│    }                                                             │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 4: SUMMARY STORAGE                                  │
│  File: chat.ts:766-795 (onFinish callback)                     │
│                                                                  │
│  Actions:                                                       │
│    1. Save summary:                                             │
│       session.memoryPrompt = finalSummary                       │
│                                                                  │
│    2. Update tracking:                                          │
│       session.lastSummarizeIndex = currentMessageCount          │
│                                                                  │
│    3. Next request will inject summary via:                     │
│       Locale.Store.Prompt.History(memoryPrompt)                 │
│       = "This is a summary of the chat history as a             │
│          recap: " + memoryPrompt                                │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         EXIT: MEMORY COMPRESSED                                  │
│                                                                  │
│  Result: Long conversation → 200-word summary                   │
│  Benefit: Reduces token usage while preserving context          │
│  Reuse: Summary injected in future requests (Step 3 of          │
│         Message Processing Pipeline)                             │
└─────────────────────────────────────────────────────────────────┘
```

#### Example Flow

**Input (50 messages about Python programming):**
```
[Message 1-50 discussing lists, dictionaries, loops, functions, etc.]
Total: ~5000 characters
Threshold: 1000 characters
```

**Summarization Request:**
```
Messages: [All 50 messages]
Prompt: "Summarize the discussion briefly in 200 words or less..."
Model: gpt-4o-mini
Stream: true
```

**Generated Summary (Example):**
```
The conversation covered Python programming fundamentals. We discussed:
1. List operations - sorting with sorted() vs .sort(), list comprehensions
2. Dictionary manipulation - accessing keys/values, dict comprehensions
3. Loop constructs - for loops, while loops, enumerate(), zip()
4. Function basics - def keyword, parameters, return values, *args/**kwargs
5. Error handling - try/except blocks, raising exceptions

Key takeaways: sorted() creates new lists while .sort() modifies in-place;
dictionary comprehensions provide concise dict creation; enumerate() gives
index+value pairs; *args captures variable positional arguments.
```

**Storage:**
```javascript
session.memoryPrompt = "The conversation covered Python programming..."
session.lastSummarizeIndex = 50
```

**Future Request Context:**
```
System Message: "This is a summary of the chat history as a recap:
                 The conversation covered Python programming..."
[Only last 10 recent messages]
User: "How do I read a file in Python?"
```

#### Data Flow Comparison

```
WITHOUT SUMMARIZATION:
┌──────────────────────────────────────┐
│ Request to API (exceeds token limit) │
│                                       │
│ System Prompt (500 tokens)            │
│ Message 1-100 (10,000 tokens) ← Too much!
│ User Query (50 tokens)                │
└──────────────────────────────────────┘
Result: Context truncated or error

WITH SUMMARIZATION:
┌──────────────────────────────────────┐
│ Request to API (within limits)        │
│                                       │
│ System Prompt (500 tokens)            │
│ Memory Summary (200 tokens) ← Compressed!
│ Message 90-100 (500 tokens)           │
│ User Query (50 tokens)                │
└──────────────────────────────────────┘
Result: Full context preserved, no errors
```

### Workflow Interaction

```
Message Processing Pipeline (Section 1)
        │
        │ onFinish() callback
        ▼
  onNewMessage(botMessage)
        │
        ├─────────────┬─────────────┐
        │             │             │
        ▼             ▼             ▼
Title Generation  Summarization  Continue Chat
(if conditions  (if conditions
    met)            met)
        │             │
        └──────┬──────┘
               │
               ▼
        Continue Chat with
        updated topic and/or
        compressed memory
```

---

## 3. MCP Tool Calling Pipeline

### Overview
The Model Context Protocol (MCP) pipeline enables the AI to execute system tools and primitives. The AI detects when it needs a tool, generates properly formatted JSON, the system executes the tool, and returns results for the AI to process.

### Entry Point
- **File:** `app/store/chat.ts`
- **Function:** `checkMcpJson(message)`
- **Lines:** 826-856
- **Trigger:** After AI generates a response containing MCP JSON code block

### MCP System Initialization

Before tools can be called, MCP clients must be initialized:

**File:** `app/mcp/actions.ts`
- **Function:** `initializeMcpSystem()` (lines 142-161)
- **Config:** Loaded from `app/mcp/mcp_config.json`

```
Application Startup
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│         MCP SYSTEM INITIALIZATION                                │
│  File: app/mcp/actions.ts:142-161                               │
│                                                                  │
│  FOR EACH server in mcp_config.json:                            │
│    1. Create StdioClientTransport                               │
│       - command: e.g., "npx"                                    │
│       - args: e.g., ["-y", "@modelcontextprotocol/server-fs"]   │
│       - env: Environment variables                              │
│                                                                  │
│    2. Initialize MCP Client                                     │
│       - Connect to transport                                    │
│       - Exchange capabilities                                   │
│                                                                  │
│    3. List Available Tools                                      │
│       - Call client.listTools()                                 │
│       - Store tool definitions                                  │
│                                                                  │
│    4. Store in clientsMap                                       │
│       clientsMap[serverId] = {                                  │
│         client: mcpClient,                                      │
│         tools: toolsList,                                       │
│         status: "active",                                       │
│         errorMsg: null                                          │
│       }                                                          │
│                                                                  │
│  Example Tools for "filesystem" server:                         │
│    - list_allowed_directories                                   │
│    - read_file                                                  │
│    - write_file                                                 │
│    - list_directory                                             │
│    - create_directory                                           │
└─────────────────────────────────────────────────────────────────┘
```

### Tool Calling Flow

#### ASCII Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│         USER REQUEST NEEDS TOOL                                  │
│  "List the files in my documents folder"                        │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 1: AI RESPONSE WITH MCP JSON                        │
│  File: Message Processing Pipeline (Section 1)                  │
│                                                                  │
│  AI receives MCP_SYSTEM_TEMPLATE prompt instructing:            │
│  Location: app/constant.ts:306-421                              │
│                                                                  │
│  Key Instructions:                                              │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ 2. WHEN TO USE TOOLS:                                     │ │
│  │    - ALWAYS USE TOOLS when they can help                  │ │
│  │    - DO NOT just describe - TAKE ACTION immediately       │ │
│  │                                                            │ │
│  │ 3. HOW TO USE TOOLS:                                      │ │
│  │    A. Tool Call Format:                                   │ │
│  │       ```json:mcp:{clientId}```                           │ │
│  │       {                                                    │ │
│  │         "method": "tools/call",                           │ │
│  │         "params": {                                        │ │
│  │           "name": "tool_name",                            │ │
│  │           "arguments": { ... }                            │ │
│  │         }                                                  │ │
│  │       }                                                    │ │
│  │                                                            │ │
│  │    C. Important Rules:                                    │ │
│  │       - Only use tools/call method                        │ │
│  │       - Only ONE tool call per message                    │ │
│  │       - ALWAYS TAKE ACTION (no asking permission)         │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                  │
│  AI Generates:                                                  │
│  ```json:mcp:filesystem                                         │
│  {                                                               │
│    "method": "tools/call",                                      │
│    "params": {                                                  │
│      "name": "list_directory",                                 │
│      "arguments": {                                             │
│        "path": "/Users/username/Documents"                     │
│      }                                                           │
│    }                                                             │
│  }                                                               │
│  ```                                                             │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 2: MCP JSON DETECTION                               │
│  File: chat.ts:826-856                                          │
│  Function: checkMcpJson(message)                                │
│                                                                  │
│  Sub-function: isMcpJson(content)                               │
│  File: app/mcp/utils.ts:1-3                                     │
│  Pattern: /```json:mcp:([^{\s]+)([\s\S]*?)```/                 │
│                                                                  │
│  Process:                                                       │
│    1. Check if message.content contains MCP JSON pattern        │
│    2. If NOT found → return false (normal message flow)         │
│    3. If FOUND → Extract and process                            │
└────────────────────┬────────────────────────────────────────────┘
                     │ MCP JSON DETECTED
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 3: EXTRACTION                                       │
│  Function: extractMcpJson(content)                              │
│  File: app/mcp/utils.ts:5-11                                    │
│                                                                  │
│  Extracts:                                                      │
│    {                                                            │
│      clientId: "filesystem",  // From code block tag           │
│      mcp: {                   // Parsed JSON object            │
│        method: "tools/call",                                    │
│        params: {                                                │
│          name: "list_directory",                               │
│          arguments: {                                           │
│            path: "/Users/username/Documents"                   │
│          }                                                       │
│        }                                                         │
│      }                                                           │
│    }                                                             │
│                                                                  │
│  Validation:                                                    │
│    - clientId must exist                                        │
│    - JSON must be valid                                         │
│    - method must be "tools/call"                                │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 4: TOOL EXECUTION                                   │
│  File: app/mcp/actions.ts:337-352                               │
│  Function: executeMcpAction(clientId, mcpRequest)               │
│                                                                  │
│  Process:                                                       │
│    1. Retrieve client from clientsMap:                          │
│       const clientInfo = clientsMap.get(clientId)               │
│                                                                  │
│    2. Check client status:                                      │
│       IF status !== "active" → throw error                      │
│                                                                  │
│    3. Execute via MCP SDK:                                      │
│       File: app/mcp/client.ts:50-55                             │
│       Function: executeRequest(client, request)                 │
│                                                                  │
│       Uses @modelcontextprotocol/sdk:                           │
│       result = await client.request(                            │
│         {                                                        │
│           method: "tools/call",                                 │
│           params: {                                              │
│             name: "list_directory",                             │
│             arguments: { path: "/Users/username/Documents" }    │
│           }                                                      │
│         },                                                       │
│         CallToolResultSchema                                    │
│       )                                                          │
│                                                                  │
│    4. Tool executes (in MCP server process):                    │
│       - Reads directory via Node.js fs module                   │
│       - Returns file list                                       │
│                                                                  │
│  Result Example:                                                │
│    {                                                            │
│      content: [                                                 │
│        {                                                        │
│          type: "text",                                          │
│          text: "file1.txt\nfile2.pdf\nsubfolder/"              │
│        }                                                         │
│      ]                                                           │
│    }                                                             │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 5: RESPONSE INJECTION                               │
│  File: chat.ts:838-849                                          │
│                                                                  │
│  Format result as MCP response:                                 │
│    content = ```json:mcp-response:${clientId}                   │
│    ${JSON.stringify(result, null, 2)}                           │
│    ```                                                           │
│                                                                  │
│  Example:                                                       │
│    ```json:mcp-response:filesystem                              │
│    {                                                             │
│      "content": [                                               │
│        {                                                        │
│          "type": "text",                                        │
│          "text": "file1.txt\nfile2.pdf\nsubfolder/"            │
│        }                                                         │
│      ]                                                           │
│    }                                                             │
│    ```                                                           │
│                                                                  │
│  Call onUserInput() with:                                       │
│    - content: formatted MCP response                            │
│    - isMcpResponse: true (skips template filling)               │
│                                                                  │
│  This triggers a NEW request to AI with tool result!            │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         STEP 6: AI PROCESSES TOOL RESULT                         │
│  Back to Message Processing Pipeline (Section 1)                │
│                                                                  │
│  Messages sent to AI:                                           │
│    [                                                            │
│      { role: "user",                                            │
│        content: "List the files in my documents folder" },      │
│      { role: "assistant",                                       │
│        content: "```json:mcp:filesystem\n{...}\n```" },         │
│      { role: "user",  ← Tool result injected as user message   │
│        content: "```json:mcp-response:filesystem\n{...}\n```" } │
│    ]                                                             │
│                                                                  │
│  AI understands it received tool result and responds:           │
│    "I found 3 items in your Documents folder:                   │
│     - file1.txt                                                 │
│     - file2.pdf                                                 │
│     - subfolder/ (directory)"                                   │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│         EXIT: TOOL RESULT EXPLAINED TO USER                      │
│                                                                  │
│  User sees: Natural language explanation of tool result         │
│  Behind scenes: Tool was called, result processed, explained    │
│  Message history: Contains both tool call and result            │
└─────────────────────────────────────────────────────────────────┘
```

### Multi-Turn Tool Calling Example

```
CONVERSATION FLOW:

Turn 1:
  User: "Create a folder called 'projects' and then list all folders"

Turn 2:
  AI: Generates MCP call for create_directory
  ```json:mcp:filesystem
  {
    "method": "tools/call",
    "params": {
      "name": "create_directory",
      "arguments": { "path": "/Users/username/projects" }
    }
  }
  ```

Turn 3:
  System: Executes tool, injects response
  ```json:mcp-response:filesystem
  { "content": [{ "type": "text", "text": "Directory created" }] }
  ```

Turn 4:
  AI: Receives confirmation, makes SECOND tool call
  ```json:mcp:filesystem
  {
    "method": "tools/call",
    "params": {
      "name": "list_allowed_directories",
      "arguments": {}
    }
  }
  ```

Turn 5:
  System: Executes second tool, injects response
  ```json:mcp-response:filesystem
  {
    "content": [{
      "type": "text",
      "text": "/Users/username/\n/Users/username/projects/\n..."
    }]
  }
  ```

Turn 6:
  AI: Processes both results, responds to user
  "I've created the 'projects' folder. Here are all your folders:
   - Home directory
   - projects (newly created)
   - ..."
```

### MCP Tools Template Injection

**How available tools are communicated to AI:**

```
┌─────────────────────────────────────────────────────────────────┐
│         getMessagesWithMemory() - MCP Section                    │
│  File: chat.ts:205-224                                          │
│  Function: getMcpSystemPrompt()                                 │
│                                                                  │
│  IF MCP enabled:                                                │
│                                                                  │
│    1. Get MCP_SYSTEM_TEMPLATE from constant.ts:306-421          │
│                                                                  │
│    2. Build tools list for each client:                         │
│       FOR EACH client in clientsMap:                            │
│         toolsList += MCP_TOOLS_TEMPLATE                         │
│         Replace {{clientId}} with client ID                     │
│         Replace {{tools}} with JSON.stringify(client.tools)     │
│                                                                  │
│    3. Inject tools into MCP_SYSTEM_TEMPLATE:                    │
│       Replace {{MCP_TOOLS}} with all toolsList                  │
│                                                                  │
│  Example Result:                                                │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ You are an AI assistant with access to system tools.     │ │
│  │                                                            │ │
│  │ 1. AVAILABLE TOOLS:                                       │ │
│  │                                                            │ │
│  │ [clientId]                                                │ │
│  │ filesystem                                                │ │
│  │ [tools]                                                   │ │
│  │ [                                                         │ │
│  │   {                                                       │ │
│  │     "name": "read_file",                                  │ │
│  │     "description": "Read contents of a file",             │ │
│  │     "inputSchema": {                                      │ │
│  │       "type": "object",                                   │ │
│  │       "properties": {                                     │ │
│  │         "path": { "type": "string" }                      │ │
│  │       },                                                  │ │
│  │       "required": ["path"]                                │ │
│  │     }                                                      │ │
│  │   },                                                      │ │
│  │   ...more tools...                                        │ │
│  │ ]                                                         │ │
│  │                                                            │ │
│  │ 2. WHEN TO USE TOOLS: [instructions...]                  │ │
│  │ 3. HOW TO USE TOOLS: [format examples...]                │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                  │
│  This prompt is inserted as system message BEFORE user message  │
└─────────────────────────────────────────────────────────────────┘
```

### Error Handling

**Client Initialization Failure:**
```
Error during MCP init → clientsMap[id].errorMsg = error
                     → clientsMap[id].status = "error"
                     → Toast notification shown to user
```

**Tool Execution Failure:**
```
Tool call error → Catch in executeMcpAction
               → Show error toast
               → Throw error to chat
               → AI sees error message
               → AI can retry with different arguments or different tool
```

**Invalid JSON Format:**
```
AI generates wrong format → JSON parse error
                         → Error injected as MCP response
                         → AI receives error, learns from mistake
                         → MCP_SYSTEM_TEMPLATE includes examples of
                           WRONG formats to avoid
```

### Data Transformation

**Tool Definition (from MCP server) → AI-readable format:**
```javascript
// MCP Server provides
{
  name: "write_file",
  description: "Write content to a file",
  inputSchema: {
    type: "object",
    properties: {
      path: { type: "string", description: "File path" },
      content: { type: "string", description: "Content to write" }
    },
    required: ["path", "content"]
  }
}

// Injected into MCP_SYSTEM_TEMPLATE as:
{
  "name": "write_file",
  "description": "Write content to a file",
  "inputSchema": {
    "type": "object",
    "properties": {
      "path": { "type": "string", "description": "File path" },
      "content": { "type": "string", "description": "Content to write" }
    },
    "required": ["path", "content"]
  }
}

// AI generates tool call
{
  "method": "tools/call",
  "params": {
    "name": "write_file",
    "arguments": {
      "path": "/Users/test/file.txt",
      "content": "Hello World"
    }
  }
}

// System executes → Result
{
  "content": [{
    "type": "text",
    "text": "File written successfully"
  }]
}
```

### Performance Notes
- **One Tool Per Turn:** MCP enforces one tool call per message to prevent chaos
- **Client Reuse:** MCP clients persist across conversation, avoiding reconnection overhead
- **Async Execution:** Tool calls are async, allowing long-running operations
- **Error Recovery:** AI can retry failed tools with corrected parameters

---
