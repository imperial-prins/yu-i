# AionUi Personal Assistant Feature Development Plan

> This document records the complete development plan for the personal assistant feature, including architecture design, plugin system, interaction design, etc.

---

## 1. Feature Overview

### 1.1 Basic Information

- **Feature Name**: Personal Assistant Feature
- **Module**: Agent Layer, Dialogue System
- **Processes Involved**: Main Process, Worker
- **Runtime Environment**: GUI Mode (AionUi Running)

### 1.2 Feature Description

1. Similar to the WebUI function, users can directly use Aion features through a personal terminal.
2. Mainly involves IM communication tools for personal users (Telegram, Lark/Feishu, etc.).
3. Creates a 24/7 personal terminal assistant.
4. **Supported Platforms**: Telegram (grammY), Lark/Feishu (Official SDK).
5. **Supported Agents**: Gemini, ACP, Codex.

### 1.3 User Scenarios

```
Trigger: User sends a message via mobile IM tool (like Telegram)
Process: Platform bot receives message → Forwards to Aion Agent → LLM processing
Result: After processing, pushes the result to the user through the same platform
```

### 1.4 Reference Projects

- **Clawdbot**: https://github.com/clawdbot/clawdbot
- Adopt its design concepts such as plugin-based design, pairing security mode, Channel abstraction, etc.

---

## 2. Overall Architecture

### 2.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                  ChannelManager (Singleton)                   │
│                  (Unified management of all components)       │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │
│  │PluginManager│ │SessionManager│ │PairingService│           │
│  └─────────────┘ └─────────────┘ └─────────────┘            │
│  ┌─────────────┐ ┌─────────────────────────────┐            │
│  │ActionExecutor│ │ChannelMessageService         │            │
│  └─────────────┘ └─────────────────────────────┘            │
└────────────────────┼────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                     Layer 1: Plugin                          │
│                     (Platform Adaptation Layer)              │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────┐ ┌──────────┐                                   │
│  │ Telegram │ │  Lark    │  ... (Slack, Discord to be implemented) │
│  │  Plugin  │ │  Plugin  │                                   │
│  └────┬─────┘ └────┬─────┘                                   │
│       └────────────┴────────────┘                           │
│                    │                                         │
│  Responsibilities: Receive platform messages/callbacks → Convert to unified format → Send responses │
│  Does not care about: Agent type, business logic            │
└────────────────────┼────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                     Layer 2: Gateway                         │
│                     (Business Logic Layer)                   │
├─────────────────────────────────────────────────────────────┤
│  ActionExecutor: System Action handling, dialogue routing    │
│  SessionManager: Session management, user authorization      │
│  PairingService: Pairing code generation and verification    │
│  ChannelMessageService: Message stream processing            │
│                                                              │
│  Responsibilities: System Action handling, dialogue routing, session management, permission control │
│  Does not care about: Platform details, Agent implementation details │
└────────────────────┼────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                     Layer 3: Agent                           │
│                     (AI Processing Layer)                    │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                      │
│  │ Gemini  │  │   ACP   │  │  Codex  │                      │
│  │  Agent  │  │  Agent  │  │  Agent  │                      │
│  └─────────┘  └─────────┘  └─────────┘                      │
│                                                              │
│  Responsibilities: Communicate with AI services, manage dialogue context, return unified response │
│  Does not care about: Message source platform, system-level operations │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Data Flow

```
Inbound Flow:
  Platform Message → Plugin(Conversion) → ActionExecutor(Routing) → Agent(Processing)

  Detailed Process:
  1. Plugin receives platform message → toUnifiedIncomingMessage()
  2. PluginManager calls messageHandler → ActionExecutor.handleMessage()
  3. ActionExecutor routes based on Action type:
     - Platform Action → Handled by Plugin itself
     - System Action → Handled by SystemActions
     - Chat Action → ChannelMessageService → Agent

Outbound Flow:
  Agent Response → ChannelEventBus → ChannelMessageService → ActionExecutor → Plugin(Conversion) → Sent to Platform

  Detailed Process:
  1. Agent Worker sends message → ChannelEventBus.emitAgentMessage()
  2. ChannelMessageService listens to event → handleAgentMessage()
  3. transformMessage + composeMessage → StreamCallback
  4. ActionExecutor calls context.sendMessage/editMessage()
  5. Plugin converts message format → sendMessage/editMessage()
```

---

## 3. Plugin System Design

### 3.1 Plugin Responsibility Boundaries

