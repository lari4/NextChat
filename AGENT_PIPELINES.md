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
