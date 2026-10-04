# Support Module — Frontend Integration Guide


## 1. Integration rules

Support uses three channels:

| Purpose | Channel |
|---|---|
| CRUD/actions | REST: `/support` and `/admin/support` |
| Realtime | Socket.IO: `/support` |
| Offline alerts | Push: `data.type = "support_ticket"` |


## 2. Core model

### Roles and bases

| Role | REST base | Client |
|---|---|---|
| `rider`, `ride_driver` | `/support` | Rider/driver app |
| `admin`, `super_admin` | `/admin/support` | Admin dashboard |

Rider and driver endpoints are identical; the token determines the requester role. Calling the other side's base path returns `403`.

### Ticket

- `id`: UUID used in URLs.
- `reference`: e.g. `TR-2048`; display as `#TR-2048`.
- `ticketNumber`: numeric form, e.g. `2048`.
- `priority`: `low | medium | high` (`medium` default).
- `status`: `open | in_progress | resolved`.

Requester UI maps `open` and `in_progress` to **Open**, and `resolved` to **Closed**.

Requester/driver filters:
- `status=open` → `open` + `in_progress`
- `status=closed` → `resolved`
- Omit `status` for All.
- Do **not** send `status=resolved`.

### Lifecycle

```text
open → in_progress → resolved
 ↑          ↑             │
 └──────────┴── reopen ───┘
```

Reopen returns `in_progress` when owned, otherwise `open`.

Changing priority does not change status or owner. Changing priority to `high` creates an `escalated` system notice.

### Messages

| `senderType` | `type` | UI |
|---|---|---|
| `requester`, `agent` | `text`, `image`, `audio`, `file` | Chat bubble |
| `system` | `text` | Support bubble |
| `system` | `system` | Centered notice |

The viewer's own messages are `requester` in rider/driver apps and `agent` in admin.

REST returns messages **newest first**. Reverse them for display. Do not re-sort by timestamp alone: the opening message and greeting share the same `createdAt`.

### Permissions

```json
{
  "canReply": true,
  "canResolve": true,
  "canReopen": false,
  "canAssign": false,
  "canChangePriority": false
}
```

| Permission | UI |
|---|---|
| `canReply` | Composer |
| `canResolve` | Mark Resolved |
| `canReopen` | Reopen |
| `canAssign` | Admin Claim/Assign |
| `canChangePriority` | Admin priority control |

Refetch ticket detail after `ticket_updated`.

### Badges

- `needsYourReply`: requester/driver should reply.
- `awaitingAgent`: requester spoke last; admin should reply.
- `unreadCount`: unread human messages for the viewer.

### Limits

| Field | Limit |
|---|---|
| Subject | Required, 150 chars |
| Description | Required, 2,000 chars |
| Message | Required, 2,000 chars |
| Open tickets/person | 5 by default |
| Attachment | 1 file, 10 MB |
| Voice note | 5 minutes |

Server trims text; whitespace-only messages are rejected.

## 3. Rider / driver app

### Create ticket

`POST /support/tickets`

```json
{
  "subject": "Required",
  "description": "Required",
  "priority": "medium",
  "rideId": "optional-uuid"
}
```

Omit optional fields when unused.

`GET /support/rides` returns the 20 most recent rides involving the user.

After `201`, navigate to the returned ticket's chat using `id`.

### Ticket list

`GET /support/tickets`

Query:

| Param | Values |
|---|---|
| `status` | `open`, `closed` |
| `priority` | `low`, `medium`, `high` |
| `startDate`, `endDate` | `YYYY-MM-DD` or ISO timestamp |
| `page` | Starts at `1` |
| `limit` | `1–100`, default `30` |

Omit unused filters. Never send `status=` or `priority=`.

Dates filter by ticket creation date. Bare dates cover the full UTC day. Invalid ranges return `SUPPORT_INVALID_DATE_RANGE`.

Server order is latest message first; do not re-sort.

Card fields:

| UI | Field |
|---|---|
| Status | `status` (`resolved` → Closed) |
| Priority | `priority` |
| Reference | `reference` |
| Title | `subject` |
| Description | `descriptionPreview` |
| Agent | `agent?.name` or `Support agent` |
| Action | Reply unless resolved; otherwise Details |
| Relative time | `lastMessageAt ?? createdAt` |
| Reply badge | `needsYourReply` |
| Unread | `unreadCount` |

If `meta.total === 0`, show **No tickets found**. Show Clear filters only when a filter is active.