| Plugin Is Responsible For                | Plugin Is Not Responsible For      |
| ---------------------------------------- | ---------------------------------- |
| Connecting to platform API               | Agent scheduling and execution     |
| Receiving message → Converting to unified format | Session management and persistence |
| Unified format → Converting to platform message | User authentication and permission control |
| Handling platform-specific commands      | Message routing decisions          |
| Stream message updates (editing sent messages) |                                    |

### 3.2 Plugin Lifecycle

```
created → initializing → ready → starting → running → stopping → stopped
                ↓                    ↓           ↓
              error ←←←←←←←←←←←←←←←←←←←←←←←←←←←←
```

| Status         | Description                          |
| -------------- | ------------------------------------ |
| `created`      | Plugin instance has been created     |
| `initializing` | Validating configuration and initializing |
| `ready`        | Initialization complete, waiting to start |
| `starting`     | Connecting to platform               |
| `running`      | Running normally                     |
| `stopping`     | Disconnecting                        |
| `stopped`      | Stopped                              |
| `error`        | Error occurred                       |

### 3.3 Plugin Interface (BasePlugin Abstract Class)

| Interface Method       | Direction               | Description                           |
| ---------------------- | ----------------------- | ------------------------------------- |
| `initialize(config)`   | PluginManager → Plugin  | Initialize plugin configuration       |
| `start()`              | PluginManager → Plugin  | Start platform connection             |
| `stop()`               | PluginManager → Plugin  | Stop platform connection              |
| `sendMessage(...)`     | ActionExecutor → Plugin | Send message to platform              |
| `editMessage(...)`     | ActionExecutor → Plugin | Edit sent message (stream update)     |
| `getStatus()`          | PluginManager → Plugin  | Get plugin status                     |
| `getActiveUserCount()` | PluginManager → Plugin  | Get active user count                 |
| `getBotInfo()`         | PluginManager → Plugin  | Get Bot info                          |
| `onInitialize()`       | Implemented by subclass | Platform-specific initialization logic|
| `onStart()`            | Implemented by subclass | Platform-specific startup logic       |
| `onStop()`             | Implemented by subclass | Platform-specific stopping logic      |

### 3.4 Unified Message Format

**Inbound Message (Platform → System)** - `IUnifiedIncomingMessage`

| Field              | Description                                       |
| ------------------ | ------------------------------------------------- |
| `id`               | System-generated unique ID                        |
| `platform`         | Source platform (telegram/lark/slack/discord)     |
| `chatId`           | Chat ID                                           |
| `user`             | User info (id, username, displayName)             |
| `content`          | Message content (type, text, attachments)         |
| `timestamp`        | Timestamp                                         |
| `replyToMessageId` | Replied message ID (optional)                     |
| `action`           | Action info (for button callbacks)                |
| `raw`              | Original platform message (optional)              |

**Outbound Message (System → Platform)** - `IUnifiedOutgoingMessage`

| Field              | Description                                  |
| ------------------ | -------------------------------------------- |
| `type`             | Message type (text/image/file/buttons)       |
| `text`             | Text content                                 |
| `parseMode`        | Parse mode (HTML/Markdown/MarkdownV2)        |
| `buttons`          | Inline button group (optional)               |
| `keyboard`         | Reply Keyboard (optional)                    |
| `replyMarkup`      | Platform-specific Markup (optional, e.g., Lark Card) |
| `replyToMessageId` | Replied message ID (optional)                |
| `imageUrl`         | Image URL (image type)                       |
| `fileUrl`          | File URL (file type)                         |
| `fileName`         | File name (file type)                        |
| `silent`           | Silent send (optional)                       |

### 3.5 Steps to Extend New Platform

1. Create `src/channels/plugins/[platform]/` directory
2. Implement `[Platform]Plugin` extending `BasePlugin`
3. Implement `[Platform]Adapter` to handle message conversion (toUnifiedIncomingMessage, to[Platform]SendParams)
4. Register plugin in `ChannelManager` constructor: `registerPlugin('platform', PlatformPlugin)`
5. Add platform type to `PluginType` in `types.ts`
6. Add settings page UI
7. Add i18n translation
8. Implement platform-specific interaction components (like Keyboard, Card, etc.)

---

## 4. Implemented Platforms

### 4.1 Telegram Integration

#### Technology Stack

| Item           | Selection         | Description                    |
| -------------- | ----------------- | ------------------------------ |
| Bot Library    | grammY            | Used by Clawdbot, elegant API  |
| Operation Mode | Polling           | Automatic reconnection mechanism |

### 4.1 Technology Stack

| Item           | Selection                        | Description                    |
| -------------- | -------------------------------- | ------------------------------ |
| Bot Library    | grammY                           | Used by Clawdbot, elegant API  |
| Operation Mode | Polling (Dev) / Webhook (Prod)   | Configurable                   |

