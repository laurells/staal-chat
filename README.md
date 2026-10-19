# staal-chat

Messages to Slack channels: a [Staal](https://github.com/laurells/staal) package that gives
programs the `chat.slack.post` capability, built on Staal's general `net.http`
capability. It talks to [Slack](https://api.slack.com/methods/chat.postMessage).

## Authority

This is everything the package can reach. `staalc fetch` prints it when the
package is added, and the compiler refuses the package if its code reaches
beyond it.

| Capability | Targets |
| --- | --- |
| `net.http.post` | `https://slack.com/api/chat.postMessage` |

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

The capability's action:

```
chat.slack: post(channel: Text @target, text: Text) -> Text
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

## Status

Tested end to end against a local server that answers like the service, as part of
the [Staal test suite](https://github.com/laurells/staal/blob/main/tests/packages_test.rs).
Not yet run against the live service.

## License

Private and unlicensed: no permission is granted to use, copy or distribute this
code. A license will be chosen before any public release.
