# staal-chat

Messages to Slack and Discord channels: a [Staal](https://github.com/laurells/staal)
package that gives programs the `chat.slack.post` and `chat.discord.post` capabilities,
built on Staal's general `net.http` capability. It talks to
[Slack](https://api.slack.com/methods/chat.postMessage) and
[Discord](https://discord.com/developers/docs/resources/message#create-message) (API v10).

## Authority

This is everything the package can reach. `staalc fetch` prints it when the
package is added, and the compiler refuses the package if its code reaches
beyond it.

| Capability | Targets |
| --- | --- |
| `net.http.post` | `https://slack.com/api/chat.postMessage`, `https://discord.com/api/v10/channels/*/messages` |

## Install

Add the package to your project's `staal.json`, pinned to an exact commit:

```json
{
  "dependencies": {
    "chat": {
      "git": "https://github.com/laurells/staal-chat",
      "commit": "<full 40-character commit id>"
    }
  }
}
```

Then fetch it. This writes `staal.lock` with the SHA-256 of every file, and every
later check verifies the stored copy against it:

```
staalc fetch
```

## Use

Each capability's action:

```
chat.slack: post(channel: Text @target, text: Text) -> Text
chat.discord: post(channel: Text @target, text: Text) -> Text
```

A program imports the module and asks for the capability with its own limits:

```
import chat.slack

fn announce(channel: cap chat.slack.post(only: ["C0123456789"], uses: 1)) -> Receipt {
    return channel.post("C0123456789", "Deploy finished.")
}
```

The operator supplies what the library needs in the run's `grants`:

```json
{
  "grants": {
    "channel": {
      "cap": "chat.slack.post",
      "needs": {
        "http": {
          "cap": "net.http.post"
        },
        "token": {
          "env": "SLACK_BOT_TOKEN"
        }
      }
    }
  }
}
```

## Notes

Uses a bot token and `chat.postMessage`, not an incoming webhook: a webhook's secret is its URL, and Staal never puts secrets in URLs.

Slack answers 200 even when it refuses a message; the library turns `"ok": false` into a failure.

Slack does not deduplicate requests, so do not grant `idempotency_key`: a post interrupted mid-request stops the run for an operator instead of risking a second message.

Discord posts as a bot. Targets are channel ids (digits), and the bot needs the
SEND_MESSAGES permission in the channel. Its grant:

```json
"discord": { "cap": "chat.discord.post", "only": ["1234567890123456789"], "uses": 5,
             "needs": { "http": { "cap": "net.http.post", "only": ["https://discord.com/api/v10/channels/1234567890123456789/messages"] },
                        "token": { "env": "DISCORD_BOT_TOKEN" } } }
```

Mentions are never parsed (`allowed_mentions` with an empty `parse`), so text from an
agent cannot ping `@everyone`, a role or a person. A text over Discord's 2000
characters is refused before any request. Discord's `enforce_nonce` deduplicates only
"in the past few minutes", which a resumed run is not held to, so it is not used and
`http` is granted without `idempotency_key`. The receipt is the message's id; an answer
for another channel is a failure.

## Status

Tested end to end against a local server that answers like each service, as part of
the [Staal test suite](https://github.com/laurells/staal): Slack in `tests/packages_test.rs`,
Discord by the cases in `tests/` here, run on both engines and killed at every journal
write. Not yet run against the live services.

## License

Copyright 2026 Laurels Ozy Echichinwo. Licensed under the
[Apache License, Version 2.0, with the LLVM Exceptions](LICENSE)
(`Apache-2.0 WITH LLVM-exception`), the same terms as
[Staal](https://github.com/laurells/staal). The licence grants no right to the
name "Staal".
