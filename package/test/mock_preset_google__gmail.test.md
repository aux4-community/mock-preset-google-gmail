# community/mock-preset-google-gmail

Proves that installing this package makes `aux4 mock preset google gmail` available through aux4's
profile merge — the `aux4/mock` core knows nothing about Gmail — and that the stubs it applies
answer every endpoint a Gmail client calls.

The server is started inside the first test's `execute` block (not `beforeAll`, which the harness
would kill) and torn down in `afterAll` on a unique port. It is a detached process, so it survives
across the sibling `##` scenarios; each override scenario clears the stub store first. All checks use
plain `curl`, with no external network.

```afterAll
aux4 mock stop --port 18903 2>/dev/null
pkill -f "18903" 2>/dev/null
true
```

## the gmail preset on a running mock

### should start the server and apply the gmail preset

```execute
aux4 mock start --port 18903
sleep 1
aux4 mock preset google gmail --port 18903
```

```expect:partial
stub GET /users/me/messages -> 200
```

### should list messages

```execute
curl -s http://localhost:18903/api/users/me/messages
```

```expect
{"messages":[{"id":"18f8a9b0c1d2e3f4","threadId":"18f8a9b0c1d2e3f4"}],"resultSizeEstimate":1}
```

### should return a message by id

```execute
curl -s http://localhost:18903/api/users/me/messages/18f8a9b0c1d2e3f4
```

```expect:partial
"value":"Weekly update"
```

### should echo back whichever id was requested

A test that lists and then fetches gets a message whose id matches the one it followed.

```execute
curl -s http://localhost:18903/api/users/me/messages/some-other-id
```

```expect:partial
"id":"some-other-id"
```

### should send a message

```execute
curl -s -X POST http://localhost:18903/api/users/me/messages/send -H 'Content-Type: application/json' -d '{"raw":"..."}'
```

```expect
{"id":"msg-sent-0001","threadId":"18f8a9b0c1d2e3f4","labelIds":["SENT"]}
```

### should keep the thread a reply is sent on

```execute
curl -s -X POST http://localhost:18903/api/users/me/messages/send -H 'Content-Type: application/json' -d '{"raw":"...","threadId":"thr-99"}'
```

```expect
{"id":"msg-sent-0001","threadId":"thr-99","labelIds":["SENT"]}
```

### should not let the send path fall through to the message-by-id stub

`send` is a literal path that sits where an id would go, so it has to win over the template.

```execute
curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:18903/api/users/me/messages/send -d '{}'
```

```expect
200
```

### should list labels

```execute
curl -s http://localhost:18903/api/users/me/labels
```

```expect:partial
"name":"Work"
```

### should accept a batchModify with no content

```execute
curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:18903/api/users/me/messages/batchModify -H 'Content-Type: application/json' -d '{"ids":["18f8a9b0c1d2e3f4"],"addLabelIds":["TRASH"]}'
```

```expect
204
```

## stateful effects: GET reflects an earlier send/trash

### should re-apply a fresh preset

```execute
aux4 mock reset --port 18903 --stubs true
aux4 mock preset google gmail --port 18903
```

```expect:partial
stub POST /users/me/messages/{id}/trash -> 200
```

### should make the sent message retrievable by its returned id

```execute
curl -s -X POST http://localhost:18903/api/users/me/messages/send -H 'Content-Type: application/json' -d '{"to":"sally@aux4.io","subject":"hi","body":"hello sally"}'
curl -s http://localhost:18903/api/users/me/messages/msg-sent-0001
```

```expect
{"id":"msg-sent-0001","threadId":"18f8a9b0c1d2e3f4","labelIds":["SENT"]}{"id":"msg-sent-0001","threadId":"18f8a9b0c1d2e3f4","labelIds":["SENT"],"snippet":"hello sally","payload":{"headers":[{"name":"To","value":"sally@aux4.io"},{"name":"Subject","value":"hi"}]}}
```

### should leave the inbox message unchanged after a send

```execute
curl -s http://localhost:18903/api/users/me/messages/18f8a9b0c1d2e3f4
```

```expect:partial
"labelIds":["INBOX"]
```

### should show the TRASH label on a later GET after trashing the message

```execute
curl -s -X POST http://localhost:18903/api/users/me/messages/18f8a9b0c1d2e3f4/trash
curl -s http://localhost:18903/api/users/me/messages/18f8a9b0c1d2e3f4
```

```expect:partial
"labelIds":["TRASH"]
```

## overrides: name the sender and the subject

### should re-apply the preset with overrides

```execute
aux4 mock reset --port 18903 --stubs true
aux4 mock preset google gmail --port 18903 --subject "Invoice #42" --from billing@corp.io --snippet "Your invoice is attached" --messageId msg99
```

```expect:partial
stub GET /users/me/labels -> 200
```

### should return the overridden headers

```execute
curl -s http://localhost:18903/api/users/me/messages/msg99
```

```expect:partial
"value":"billing@corp.io"
```

### should return the overridden snippet

```execute
curl -s http://localhost:18903/api/users/me/messages/msg99
```

```expect:partial
"snippet":"Your invoice is attached"
```

### should return the overridden id in the list

```execute
curl -s http://localhost:18903/api/users/me/messages
```

```expect:partial
"id":"msg99"
```

## an empty inbox

### should return no messages when the list body is replaced

```execute
aux4 mock reset --port 18903 --stubs true
aux4 mock preset google gmail --port 18903 --messagesBody '{"messages":[],"resultSizeEstimate":0}'
```

```expect:partial
stub GET /users/me/messages -> 200
```

### should serve the empty inbox

```execute
curl -s http://localhost:18903/api/users/me/messages
```

```expect
{"messages":[],"resultSizeEstimate":0}
```

## layering: an error path over the preset

### should apply the preset then override send

A later stub for the same method and path replaces the preset's, which is how an error path is
tested without hand-stubbing the rest of the API.

```execute
aux4 mock reset --port 18903 --stubs true
aux4 mock preset google gmail --port 18903
aux4 mock stub --port 18903 --method POST --path /users/me/messages/send --status 429 --body '{"error":{"code":429,"message":"User-rate limit exceeded","status":"RESOURCE_EXHAUSTED"}}'
```

```expect:partial
stub POST /users/me/messages/send -> 429
```

### should serve the error on the overridden endpoint

```execute
curl -s -w "\n%{http_code}" -X POST http://localhost:18903/api/users/me/messages/send -d '{}'
```

```expect
{"error":{"code":429,"message":"User-rate limit exceeded","status":"RESOURCE_EXHAUSTED"}}
429
```

### should still serve the untouched endpoints from the preset

```execute
curl -s http://localhost:18903/api/users/me/labels
```

```expect:partial
"id":"INBOX"
```