#### Bot Configuration Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Create Bot                                          │
│   User uses @BotFather in Telegram → /newbot → Get Token    │
├─────────────────────────────────────────────────────────────┤
│ Step 2: Configure Token                                     │
│   AionUi settings page → Paste Token → Verify → Save        │
├─────────────────────────────────────────────────────────────┤
│ Step 3: Start Bot                                           │
│   Turn on switch → Bot starts listening                     │
├─────────────────────────────────────────────────────────────┤
│ Step 4: User Pairing (See security mechanism below)         │
└─────────────────────────────────────────────────────────────┘
```

#### Configuration Items

| Configuration Item | Type              | Description                                |
| ------------------ | ----------------- | ------------------------------------------ |
| Bot Token          | string            | Obtained from @BotFather                   |
| Operation Mode     | polling / webhook | Polling is suitable for development        |
| Webhook URL        | string            | Only needed for webhook mode               |
| Pairing Mode       | boolean           | Whether pairing code authorization is needed |
| Rate Limit         | number            | Maximum messages per minute                |
| Group @Mention     | boolean           | Whether @bot is required to respond in groups |
| Default Agent      | gemini            | Fixed to Gemini in MVP phase               |

#### Pairing Security Mechanism (Adopted from Clawdbot Mode)

**Core Principle**: Approval operation is done on the user's local device, not in Telegram.

```
┌─────────────────────────────────────────────────────────────┐
│ ① User initiates in Telegram                                 │
│    User → @YourBot: /start or any message                    │
├─────────────────────────────────────────────────────────────┤
│ ② Bot returns pairing request                                │
│    Bot → User:                                               │
│    "👋 Welcome to Aion Assistant!                             │
│     Your pairing code: ABC123                                │
│     Please approve this pairing in AionUi:                   │
│     Settings → Telegram → Pending Requests → [Approve]"      │
├─────────────────────────────────────────────────────────────┤
│ ③ AionUi displays pending request                            │
│    Settings page shows: Username, pairing code, request time, [Approve]/[Reject] │
├─────────────────────────────────────────────────────────────┤
│ ④ User clicks [Approve] in AionUi                            │
├─────────────────────────────────────────────────────────────┤
│ ⑤ Bot notifies successful pairing                            │
│    Bot → User: "✅ Pairing successful! You can start chatting now" │
└─────────────────────────────────────────────────────────────┘
```

**Security Measures**

| Mechanism              | Description                                      |
| ---------------------- | ------------------------------------------------ |
| Pairing Code Auth      | 6-digit random code, valid for 10 minutes        |
| Local Approval         | Must be approved in AionUi, not in Telegram      |
| User Whitelist         | Only authorized users can use                    |
| Rate Limit             | Prevent abuse                                    |
| Token Encrypted Storage| Stored using bcrypt encryption                   |

#### Message Conversion Rules

**Inbound Conversion (Telegram → Unified Format)**

| Telegram Message Type | Unified Message content.type     |
| --------------------- | -------------------------------- |
| `message:text`        | `text` or `command` (starts with /) |
| `message:photo`       | `image`                          |
| `message:document`    | `file`                           |
| `message:voice`       | `audio`                          |

**Outbound Conversion (Unified Format → Telegram)**

| Unified Message type | Telegram API                      |
| -------------------- | --------------------------------- |
| `text`               | `sendMessage`                     |
| `image`              | `sendPhoto`                       |
| `file`               | `sendDocument`                    |
| `buttons`            | `sendMessage` + `inline_keyboard` |

**Special Handling**

| Scenario         | Handling Method                              |
| ---------------- | -------------------------------------------- |
| Stream Response  | Use `editMessageText` to update message, add ▌ cursor |
| Markdown         | Escape special characters, use `parse_mode: Markdown` |
| @Mention Removal | Clean up `@bot_username` in messages         |
| Group Filtering  | Check if @mention is included (configurable) |

### 4.2 Lark/Feishu Integration

#### Technology Stack

| Item           | Selection                        | Description                 |
| -------------- | -------------------------------- | --------------------------- |
| SDK            | @larksuiteoapi/node-sdk          | Official SDK                |
| Operation Mode | WebSocket Long Connection        | No public URL required      |
| Domain         | Feishu (configurable to Lark International) | Default uses Feishu domain |

#### Bot Configuration Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Create Application                                  │
│   Create enterprise custom app in Feishu Open Platform → Get App ID and App Secret │
├─────────────────────────────────────────────────────────────┤
│ Step 2: Configure Permissions                               │
│   App Permission Management → Enable "Receive and send single chat, group messages" permissions │
├─────────────────────────────────────────────────────────────┤
│ Step 3: Configure Event Subscription                        │
│   Event Subscription → Subscribe to "Receive Message" event → Configure encryption key (optional) │
├─────────────────────────────────────────────────────────────┤
│ Step 4: Configure Credentials                               │
│   AionUi settings page → Paste App ID, App Secret → Verify → Save │
├─────────────────────────────────────────────────────────────┤
│ Step 5: Start Bot                                           │
│   Turn on switch → Bot starts listening via WebSocket connection │
├─────────────────────────────────────────────────────────────┤
│ Step 6: User Pairing (See security mechanism below)         │
└─────────────────────────────────────────────────────────────┘
```

