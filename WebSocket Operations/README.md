# WebSocket Operations

## Connect to Vault events

### `connectToWebsocket()`

Opens the general Vault WebSocket configured by `VAULT_WS_URL`. The SDK sends
the signed credentials in the WebSocket handshake subprotocol rather than in
the URL.

```javascript
await vault.connectToWebsocket();

vault.on('message', (payload) => {
  console.log('Vault event:', payload);
});

vault.on('websocket_close', ({ code, reason }) => {
  console.log('Vault WebSocket closed:', code, reason);
});

vault.on('stream_error', (error) => {
  console.error('Vault WebSocket error:', error);
});
```

`VAULT_WS_URL` is required for this method. Calling it while another general
Vault socket is open closes the previous socket first. The method resolves when
the socket opens; later server messages and transport failures are delivered as
events.

**Errors:**

- `MISSING_CONFIG`: `VAULT_WS_URL` was not configured.
- `WEBSOCKET_CLOSED`: The socket closed before it opened.
- `WEBSOCKET_ERROR`: The connection failed or a later socket error occurred.

This socket is separate from the bot chat socket. Use
**[Bot Chat Operations](../Bot%20Chat%20Operations/README.md)** for
`connectToBotChat()`, bot joins, messages, typing indicators, and disconnect.
