# Bot Chat Operations

This section covers live bot chat connections, messages, typing indicators, and
connection cleanup. See **[Bot Operations](../Bot%20Operations/README.md)** for
creating bots and managing their knowledge.

## Connect to Bot Chat

### `connectToBotChat(vaultId, options)`

Opens an authenticated WebSocket connection to the live bot chat service.
Pass `options.botId` to send a join request automatically when the socket opens.
Without it, call `vault.joinBotChat(botId, sessionId)` after connecting.

**Parameters:**

- `vaultId` (String): The vault ID to authenticate for chat. Required.
- `options` (Object, optional): Connection settings. Defaults to `{}`.

| Option | Type | Description |
| --- | --- | --- |
| `token` | String | Existing vault access token for the intended vault user. |
| `launchToken` | String | Existing one-time launch token to redeem when no access token is supplied. |
| `botId` | String | Bot ID to join automatically after the socket opens. |
| `sessionId` | String | Existing chat session ID to resume when auto-joining the bot. Omit for a new conversation. |
| `wsUrl` | String | Explicit WebSocket base URL. Overrides the configured URL. |
| `returnTo`, `clientId`, `adminId`, `sourceUserId` | String | Optional launch context forwarded when the SDK creates a launch token automatically. |

Authentication uses a non-empty `token` first, then redeems `launchToken` if
provided. If no access token is available, the SDK creates and redeems a launch
token using `vaultId` and the configured SDK credentials. Token creation or
redemption errors reject the call.

The socket base URL is chosen from `options.wsUrl`, then `VAULT_WS_URL`, then
`VAULT_BASE_URL`. The SDK sets the path to `/ws/bot-chat`, converts HTTP/HTTPS to
WS/WSS, and adds the access token as a query parameter.

**Example:**

Register event listeners before connecting so they receive the initial events.
Wait for `bot_chat_chat_history` to confirm the bot was joined before sending
a message.

```javascript
vault.on('bot_chat_chat_history', ({ botName, sessionId, history }) => {
  console.log('Joined:', botName, 'Session:', sessionId);
  console.log('Previous messages:', history);
  vault.sendBotChatMessage('What can you help me with?');
});

vault.on('bot_chat_token', ({ token }) => {
  console.log('Reply fragment:', token);
});

vault.on('bot_chat_message_complete', ({ content, sessionId, sources }) => {
  console.log('Reply:', content);
  console.log('Session:', sessionId, 'Sources:', sources);
});

vault.on('bot_chat_error', (payload) => {
  console.error('Chat request failed:', payload.message);
});

vault.on('bot_chat_stream_error', (error) => {
  console.error('Chat connection error:', error.code, error.message);
});

vault.on('bot_chat_close', (event) => {
  console.log('Chat closed:', event.code, event.reason);
});

try {
  await vault.connectToBotChat('your-vault-id', {
    botId: 'bot-id',
    // sessionId: 'existing-session-id', // Optional: resume a conversation
  });
} catch (error) {
  console.error('Could not connect:', error.code, error.message);
}

// When the user finishes chatting, call:
// vault.disconnectBotChat();
```

To use an existing access token, include `token: accessToken` in the options.
The same options can include `sessionId` to resume an existing bot conversation.

**Return value:**

The promise resolves with connection metadata when the socket opens:

- `token` (String): The access token used for the connection.
- `url` (String): The WebSocket URL, including its token query parameter.
- `botId` (String | null): The requested bot ID, or `null` when omitted.
- `sessionId` (String | null): The requested session ID, or `null` when omitted.

This is a plain metadata object, without a `success` / `data` wrapper. It confirms
the socket opened, not that the server accepted the bot join request. A new
session's ID arrives through chat events. Both `token` and `url` contain
credentials; avoid logging the connection metadata.

**Events:**

| Event | Payload / purpose |
| --- | --- |
| `bot_chat_open` | No payload; the socket opened, before the automatic join request. |
| `bot_chat_message` | The complete parsed server message, including `type` and `payload`. |
| `bot_chat_chat_history` | Joined bot details, session ID, and previous messages. |
| `bot_chat_session_info` | Session information, including `sessionId`. |
| `bot_chat_token` | A streamed reply fragment in `payload.token`. |
| `bot_chat_message_complete` | Completed reply with `content`, `sessionId`, and `sources`. |
| `bot_chat_error` | A server-reported chat error, including `message`. |
| `bot_chat_stream_error` | A `VaultError` for socket errors or unreadable server messages. |
| `bot_chat_close` | The WebSocket close event. |

Every server message with a `type` also emits `bot_chat_<type>` with that
message's `payload`.

**Lifecycle and errors:**

Calling this method again closes an existing open or connecting bot chat socket
on the same SDK instance. There is no automatic reconnect. Use
`vault.disconnectBotChat()` to close the connection when finished.

- `INVALID_PARAMETER`: Required arguments fail SDK validation.
- `MISSING_CONFIG`: No base URL is available to build the chat socket URL.
- `BAD_RESPONSE`: Automatic authentication does not return the expected tokens.
  Malformed socket messages emit this code through `bot_chat_stream_error`.
- `WEBSOCKET_CLOSED`: The socket closes before the connection is established.
- `WEBSOCKET_ERROR`: A socket connection error occurs. This also emits
  `bot_chat_stream_error`.

Authentication API and network errors can also reject the call. After the
connection promise resolves, handle failures through the chat events; a
`try...catch` around the connection call does not catch later server errors.
See **[Error Handling](../Error%20Handling/README.md)** for common SDK errors.

