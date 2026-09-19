# Bot Session Operations

This section covers retrieving chat sessions and message history for a bot.
For the live WebSocket conversation, see **[Bot Chat Operations](../Bot%20Chat%20Operations/README.md)**.

## Get Bot Sessions

### `getBotSessions(vaultId, botId, sessionId)`

Retrieves chat sessions for a bot, or the messages in one specific session.
The optional `sessionId` changes the response from a session list to that
session's message history.

**Parameters:**

- `vaultId` (String): The ID of the vault that owns the bot. Required.
- `botId` (String): The bot ID. Required.
- `sessionId` (String, optional): A session ID to retrieve. When omitted, the
  method lists all sessions for the bot. Blank whitespace is treated as omitted.

The SDK trims `botId` and `sessionId` before making the request. A session must
belong to both the specified bot and the authenticated vault user.

**List sessions:**

```javascript
try {
  const result = await vault.getBotSessions('your-vault-id', 'bot-id');

  // result.data is an array, newest activity first.
  for (const session of result.data) {
    console.log(session.id, session.title, session.updatedAt);
  }
} catch (error) {
  console.error('Could not fetch sessions:', error.code, error.message);
}
```

The session objects are returned by the backend and include fields such as
`id`, `botId`, `userId`, `title`, `createdAt`, and `updatedAt`.

**Get one session's messages:**

```javascript
try {
  const result = await vault.getBotSessions(
    'your-vault-id',
    'bot-id',
    'session-id'
  );

  // result.data is an array, in chronological order.
  for (const message of result.data) {
    console.log(message.role, message.content, message.createdAt);
  }
} catch (error) {
  console.error('Could not fetch session messages:', error.code, error.message);
}
```

Message objects include fields such as `id`, `sessionId`, `role`, `content`,
`sources`, and `createdAt`. The order is oldest first so the array can be
rendered directly as a conversation.

**Response:**

The SDK returns the backend response object:

- When `sessionId` is omitted: `data` is an array of sessions and the message is
  `"Bot sessions fetched successfully"`.
- When `sessionId` is supplied: `data` is an array of messages and the message is
  `"Session messages fetched successfully"`.
- `success` is `true` for a successful request.

This method does not return a WebSocket connection or live updates. Use
`connectToBotChat()` and the events in **[Bot Chat Operations](../Bot%20Chat%20Operations/README.md)**
for live messages.

## Delete Bot Sessions

### `deleteBotSessions(vaultId, botId, sessionIds)`

Permanently deletes one or more saved chat sessions and their messages for a
bot. This does not disconnect an active WebSocket chat or affect the bot itself.

**Parameters:**

- `vaultId` (String): The vault that owns the bot. Required.
- `botId` (String): The bot ID whose sessions are being deleted. Required.
- `sessionIds` (String | Array<String>): One session ID or an array of session
  IDs. Required. The SDK trims IDs, removes duplicates, and ignores blank or
  non-string array entries.

Only sessions belonging to both the specified bot and authenticated vault user
are deleted. If the request includes a mixture of valid and invalid IDs, valid
sessions are deleted and invalid IDs are ignored.

**Example:**

```javascript
try {
  const result = await vault.deleteBotSessions(
    'your-vault-id',
    'bot-id',
    ['session-id-1', 'session-id-2']
  );

  console.log(result.message);
  console.log('Deleted sessions:', result.data.deletedCount);
} catch (error) {
  console.error('Could not delete sessions:', error.code, error.message);
}
```

A single session ID is also accepted:

```javascript
await vault.deleteBotSessions('your-vault-id', 'bot-id', 'session-id');
```

**Response:**

Returns the standard response:

```javascript
{
  success: true,
  message: '2 session(s) deleted successfully',
  data: { deletedCount: 2 }
}
```

`data.deletedCount` reports how many valid sessions were actually removed. The
operation is permanent; export sessions first if you need an archive.

**Errors:**

- `INVALID_PARAMETER`: `vaultId` or `botId` is missing or invalid, or no usable
  session IDs remain after normalization.
- `NOT_FOUND`: The bot is missing or not owned by the vault user, or none of the
  supplied session IDs belongs to that bot and user. The backend message is
  `"No valid sessions found"`.
