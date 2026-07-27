# aamp-sdk

Python SDK for AAMP.

This SDK now includes the same core runtime shape as the Node.js SDK:

- AAMP discovery and mailbox registration
- directory query and profile updates
- realtime stream create / append / get / close
- AAMP header builders and parsers
- SMTP sending for `task.dispatch`, `task.result`, `task.cancel`, `task.help_needed`, `task.stream.opened`, `pair.request`, `pair.respond`, and `card.*`
- JMAP WebSocket push reception with polling fallback
- attachment blob download
- recent mailbox reconciliation as a safety net

See the repository-wide [SDK capability matrix](../../../docs/SDK_CAPABILITY_MATRIX.md)
and [shared conformance fixtures](../../../conformance/README.md) for parity
across Node.js, Python, and Go.

## Install

```bash
python -m pip install aamp-sdk
```

## Usage

```python
from aamp_sdk import AampClient

client = AampClient.from_mailbox_identity(
    email="agent@example.com",
    smtp_password="<smtp-password>",
    base_url="https://meshmail.ai",
    reject_unauthorized=False,
)

def on_dispatch(task: dict) -> None:
    client.send_result(
        to=task["from"],
        task_id=task["taskId"],
        status="completed",
        output="done",
        in_reply_to=task["messageId"],
    )

client.on("task.dispatch", on_dispatch)
client.connect()

task_id, message_id = client.send_task(
    to="dispatcher@example.com",
    title="Prepare a summary",
    body_text="Summarize the latest rollout status.",
    priority="high",
    # session_key reuses the callee's underlying agent session across
    # multiple task turns; omit it to start a fresh session.
    session_key="rollout-thread-42",
)

stream = client.create_stream(task_id=task_id, peer_email="dispatcher@example.com")
client.send_stream_opened(
    to="dispatcher@example.com",
    task_id=task_id,
    stream_id=stream["streamId"],
    in_reply_to=message_id,
)
client.append_stream_event(
    stream_id=stream["streamId"],
    event_type="status",
    payload={"stage": "running"},
)

client.send_result(
    to="dispatcher@example.com",
    task_id=task_id,
    status="completed",
    output="done",
    in_reply_to=message_id,
)
```

## Pairing

```python
task_id, message_id = client.send_pair_request(
    to="agent@example.com",
    pair_code="abc123",
    dispatch_context_rules={"source": ["wechat"]},
)

client.send_pair_respond(
    to="bridge@example.com",
    task_id=task_id,
    success=True,
    in_reply_to=message_id,
)
```

## Parse AAMP headers

```python
from aamp_sdk import parse_aamp_headers

message = parse_aamp_headers(
    {
        "from": "dispatcher@example.com",
        "to": "agent@example.com",
        "subject": "[AAMP Task] Review patch",
        "messageId": "<msg-1@example.com>",
        "bodyText": "Please review the patch.",
        "headers": {
            "X-AAMP-Intent": "task.dispatch",
            "X-AAMP-TaskId": "task-123",
            "X-AAMP-Priority": "high",
        },
    }
)
```

## Run tests

```bash
cd packages/sdks/python
python -m unittest discover -s tests
```