Unread badge:

`GET /support/tickets/unread`

```json
{
  "total": 3,
  "tickets": [{ "ticketId": "...", "unreadCount": 2 }]
}
```

### Chat

On open:

1. `GET /support/tickets/:id`
2. `GET /support/tickets/:id/messages?page=1&limit=50`
3. Reverse messages for display.
4. Connect the socket.
5. Mark read with `PATCH /support/tickets/:id/read`.

Mark read again when an agent message arrives while the chat is open.

Message shape:

```json
{
  "id": "uuid",
  "ticketId": "uuid",
  "senderId": "uuid",
  "senderType": "agent",
  "type": "text",
  "content": "Hello",
  "attachment": null,
  "createdAt": "ISO"
}
```

#### Send message

Prefer socket:

```text
send_message { ticketId, content }
→ { success: true, message }
```

Fallback:

`POST /support/tickets/:id/messages`

```json
{ "content": "Hello" }
```

Use an optimistic pending bubble. Replace it with the server message and de-duplicate the `new_message` echo by `id`.

If a network send fails, refetch messages before offering Retry.

#### Typing

- Send `typing_start` at most every 2.5s.
- Send `typing_stop` after ~2s of silence or on send.
- Hide typing after `user_stopped_typing`, their message, or ~4s.

#### Seen

`read_receipt` provides `readAt`. Mark own messages with `createdAt <= readAt` as seen.

#### Resolve / reopen

Resolve:

`POST /support/tickets/:id/resolve`

Show only when `permissions.canResolve`. Confirm first, then refetch detail.

When `canReply === false`, replace the composer with the resolved footer.

- `canReopen === false`: show **This ticket has been resolved. No further actions can be taken.**
- `canReopen === true`: show Reopen.
- Reopen: `POST /support/tickets/:id/reopen`
- `SUPPORT_REOPEN_WINDOW_EXPIRED`: explain it cannot be reopened and offer a new ticket.

## 4. Admin dashboard

### Inbox

Stats:

`GET /admin/support/tickets/stats`

Returns:

`total, open, inProgress, resolved, unassigned, mine, awaitingAgent, highPriority`

`unassigned`, `awaitingAgent`, and `highPriority` count unresolved tickets. `mine` counts tickets assigned to the caller.

List:

`GET /admin/support/tickets`

| Param | Values |
|---|---|
| `status` | `open`, `in_progress`, `resolved` |
| `priority` | `low`, `medium`, `high` |
| `assignee` | `me`, `unassigned`, admin UUID |
| `requesterRole` | `rider`, `ride_driver` |
| `search` | Reference, number, subject, requester name/phone; max 100 chars |
| `sort` | `recent`, `oldest`, `priority` |
| `startDate`, `endDate` | Creation date |
| `page`, `limit` | Same as requester API |

Admin rows include requester, assignee and `awaitingAgent`.

Realtime inbox:
- `ticket_created` and `ticket_updated` are delivered to connected agents without joining.
- Patch existing rows from `ticket_updated`.
- Refetch/debounce the inbox for a new ticket because requester details and `unreadCount` are not included.

### Ticket workspace

On open:

1. `GET /admin/support/tickets/:id`
2. `GET /admin/support/tickets/:id/messages`
3. Reverse messages.
4. Emit `join_ticket { ticketId }`.
5. Mark read.

Emit `leave_ticket` when leaving/switching tickets.

Detail may include `description`, `requester`, `assignee`, `ride`, `resolutionNote`, `resolvedBy`, `reopenCount`, and `permissions`.

### Admin actions

| Action | Endpoint | Rules |
|---|---|---|
| Claim | `POST /admin/support/tickets/:id/assign` `{}` | Admin; unowned ticket |
| Assign | Same + `{ assigneeId }` | `super_admin` only |
| Reply | `POST .../messages` or socket | Owner or `super_admin`; reply on unowned ticket claims it |
| Priority | `PATCH .../priority` | Any agent on unresolved ticket |
| Resolve | `POST .../resolve` `{ note? }` | Owner or `super_admin`; note is internal, max 1,000 chars |
| Reopen | `POST .../reopen` | Any agent |

Action responses are ticket snapshots, not full details. Refetch detail afterward.

`SUPPORT_TICKET_ASSIGNED_TO_OTHER` → refetch and hide controls.

## 5. Socket.IO

Namespace:

`/support`

### Server → client