#### Configuration Items

| Configuration Item | Type    | Description                            |
| ------------------ | ------- | -------------------------------------- |
| App ID             | string  | Obtained from Feishu Open Platform     |
| App Secret         | string  | Obtained from Feishu Open Platform     |
| Encrypt Key        | string  | Event encryption key (optional)        |
| Verification Token | string  | Event verification token (optional)    |
| Pairing Mode       | boolean | Whether pairing code authorization is needed |
| Rate Limit         | number  | Maximum messages per minute            |
| Default Agent      | gemini  | Fixed to Gemini in MVP phase           |

#### Pairing Security Mechanism

Same as Telegram, adopts local approval mode. The pairing code is sent to the user via Lark message, and the user approves it in AionUi.

#### Message Conversion Rules

**Inbound Conversion (Lark → Unified Format)**

| Lark Message Type | Unified Message content.type       |
| ----------------- | ---------------------------------- |
| `message:text`    | `text` or `command` (starts with /)|
| `message:image`   | `photo`                            |
| `message:file`    | `document`                         |
| `message:audio`   | `audio`                            |
| Card Action       | `action` (via extractCardAction)   |

**Outbound Conversion (Unified Format → Lark)**

| Unified Message type | Lark API                   |
| -------------------- | -------------------------- |
| `text`               | `im.message.create`        |
| `buttons`            | `im.message.create` + Card |
| Interactive Card     | Use Lark Card format       |

**Special Handling**

| Scenario             | Handling Method                                        |
| -------------------- | ------------------------------------------------------ |
| Stream Response      | Use `im.message.update` to update message              |
| HTML to Markdown     | convertHtmlToLarkMarkdown() converts HTML to Lark Markdown |
| Card Interaction     | Use Lark Card format, supports buttons, confirmations, etc. |
| Event Deduplication  | 5-minute event cache to prevent duplicate processing   |

---

## 5. Interaction Design

### 5.1 Design Principles

**Buttons first, commands reserved**: Ordinary users operate via buttons, advanced users can use commands.

### 5.2 Telegram Interaction Components

| Type                | Description                 | Applicable Scenarios         |
| ------------------- | --------------------------- | ---------------------------- |
| **Inline Keyboard** | Buttons below the message   | Operation confirmation, option selection |
| **Reply Keyboard**  | Replaces input keyboard     | Quick access to common operations |
| **Menu Button**     | Left of chat input box      | Fixed feature access         |

### 5.3 Interaction Scenario Design

**Scenario 1: First Use / Pairing**

```
Bot Message:
┌─────────────────────────────────────────┐
│ 👋 Welcome to Aion Assistant!           │
│                                         │
│ 🔑 Pairing Code: ABC123                 │
│ Please approve this pairing in AionUi settings │
│                                         │
│ [📖 User Guide]  [❓ Get Help]          │
└─────────────────────────────────────────┘
```

**Scenario 2: After Successful Pairing (Reply Keyboard persistent)**

```
┌─────────────────────────────────────────┐
│ ... Chat Content ...                    │
├─────────────────────────────────────────┤
│ Reply Keyboard (Persistent quick operations)│
│ [🆕 New Chat] [📊 Status] [❓ Help]     │
├─────────────────────────────────────────┤
│ [Enter message...]              [Send]  │
└─────────────────────────────────────────┘
```

**Scenario 3: AI Reply with Action Buttons**

````
Bot Message:
┌─────────────────────────────────────────┐
│ Here is an implementation of quicksort: │
│                                         │
│ ```python                               │
│ def quicksort(arr):                     │
│     ...                                 │
│ ```                                     │
│                                         │
│ [📋 Copy] [🔄 Regenerate] [💬 Continue] │
└─────────────────────────────────────────┘
````

**Scenario 4: Settings Page (Card-based Selection)**

