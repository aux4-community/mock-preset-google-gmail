# Release notes

## 0.0.3

- New `POST /users/me/messages/{id}/trash` endpoint; a later read of the message shows the
  `TRASH` label.
- Stateful send (needs aux4/mock 0.1.0+): a sent message gets its own id (`sentMessageId`,
  default `msg-sent-0001`), keeps the thread it was sent on, and can be read back by id; the
  inbox message is left unchanged.
- Scratch files the preset writes are removed after use.
