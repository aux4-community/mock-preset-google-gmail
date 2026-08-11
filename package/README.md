# community/mock-preset-google-gmail

Gmail preset for [aux4/mock](https://hub.aux4.io/r/public/packages/aux4/mock).

One command replaces the Gmail API with a local mock: the message list, a message by id, send,
labels, and `batchModify`. Point a Gmail client at the mock with `--apiUrl` and it behaves like
the real service, offline.

## Installation

```bash
aux4 aux4 pkger install community/mock-preset-google-gmail
```

Installing pulls in `community/mock-preset-google-auth`, which supplies the credentials every
Google API call needs.

## Usage

Start a mock, apply the credentials, then apply the Gmail endpoints:

```bash
aux4 mock start --port 18901
aux4 mock preset google auth --port 18901
aux4 mock preset google gmail --port 18901
```

A Gmail client then runs entirely against the mock:

```bash
aux4 google gmail list --tokenFile google-token.json --apiUrl http://127.0.0.1:18901/api
```

```json
{"messages":[{"id":"18f8a9b0c1d2e3f4","threadId":"18f8a9b0c1d2e3f4"}],"resultSizeEstimate":1}
```

### Endpoints

| Method | Path | Returns |
|---|---|---|
| `GET` | `/users/me/messages` | a one-message list |
| `GET` | `/users/me/messages/{id}` | that message, echoing the id requested |
| `POST` | `/users/me/messages/send` | the sent message, labelled `SENT` |
| `POST` | `/users/me/messages/batchModify` | `204`, no body |
| `GET` | `/users/me/labels` | `INBOX`, `SENT`, and a user label `Work` |

A get by id echoes back whatever id was asked for, so a test that lists then fetches gets a
message whose id matches the one it followed.

### Shaping the responses

```bash
aux4 mock preset google gmail --port 18901 \
  --subject "Invoice #42" --from billing@corp.io --snippet "Your invoice is attached"
```

| Option | Description | Default |
|---|---|---|
| `--port` | Port of the running mock server | `7070` |
| `--name` | Stable handle of the server, instead of `--port` | |
| `--stateDir` | Explicit state directory | |
| `--messageId` | Id in the list and send responses | `18f8a9b0c1d2e3f4` |
| `--threadId` | Thread id in the message bodies | `18f8a9b0c1d2e3f4` |
| `--subject` | `Subject` header of the fetched message | `Weekly update` |
| `--from` | `From` header of the fetched message | `sally@example.com` |
| `--snippet` | Preview text of the fetched message | `Hi there, this is a preview` |

For a response the flags do not reach, replace a whole body: `--messagesBody`, `--messageBody`,
`--sentBody`, and `--labelsBody` each take raw JSON. Use `--messagesBody` to return several
messages, or an empty `messages` array to test an empty inbox.

### Layering your own stubs

Apply the preset, then override any single endpoint with `aux4 mock stub` — the later stub wins:

```bash
aux4 mock preset google gmail --port 18901
aux4 mock stub --port 18901 --method POST --path /users/me/messages/send --status 429 \
  --body '{"error":{"code":429,"message":"User-rate limit exceeded","status":"RESOURCE_EXHAUSTED"}}'
```

That is how to test an error path: take the whole API from the preset and bend the one endpoint
under test.

## See also

- [community/mock-preset-google-auth](https://hub.aux4.io/r/public/packages/community/mock-preset-google-auth) — the credentials every Google preset needs
- [community/mock-preset-google-calendar](https://hub.aux4.io/r/public/packages/community/mock-preset-google-calendar) — the Calendar API
- [aux4/mock](https://hub.aux4.io/r/public/packages/aux4/mock) — the mock server itself