## Join Bot Chat

### `joinBotChat(botId, sessionId)`

Sends a request to join a bot on an already-open chat socket. Call
`connectToBotChat(vaultId)` first. Supplying `botId` to `connectToBotChat()`
already performs this step automatically.

**Parameters:**

- `botId` (String): The bot to join. Required. The SDK trims the ID.
- `sessionId` (String, optional): An existing session for this bot to resume.
  Defaults to `null`. When omitted, the server starts a new session when the
  first message is sent.

**Example:**

```javascript
vault.on('bot_chat_chat_history', ({ botName, sessionId, history }) => {
  console.log('Joined:', botName, 'Session:', sessionId);
  console.log('Previous messages:', history);
  // The bot is now ready to receive messages.
});
vault.on('bot_chat_error', (payload) => console.error(payload.message));
vault.on('bot_chat_stream_error', (error) => console.error(error.message));

try {
  await vault.connectToBotChat('your-vault-id');
  vault.joinBotChat('bot-id');
  // To resume instead, use:
  // vault.joinBotChat('bot-id', 'existing-session-id');
} catch (error) {
  console.error('Could not connect or join:', error.code, error.message);
}
```

**Return value and errors:**

Returns `undefined` immediately after sending the request. Wait for
`bot_chat_chat_history` before sending messages. Calling the method again changes
the bot/session context on the same socket.

Invalid arguments throw `INVALID_PARAMETER`. A socket that is not open throws
`WEBSOCKET_NOT_CONNECTED`. Server-side join failures, such as a bot not being
found, arrive through `bot_chat_error`.

## Send a Chat Message

### `sendBotChatMessage(message, history)`

Sends a user message to the currently joined bot. Connect and wait for
`bot_chat_chat_history` before calling this method.

**Parameters:**

- `message` (String): Required, non-empty message text. The SDK trims whitespace.
- `history` (Array<Object>, optional): Previous conversation turns, in
  chronological order. Defaults to `[]`. Each entry contains `role` (`'user'`
  or `'assistant'`) and non-empty string `content`.

The SDK discards history entries with unsupported roles or unusable content and
trims each retained entry's content. Include only prior turns; the new `message`
is sent separately. The backend uses supplied usable history as context, or loads
recent saved session history when none remains.

**Example:**

```javascript
// Register reply listeners before sending. The socket must already have joined a bot.
vault.on('bot_chat_token', ({ token }) => console.log('Reply fragment:', token));
vault.on('bot_chat_session_info', ({ sessionId }) => console.log('Session:', sessionId));
vault.on('bot_chat_message_complete', ({ content }) => console.log('Reply:', content));
vault.on('bot_chat_error', (payload) => console.error(payload.message));

try {
  vault.sendBotChatMessage('How do I upload a file?', [
    { role: 'user', content: 'What can you help with?' },
    { role: 'assistant', content: 'I can help you manage files in your vault.' },
  ]);
  // To use saved session history, omit the second argument.
} catch (error) {
  console.error('Could not send message:', error.code, error.message);
}
```

**Return value and errors:**

Returns `undefined`; the reply arrives through `bot_chat_token` and
`bot_chat_message_complete`. `bot_chat_session_info` provides the session ID.

An invalid or blank `message`, or a non-array `history`, throws
`INVALID_PARAMETER`. A socket that is not open throws
`WEBSOCKET_NOT_CONNECTED`. Server-side failures arrive through `bot_chat_error`;
socket failures arrive through `bot_chat_stream_error`.

## Send a Typing Indicator

### `sendBotChatTyping()`

Sends a typing indicator to the same user's other active chat connections joined
to the same bot. Call it after connecting and joining a bot.

**Parameters:** None.

**Example:**

```javascript
// On another active connection for the same user and bot:
vault.on('bot_chat_typing', ({ userId, isTyping }) => {
  console.log('Typing:', userId, isTyping);
});

// On the connection where the user is entering a message:
try {
  vault.sendBotChatTyping();
} catch (error) {
  console.error('Could not send typing indicator:', error.code, error.message);
}
```

**Return value and errors:**

Returns `undefined`. The SDK sends `{ isTyping: true }`; this method has no
argument for sending a stopped-typing state. The originating connection does
not receive its own indicator. Without a joined bot, the backend does not
broadcast it. A socket that is not open throws `WEBSOCKET_NOT_CONNECTED`.

## Disconnect from Bot Chat

### `disconnectBotChat()`

Closes the current bot chat socket and clears the SDK's connection reference.
Call it when the user leaves chat or when cleaning up the SDK instance.

**Parameters:**

No arguments are required. The implementation also accepts optional
`disconnectBotChat(code, reason)` arguments:

- `code` (Number, optional): WebSocket close code. Defaults to `1000`.
- `reason` (String, optional): Close reason. Defaults to
  `'Bot chat closed by client'`.

**Example:**

```javascript
vault.on('bot_chat_close', (event) => {
  console.log('Chat closed:', event.code, event.reason);
});

vault.disconnectBotChat();
```

**Return value and behavior:**

Returns `undefined` and does nothing when no socket exists. It requests closure
without waiting for the closing handshake; `bot_chat_close` reports the close
event. Custom close arguments must satisfy the WebSocket implementation's
requirements, or its `close()` call may throw.

Disconnecting does not delete saved chat sessions or messages. To resume later,
call `connectToBotChat(vaultId, { botId, sessionId })`. Sending messages or typing
indicators after disconnecting throws `WEBSOCKET_NOT_CONNECTED`.