```
Bot Message:
┌─────────────────────────────────────────┐
│ ⚙️ Settings                              │
│                                         │
│ ┌─────────────────────────────────────┐ │
│ │ 🤖 AI Model                           │ │
│ │ Current: Gemini 1.5 Pro               │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ ┌─────────────────────────────────────┐ │
│ │ 💬 Dialogue Style                     │ │
│ │ Current: Professional                 │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ [← Back]                                │
└─────────────────────────────────────────┘
```

### 5.4 Button and Command Mapping

| Command (Hidden/Reserved) | Button (User Visible) |
| ------------------------- | --------------------- |
| `/start`                  | Auto-triggered        |
| `/new`                    | 🆕 New Chat           |
| `/status`                 | 📊 Status             |
| `/help`                   | ❓ Help               |

---

## 6. Action Unified Processing Mechanism

### 6.1 Design Goals

Commands and button callbacks use unified processing to avoid duplicate logic and facilitate multi-platform expansion.

### 6.2 Action Classification

| Type              | Description                               | Handler               |
| ----------------- | ----------------------------------------- | --------------------- |
| **Platform Action**| Platform-specific operations (auth, etc.) | Handled inside Plugin |
| **System Action** | Platform-independent system operations    | Gateway ActionHandler |
| **Chat Action**   | Messages requiring Agent processing       | AgentRouter → Agent   |

```
User Input
    │
    ├─→ Platform Action → Handled inside Plugin (Does not enter Gateway)
    │       Example: Telegram pairing, Slack OAuth, Discord invite
    │
    ├─→ System Action → Gateway ActionHandler → Unified Processing
    │       Example: Session management, settings, help
    │
    └─→ Chat Action → AgentRouter → Gemini/ACP/Codex
```

### 6.3 System Action List (Platform Independent)

| Category         | Action                  | Description                |
| ---------------- | ----------------------- | -------------------------- |
| **Session Mgmt** | `session.new`           | Create new session         |
|                  | `session.status`        | View current status        |
|                  | `session.list`          | Session list (extension)   |
|                  | `session.switch`        | Switch session (extension) |
| **Settings**     | `settings.show`         | Show settings menu         |
|                  | `settings.model.list`   | Show model list            |
|                  | `settings.model.select` | Select model               |
|                  | `settings.agent.select` | Switch Agent (extension)   |
| **Help Info**    | `help.show`             | Show help                  |
| **Navigation**   | `nav.back`              | Go back                    |
|                  | `nav.cancel`            | Cancel current operation   |

### 6.4 Platform Action Examples (Implemented by Plugins)

| Platform     | Action            | Description                |
| ------------ | ----------------- | -------------------------- |
| **Telegram** | `pairing.show`    | Show pairing code          |
|              | `pairing.refresh` | Refresh pairing code       |
| **Slack**    | `oauth.start`     | Initiate OAuth auth        |
|              | `oauth.callback`  | Handle OAuth callback      |
| **Discord**  | `invite.generate` | Generate invite link       |

> **Note**: Platform Actions are handled internally by each Plugin and do not go through Gateway ActionHandler.

### 6.5 Chat Action List

| Category         | Action            | Description          | Routed To            |
| ---------------- | ----------------- | -------------------- | -------------------- |
| **Send Message** | `chat.send`       | User sends new msg   | Current session Agent|
| **Message Ops**  | `chat.regenerate` | Regenerate reply     | Current session Agent|
|                  | `chat.continue`   | Continue generation  | Current session Agent|
|                  | `chat.stop`       | Stop generation      | Current session Agent|

### 6.6 Action Data Structure

```
UnifiedAction {
  action: string          // Action type
  params?: object         // Optional parameters
  context: {
    platform: string      // Source platform
    userId: string        // User ID
    chatId: string        // Chat ID
    messageId?: string    // Trigger message ID
    sessionId?: string    // Current session ID
  }
}
```

### 6.7 Button Callback Data Format

```
Format: action:param1=value1,param2=value2

Examples:
• "session.new"
• "settings.model.select:id=gemini-pro"
• "chat.regenerate:msg=abc123"
```

### 6.8 Unified Response Format

```
ActionResponse {
  text?: string                    // Text content
  parseMode?: 'plain' | 'markdown' // Parse mode
  buttons?: ActionButton[][]       // Inline buttons
  keyboard?: ActionButton[][]      // Reply Keyboard
  behavior: 'send' | 'edit' | 'answer'  // Response behavior
  toast?: string                   // Toast prompt
}
```

---

## 7. Session Management

### 7.1 Relationship between Session and Agent