| Event | Payload / behavior |
|---|---|
| `support_ready` | `{ userId, role }` |
| `new_message` | Message |
| `ticket_created` | Ticket snapshot; agents only |
| `ticket_updated` | Snapshot + `change` |
| `read_receipt` | `{ ticketId, userId, readAt }` |
| `user_typing` | `{ ticketId, userId, senderType }` |
| `user_stopped_typing` | Same |
| `exception` | `{ statusCode, code, message }` |

`change`:

`assigned | priority_changed | resolved | reopened | message`

The `message` change is sent to agents for inbox updates.

### Client → server

| Event | Payload | Ack |
|---|---|---|
| `send_message` | `{ ticketId, content }` | `{ success, message }` |
| `typing_start/stop` | `{ ticketId }` | `{ success }` |
| `message_read` | `{ ticketId }` | `{ success, readAt }` |
| `join_ticket/leave_ticket` | `{ ticketId }` | `{ success }`; agents only |

Use a 5-second socket timeout.

Socket errors appear in both the ack and `exception`. Handle the ack for user-facing errors and only log the duplicate event.

Rate limits per connection:
- Messages: 30/min
- Typing/read/join signals: 120/min
- Exceeded → `RATE_LIMITED`

### Reconnect

Socket.IO does not replay missed events.

On every `connect`:

1. Refetch list/inbox.
2. If a ticket is open, refetch detail + messages and merge by `id`.
3. Agents re-emit `join_ticket`.

## 6. Push notifications

Push is sent only when the recipient has no `/support` socket connection.

| Trigger | Recipient | `data.event` |
|---|---|---|
| Agent message | Requester | `message` |
| Requester message | Assignee | `message` |
| Agent resolves | Requester | `resolved` |
| Super admin assigns | New assignee | `assigned` |

Payload:

```json
{
  "type": "support_ticket",
  "event": "message",
  "ticketId": "uuid",
  "ticketNumber": "2048"
}
```

On tap:
1. Check `data.type === "support_ticket"`.
2. Open the ticket chat.
3. Load fresh data over REST.
4. If the ticket is unavailable, show **Ticket not found**.

Treat push as a pointer, not state.

## 7. Errors

### Error shapes

Domain error:

```json
{
  "statusCode": 409,
  "code": "SUPPORT_TICKET_RESOLVED",
  "message": "This ticket has been resolved."
}
```

Validation error:

```json
{
  "statusCode": 400,
  "message": ["subject should not be empty"],
  "error": "Bad Request"
}
```

Validation errors have no `code`; `message` is an array.

### Domain errors

| Code | HTTP | Action |
|---|---:|---|
| `SUPPORT_TICKET_NOT_FOUND` | 404 | Go to list; show Ticket not found |
| `SUPPORT_TICKET_RESOLVED` | 409 | Refetch detail; show resolved state |
| `SUPPORT_TICKET_LIMIT_REACHED` | 429 | Show limit; link to open tickets |
| `SUPPORT_TICKET_ASSIGNED_TO_OTHER` | 403 | Refetch; hide controls |
| `SUPPORT_ALREADY_ASSIGNED` | 409 | Treat as done; refetch |
| `SUPPORT_ALREADY_RESOLVED` | 409 | Treat as done; refetch |
| `SUPPORT_NOT_RESOLVED` | 409 | Refetch |
| `SUPPORT_REOPEN_WINDOW_EXPIRED` | 403 | Cannot reopen; offer new ticket |
| `SUPPORT_ASSIGN_FORBIDDEN` | 403 | Hide assignment picker |
| `SUPPORT_INVALID_ASSIGNEE` | 400 | Reload agents |
| `SUPPORT_NOT_AN_AGENT` | 403 | Client bug |
| `SUPPORT_RIDE_NOT_FOUND` | 404 | Reload rides |
| `SUPPORT_INVALID_DATE_RANGE` | 400 | Show filter error |
| `SUPPORT_EMPTY_MESSAGE` | 400 | Keep Send disabled |
| `RATE_LIMITED` | Socket | Slow down/retry later |

Other HTTP errors:
- `401`: missing/invalid token.
- `403` without code: wrong endpoint/side.
- `413`: upload rejected.

Do not auto-retry `4xx`. Sending has no idempotency key; refetch before retrying a network failure.

## 8. Attachments

Upload:

`POST /support/tickets/:id/attachments`  
or  
`POST /admin/support/tickets/:id/attachments`

`multipart/form-data`:

| Field | Rule |
|---|---|
| `file` | Required; exactly one |
| `caption` | Optional; max 2,000 chars; becomes message `content` |