- `UNAUTHORIZED` / `FORBIDDEN`: Authentication or SDK access is rejected.
- `NETWORK_ERROR`, `SERVER_ERROR`, or another API error: The deletion request
  could not be completed. See **[Error Handling](../Error%20Handling/README.md)**.

## Export Bot Sessions

### `exportBotSessions(vaultId, botId, sessionIds, saveOption, targetBotId)`

Exports one or more saved chat sessions as a combined plain-text file. The
export includes the session title, message count, timestamps, and each message
in conversation order.

**Parameters:**

- `vaultId` (String): The vault that owns the source bot. Required.
- `botId` (String): The source bot whose sessions are being exported. Required.
- `sessionIds` (String | Array<String>): One session ID or an array of session
  IDs. Required. The SDK trims IDs, removes duplicates, and ignores blank or
  non-string array entries.
- `saveOption` (String): Where to save the generated text file. Must be exactly
  `'drive'` or `'brain'`.
- `targetBotId` (String, optional): The bot whose knowledge set receives the
  export when `saveOption` is `'brain'`. If omitted for a brain export, the
  source `botId` is used. This option is ignored for `'drive'` exports.

The selected sessions must belong to the source bot and authenticated vault
user. At least one selected session must contain messages.

**Save to the drive:**

```javascript
try {
  const result = await vault.exportBotSessions(
    'your-vault-id',
    'bot-id',
    ['session-id-1', 'session-id-2'],
    'drive'
  );

  console.log(result.message);
  console.log('Created file:', result.data.fileName);
  console.log('Saved to drive:', result.data.savedToDrive);
} catch (error) {
  console.error('Could not export sessions:', error.code, error.message);
}
```

**Save to a bot's knowledge set:**

```javascript
try {
  const result = await vault.exportBotSessions(
    'your-vault-id',
    'source-bot-id',
    'session-id',
    'brain',
    'target-bot-id' // Optional; defaults to source-bot-id
  );

  console.log('Created file:', result.data.fileName);
  console.log('Ingested by bot:', result.data.targetBotId);
  console.log('Ingested:', result.data.ingestedToBrain);
} catch (error) {
  console.error('Could not export to bot knowledge:', error.code, error.message);
}
```

When saving to `drive`, the generated file is uploaded to the drive root and is
not synchronized into bot knowledge. When saving to `brain`, it is uploaded to
the target bot's dedicated folder and ingested by that bot. A target bot must be
owned by the same vault user.

**Response:**

Returns the standard response:

```javascript
{
  success: true,
  message: 'Chat session(s) exported successfully (drive)',
  data: {
    fileName: 'chat_support_bot_2_sessions_20260910...txt',
    savedTo: 'drive',
    sessionCount: 2,
    savedToDrive: true
  }
}
```

For `saveOption: 'brain'`, `data` also contains `ingestedToBrain: true` and
`targetBotId`. The generated file name and export timestamp are assigned by the
backend; do not rely on an exact timestamp suffix.

**Errors:**

- `INVALID_PARAMETER`: A required argument is missing or invalid, no usable
  session IDs remain, or `saveOption` is not exactly `'drive'` or `'brain'`.
- `NOT_FOUND`: The source bot is not owned by the vault user, no selected session
  belongs to that bot and user, or the requested target bot does not exist or is
  not owned by the user.
- `BAD_REQUEST`: The selected sessions contain no messages, or the export could
  not be saved or ingested.
- `UNAUTHORIZED` / `FORBIDDEN`: Authentication or SDK access is rejected.

Session export does not delete or modify the original sessions. See
**[Error Handling](../Error%20Handling/README.md)** for other API and network errors.

**Errors:**

- `INVALID_PARAMETER`: `vaultId`, `botId`, or a supplied `sessionId` is missing
  or fails SDK validation.
- `NOT_FOUND`: The bot is missing or not owned by the vault user, or the session
  does not exist, belongs to another bot, or belongs to another user. The
  backend message for the latter is `"Chat session not found"`.
- `UNAUTHORIZED` / `FORBIDDEN`: Authentication or SDK access is rejected.
- `NETWORK_ERROR`, `SERVER_ERROR`, or another API error: The request could not
  be completed. See **[Error Handling](../Error%20Handling/README.md)**.