```
Session {
  id: string              // Session ID
  platform: string        // Source platform
  userId: string          // User ID
  chatId: string          // Chat ID

  // Agent configuration
  agentType: string       // gemini / acp / codex
  agentConfig: {
    modelId?: string      // Model ID
  }

  // Session state
  status: string          // active / idle / error
  context: object         // Agent session context

  // Metadata
  createdAt: number
  lastActiveAt: number
}
```

### 7.2 MVP Phase Session Strategy

| Item           | MVP Implementation                |
| -------------- | --------------------------------- |
| Session Mode   | Single active session             |
| New Session    | Click 🆕 button clears context    |
| Session Store  | Independent from AionUi GUI sess. |
| Agent          | Fixed Gemini                      |
| Model          | Uses AionUi default config        |

### 7.3 Future Extensions

| Item           | Extension Content                               |
| -------------- | ----------------------------------------------- |
| Multi-session  | Support `session.list` / `session.switch`       |
| Agent Switch   | Support `settings.agent.select`                 |
| Model Switch   | Support dynamic model selection                 |
| Session Sync   | Telegram session linked to AionUi session       |

---

## 8. Message Stream Processing Architecture

### 8.1 Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│                    Agent Worker (Gemini/ACP/Codex)             │
│                    (Agent Worker Process)                      │
├────────────────────────────────────────────────────────────────┤
│  Send message event to IPC Bridge                              │
└───────────────────────────┬───────────────────────���────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────────┐
│                    ChannelEventBus                             │
│                    (Global Event Bus - Singleton)              │
├────────────────────────────────────────────────────────────────┤
│  emitAgentMessage(conversationId, data)                        │
│  onAgentMessage(handler) → () => void (cleanup)                │
│                                                                │
│  Event Type: 'channel.agent.message'                           │
│  Data Structure: IAgentMessageEvent { ...IResponseMessage, conv_id } │
└───────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
┌───────────────────��────────────────────────────────────────────┐
│                    ChannelMessageService                       │
│                    (Message Service - Singleton)               │
├────────────────────────────────────────────────────────────────┤
│  initialize() {                                                │
│    // Register global event listener on service init           │
│    channelEventBus.onAgentMessage(this.handleAgentMessage);    │
│  }                                                             │
│                                                                │
│  handleAgentMessage(event) {                                   │
│    // Handle special events: start, finish, error              │
│    // Use transformMessage + composeMessage to merge messages  │
│    // Callback notification: callback(TMessage, isInsert)      │
│  }                                                             │
│                                                                │
│  sendMessage(sessionId, conversationId, text, callback) {      │
│    // Only send message, do not handle listening               │
│    // Call Agent Task via WorkerManage                         │
│  }                                                             │
│                                                                │
│  Internal State:                                               │
│    activeStreams: Map<conversationId, IStreamState>            │
│    messageListMap: Map<conversationId, TMessage[]>             │
└───────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────────┐
│                    ActionExecutor                              │
│                    (Business Executor)                         │
├────────────────────────────────────────────────────────────────┤
│  handleChatMessage(context, text) {                            │
│    messageService.sendMessage(                                 │
│      sessionId, conversationId, text,                           │
│      (message: TMessage, isInsert: boolean) => {               │
│        const outgoing = convertTMessageToOutgoing(message, platform); │
│        if (isInsert) context.sendMessage(outgoing);            │
│        else context.editMessage(msgId, outgoing);              │
│      }                                                         │
│    );                                                          │
│  }                                                             │
│                                                                │
│  convertTMessageToOutgoing(message, platform) {                │
│    // TMessage → IUnifiedOutgoingMessage                       │
│    // Format text based on platform (HTML/Markdown)            │
│    // text → display content                                   │
│    // tips → prompt with icon                                  │
│    // tool_group → tool status list                            │
│  }                                                             │
└───────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────────┐
│                    Plugin (Telegram/Lark)                      │
│                    (Platform Plugin)                           │
├────────────────────────────────────────────────────────────────┤
│  sendMessage(chatId, message: IUnifiedOutgoingMessage)         │
│  editMessage(chatId, messageId, message: IUnifiedOutgoingMessage)│
└────────────────────────────────────────────────────────────────┘
```

### 8.2 Event Type Handling

| Event Type          | Source         | Handling Method                              |
| ------------------- | -------------- | -------------------------------------------- |
| `start`             | Agent starts   | Reset message list                           |
| `content`           | Stream chunk   | transformMessage → composeMessage → callback |
| `tool_group`        | Tool status    | Merge into existing tool_group or append     |
| `finish`/`finished` | Response done  | resolve promise, cleanup state               |
| `error`             | Error occurred | reject promise, cleanup state                |
| `thought`           | Thinking process| Ignore (transformMessage returns undefined) |

### 8.3 Message Merge Strategy (composeMessage)

| Message Type | Merge Rule                           |
| ------------ | ------------------------------------ |
| `text`       | Accumulate content for same msg_id, append for different |
| `tool_group` | Merge tool status updates by callId  |
| `tool_call`  | Merge by callId                      |
| `tips`       | Directly append                      |

### 8.4 Message Callback Parameters

```typescript
type StreamCallback = (chunk: TMessage, isInsert: boolean) => void;

