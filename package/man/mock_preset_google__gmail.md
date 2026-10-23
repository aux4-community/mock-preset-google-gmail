#### Description

The `gmail` command registers the Gmail API on a running `aux4/mock` server, so a Gmail client can
run against a local mock instead of Google. It stubs the message list, a message by id, send,
trash, labels, and `batchModify` in one call.

It registers credentials for nothing — run `aux4 mock preset google auth` first. That command
writes the token file and stubs the OAuth endpoints; this one only serves the API.

The message endpoint is a path template, so a get by any id matches, and the body echoes back the
id that was asked for. A test that lists messages and then fetches one therefore gets a message
whose id matches the one it followed, without pinning `--messageId` to the same value.

`--subject`, `--from`, and `--snippet` shape the fetched message. When a flag does not reach the
field you need, pass the whole body instead with `--messagesBody`, `--messageBody`, `--sentBody`,
or `--labelsBody`.

The preset is a base layer. Apply it, then call `aux4 mock stub` for any endpoint you want to
behave differently — a later stub for the same method and path replaces the preset's.

#### Stateful effects

`send` and `trash` reprogram the GET-by-id endpoint for the affected message when they serve a
response (using `aux4/mock`'s `--thenFile`): a `send` makes a later `GET` of the fixed
`--messageId` return the sent message's `to`/`subject`/`body` (best-effort — a real client's
base64 `raw` payload is not decoded; send structured fields in a test to see them reflected); a
`trash` makes a later `GET` of that message id show `labelIds: ["TRASH"]`. `batchModify` is
unaffected — it targets messages by an `ids` array, which this mock's templating cannot address
generically; use `trash` when the test needs the state-aware path. Requires `aux4/mock` 0.1.0 or
later.

#### Usage

```bash
aux4 mock preset google gmail [--port <port>] [--subject <text>] [--from <address>]
```

--port          Port of the running mock server, which derives the state directory (default: `7070`)
--name          Stable handle of the server, used instead of `--port`
--stateDir      Explicit state directory, taking precedence over `--port` and `--name`
--messageId     Id in the list and send responses (default: `18f8a9b0c1d2e3f4`)
--threadId      Thread id carried by the message bodies (default: `18f8a9b0c1d2e3f4`)
--subject       `Subject` header of the fetched message (default: `Weekly update`)
--from          `From` header of the fetched message (default: `sally@example.com`)
--snippet       Preview text of the fetched message (default: `Hi there, this is a preview`)
--messagesBody  Raw JSON body for the message list
--messageBody   Raw JSON body for a message by id
--sentBody      Raw JSON body for a send
--trashBody     Raw JSON body for a trash call
--labelsBody    Raw JSON body for the label list

#### Example

```bash
aux4 mock start --port 18901
aux4 mock preset google auth --port 18901
aux4 mock preset google gmail --port 18901 --subject "Invoice #42" --from billing@corp.io
```

```text
stub GET /users/me/messages -> 200
stub POST /users/me/messages/send -> 200
stub POST /users/me/messages/batchModify -> 204
stub GET /users/me/labels -> 200
stub GET /users/me/messages/{id} -> 200
stub POST /users/me/messages/{id}/trash -> 200
```

A Gmail client pointed at the mock then returns the stubbed message:

```bash
aux4 google gmail get 18f8a9b0c1d2e3f4 --tokenFile google-token.json --apiUrl http://127.0.0.1:18901/api
```

```text
{"id":"18f8a9b0c1d2e3f4","threadId":"18f8a9b0c1d2e3f4","labelIds":["INBOX"],"snippet":"Hi there, this is a preview","payload":{"headers":[{"name":"Subject","value":"Invoice #42"},{"name":"From","value":"billing@corp.io"}]}}
```