Limits:
- 10 MB/file
- Voice notes max 5 minutes
- Server validates actual file content

| Kind | Types |
|---|---|
| Image | JPEG, PNG, WebP, GIF, HEIC/HEIF |
| Audio | M4A/MP4, AAC, MP3, OGG/Opus, WebM, WAV, AMR, 3GP |
| File | PDF, DOC, DOCX, XLS, XLSX, PPT, PPTX, TXT, CSV |

Attachment errors:

| Code | HTTP |
|---|---:|
| `ATTACHMENT_REQUIRED` | 400 |
| `ATTACHMENT_EMPTY` | 400 |
| `ATTACHMENT_TOO_LARGE` | 413 |
| `ATTACHMENT_TYPE_NOT_ALLOWED` | 400 |
| `ATTACHMENT_CONTENT_MISMATCH` | 400 |
| `ATTACHMENT_TOO_LONG` | 400 |

Render:
- Image: `thumbnailUrl ?? url`; open `url`.
- Audio: player + `durationSeconds`.
- File: `originalName` + `size`; open `url`.
- Caption is `content`.

## 9. Frontend state rules

Keep API, socket, and message-merging logic separate from UI.

### Message merge

```ts
export function upsertMessage(list: Message[], message: Message): Message[] {
  const index = list.findIndex(x => x.id === message.id);
  if (index === -1) return [...list, message];

  const next = [...list];
  next[index] = message;
  return next;
}
```

### State updates

| Event | State update |
|---|---|
| `connect` | Resync list/inbox; open ticket detail/messages; rejoin admin ticket |
| `new_message` open ticket | `upsertMessage`; mark read if from other side |
| `new_message` other ticket | Increment unread/update preview |
| `ticket_updated` | Patch row; refetch open detail for permissions |
| `ticket_created` | Debounced inbox refetch |
| `read_receipt` | Mark own messages seen |
| Typing events | Toggle typing indicator |



### Endpoint reference

### Rider / driver: `/support`

| Method | Path | Input | Returns |
|---|---|---|---|
| GET | `/support/rides` | — | Up to 20 recent rides |
| POST | `/support/tickets` | `subject`, `description`, `priority?`, `rideId?` | Ticket detail |
| GET | `/support/tickets` | Filters + paging | Paginated cards |
| GET | `/support/tickets/unread` | — | Unread summary |
| GET | `/support/tickets/:id` | — | Ticket detail |
| GET | `/support/tickets/:id/messages` | `page?`, `limit?` | Messages, newest first |
| POST | `/support/tickets/:id/messages` | `content` | Message |
| POST | `/support/tickets/:id/attachments` | multipart `file`, `caption?` | Message |
| PATCH | `/support/tickets/:id/read` | — | `{ readAt }` |
| POST | `/support/tickets/:id/resolve` | — | Snapshot |
| POST | `/support/tickets/:id/reopen` | — | Snapshot |

### Admin: `/admin/support`

| Method | Path | Input | Returns |
|---|---|---|---|
| GET | `/admin/support/agents` | — | Agents |
| GET | `/admin/support/tickets` | Filters + paging | Paginated admin cards |
| GET | `/admin/support/tickets/stats` | — | Stats |
| GET | `/admin/support/tickets/:id` | — | Admin detail |
| GET | `/admin/support/tickets/:id/messages` | `page?`, `limit?` | Messages, newest first |
| POST | `/admin/support/tickets/:id/messages` | `content` | Message |
| POST | `/admin/support/tickets/:id/attachments` | multipart `file`, `caption?` | Message |
| PATCH | `/admin/support/tickets/:id/read` | — | `{ readAt }` |
| POST | `/admin/support/tickets/:id/assign` | `{}` / `assigneeId` | Snapshot |
| PATCH | `/admin/support/tickets/:id/priority` | `priority` | Snapshot |
| POST | `/admin/support/tickets/:id/resolve` | `note?` | Snapshot |
| POST | `/admin/support/tickets/:id/reopen` | — | Snapshot |

### Socket: `/support`

**Client → server:**  
`send_message`, `typing_start`, `typing_stop`, `message_read`, `join_ticket`, `leave_ticket`

**Server → client:**  
`support_ready`, `new_message`, `ticket_created`, `ticket_updated`, `read_receipt`, `user_typing`, `user_stopped_typing`, `exception`

**Success:** `201` for ticket/message/attachment creation; `200` for other REST operations.