// isInsert = true:  New message, call sendMessage to send new message
// isInsert = false: Update message, call editMessage to edit existing message
```

### 8.5 Throttle Control

| Parameter          | Value  | Description                           |
| ------------------ | ------ | ------------------------------------- |
| UPDATE_THROTTLE_MS | 500ms  | Minimum interval for message edit     |
| Send new message   | Unlimited| Send immediately when isInsert=true |
| Edit message       | Throttle | Apply throttle when isInsert=false  |

- [ ] Use existing Context: **\*\*\*\***\_\_\_\_**\*\*\*\***
- [ ] Need new Context: **\*\*\*\***\_\_\_\_**\*\*\*\***
- [ ] Only internal component state (useState/useReducer)
- [ ] Needs persistent storage

### 8.6 Key Design Principles

1. **Separation of event listening and message sending**
   - Event listening is done during service initialization (`initialize()`)
   - `sendMessage()` is only responsible for sending messages, not handling listening
2. **Global event bus decoupling**
   - `ChannelMessageService` does not directly interact with Agent Task
   - Decoupled via `ChannelEventBus` global event bus
3. **Unified message format**
   - Internally use `TMessage` unified message format
   - Convert to `IUnifiedOutgoingMessage` upon output

---

## 9. Agent Interface Specification

### 8.1 Capabilities Each Agent Must Implement

| Capability      | Description                        |
| --------------- | ---------------------------------- |
| `sendMessage`   | Send message and get response      |
| `streamMessage` | Stream message                     |
| `regenerate`    | Regenerate previous reply          |
| `continue`      | Continue generation                |
| `stop`          | Stop current generation            |
| `getContext`    | Get session context                |
| `clearContext`  | Clear session context              |

### 8.2 Agent Response Format

```
AgentResponse {
  type: 'text' | 'stream_start' | 'stream_chunk' | 'stream_end' | 'error'
  text?: string
  chunk?: string
  error?: { code: string, message: string }
  metadata?: {
    model?: string
    tokensUsed?: number
    duration?: number
  }
  suggestedActions?: ActionButton[]
}
```

---

## 9. File Structure (Actual Implementation)

```
src/channels/
├── core/                          # Core Modules
│   ├── ChannelManager.ts          # Unified Manager (Singleton)
│   └── SessionManager.ts          # Session Management
│
├── gateway/                       # Gateway Layer
│   ├── PluginManager.ts           # Plugin Lifecycle Management
│   └── ActionExecutor.ts          # Action Executor (Routing, Message Handling)
│
├── actions/                       # Action Processing (Platform Independent)
│   ├── types.ts                   # Action/Response Type Definitions
│   ├── SystemActions.ts          # System Actions (Session, Settings, Help)
│   ├── ChatActions.ts            # Chat Actions (Send, Regenerate, etc.)
│   └── PlatformActions.ts        # Platform Actions (Pairing, etc.)
│
├── agent/                         # Agent Integration
│   ├── ChannelEventBus.ts        # Global Event Bus
│   └── ChannelMessageService.ts  # Message Stream Processing Service
│
├── pairing/                       # Pairing Service
│   └── PairingService.ts         # Pairing Code Generation & Validation (Platform Indep.)
│
├── plugins/                       # Plugins Directory
│   ├── BasePlugin.ts              # Abstract Base Class for Plugins
│   ├── telegram/
│   │   ├── TelegramPlugin.ts      # Telegram Plugin
│   │   ├── TelegramAdapter.ts     # Message Adapter
│   │   └── TelegramKeyboards.ts   # Keyboard Components
│   └── lark/
│       ├── LarkPlugin.ts          # Lark Plugin
│       ├── LarkAdapter.ts         # Message Adapter
│       └── LarkCards.ts           # Card Components
│
├── utils/                         # Utility Functions
│   └── credentialCrypto.ts        # Credential Encryption
│
└── types.ts                       # Type Definitions
```

---

## 10. Database Design

| Table Name                | Purpose                                  |
| ------------------------- | ---------------------------------------- |
| `assistant_plugins`       | Plugin configurations (Token, mode, etc.)|
| `assistant_users`         | Authorized users list                    |
| `assistant_sessions`      | User session associations                |
| `assistant_pairing_codes` | Pending pairing requests                 |

---

## 11. External Dependencies

| Dependency                | Purpose               | Description                   |
| ------------------------- | --------------------- | ----------------------------- |
| `grammy`                  | Telegram Bot          | Used by Clawdbot, elegant API |
| `@larksuiteoapi/node-sdk` | Lark/Feishu Bot       | Official SDK                  |
| `@slack/bolt`             | Slack Bot (to be impl)| Official SDK                  |
| `discord.js`              | Discord Bot (to be impl)| Official SDK                  |

---

## 12. Implementation Status

### 12.1 Implemented Features

#### Telegram

- [x] Bot Token configuration and verification
- [x] Bot start/stop control (Polling mode, automatic reconnection)
- [x] Pairing code generation and local approval flow
- [x] Authorized user management
- [x] Button interaction (Reply Keyboard + Inline Keyboard)
- [x] Chat with Gemini/ACP/Codex Agent
- [x] New session feature
- [x] Stream message response (editMessage update)
- [x] Tool confirmation interaction
- [x] Error recovery mechanism

#### Lark/Feishu

- [x] App ID/Secret configuration and verification
- [x] Bot start/stop control (WebSocket long connection)
- [x] Pairing code generation and local approval flow
- [x] Authorized user management
- [x] Card interaction (buttons, confirmation, etc.)
- [x] Chat with Gemini/ACP/Codex Agent
- [x] New session feature
- [x] Stream message response (updateMessage update)
- [x] Tool confirmation interaction (Card format)
- [x] Event deduplication mechanism (5-minute cache)
- [x] HTML to Lark Markdown

#### Core Features

- [x] ChannelManager unified management
- [x] PluginManager plugin lifecycle management
- [x] SessionManager session management
- [x] PairingService pairing service
- [x] ActionExecutor Action routing and execution
- [x] ChannelMessageService message stream processing
- [x] ChannelEventBus global event bus
- [x] Credential encrypted storage
- [x] Multi-platform unified message format

### 12.2 Security Acceptance

- [x] Pairing code 10-minute expiration
- [x] Must be approved locally in AionUi
- [x] Unauthorized users cannot use
- [x] Token/Credential encrypted storage
- [ ] Rate limiting (to be implemented)

### 12.3 Compatibility

- [x] Runs normally on macOS
- [x] Runs normally on Windows
- [x] Multi-language support (i18n)

---

## 13. Future Extension Roadmap

| Phase       | Content                     | Status       |
| ----------- | --------------------------- | ------------ |
| **Phase 1** | Telegram + Lark Integration | ✅ Completed |
| **Phase 2** | Multi-session, Switch Sess. | 🔄 Pending   |
| **Phase 3** | Agent Switch (Needs UI)     | 🔄 Partial   |
| **Phase 4** | Model Dynamic Switch        | 🔄 Pending   |
| **Phase 5** | Slack Platform Integration  | 🔄 Pending   |
| **Phase 6** | Discord Platform Integration| 🔄 Pending   |
| **Phase 7** | Rate Limiting               | 🔄 Pending   |
| **Phase 8** | Sync Session with AionUi    | 🔄 Pending   |
| **Phase 9** | Headless Independent Service| 🔄 Pending   |

---

## Template Maintenance

- **Creation Date**: 2025-01-27
- **Last Updated**: 2026-02-03
- **Applicable Version**: AionUi v1.7.8+
- **Maintainer**: Project Team

---

## Appendix: Key Implementation Details

### A.1 ChannelManager Initialization Flow

```typescript
1. ChannelManager.getInstance().initialize()
   ├─ Initialize PluginManager
   ├─ Initialize SessionManager
   ├─ Initialize PairingService
   ├─ Initialize ActionExecutor
   └─ Initialize ChannelMessageService

2. Load plugin configurations from database
3. Call initialize() and start() for each enabled plugin
```

### A.2 Message Processing Flow

```typescript
1. Plugin receives platform message
   └─ toUnifiedIncomingMessage() -> unifiedMsg

2. PluginManager.handleMessage(unifiedMsg)
   └─ ActionExecutor.handleMessage(unifiedMsg)

3. ActionExecutor routes message
   ├─ System Actions -> handleSystemAction()
   └─ Chat Actions -> handleChatMessage()

4. handleChatMessage(msg)
   └─ messageService.sendMessage(..., callback)

5. Agent returns response
   ├─ emitAgentMessage -> messageService.handleAgentMessage()
   └─ trigger callback -> ActionExecutor formatting -> Plugin send/edit
```
