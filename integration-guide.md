# Sur Drive — Chat, Voice & Video Calls

Frontend integration guide for the **rider app**, **driver app**, and web clients.

## 1. Architecture

| Feature                          | Transport                          |
| -------------------------------- | ---------------------------------- |
| Text chat                        | REST + Socket.IO `/chat`           |
| Photos / documents / voice notes | REST multipart + Socket.IO `/chat` |
| Voice calls                      | Socket.IO `/call` + WebRTC         |
| Video calls                      | Socket.IO `/call` + WebRTC         |
| Call history                     | REST                               |
| Offline notifications            | FCM                                |

Chat and calls are restricted to the **rider and driver of a ride**.

The server is authoritative for ride eligibility. Clients must use:

`GET /chat/rides/:rideId`

and respect:

* `messaging.canSend`
* `messaging.expiresAt`
* `calling.canCall`

WebRTC media does **not** pass through the Sur Drive server. The server only handles signalling and call state.

---

## 2. Availability

| Ride state                         | Chat             | Calls |
| ---------------------------------- | ---------------- | ----- |
| Searching / no driver              | No               | No    |
| Driver assigned / in progress      | Yes              | Yes   |
| Completed / payment pending / paid | Yes, temporarily | No    |
| Cancelled                          | No               | No    |

The post-ride chat window defaults to **30 minutes**.

Non-participants receive:

`404 RIDE_CHAT_NOT_PARTICIPANT`

Do not use client-side ride state as permission. The server re-checks every action.

---

## 3. Authentication

All REST and Socket.IO requests use the normal login access token.

### REST

```http
Authorization: Bearer <accessToken>
```

### Socket.IO

```ts
io(`${BASE}/chat`, {
  auth: { token: accessToken },
});

io(`${BASE}/call`, {
  auth: { token: accessToken },
});
```

Use the current access token on reconnects.

Invalid or expired tokens result in:

```text
connect_error: "Unauthorized"
```

---

# 4. Chat

## 4.1 Open a conversation

```http
GET /chat/rides/:rideId
```

This creates the conversation if necessary.

```ts
interface ConversationView {
  id: string;
  rideId: string;
  counterpart: PublicProfile | null;
  lastMessagePreview: string | null;
  lastMessageAt: string | null;
  unreadCount: number;
  createdAt: string;

  messaging: {
    canSend: boolean;
    expiresAt: string | null;
    code: string | null;
    message: string | null;
  };

  calling: {
    canCall: boolean;
    code: string | null;
    message: string | null;
  };
}
```

Use this response to control the chat and call UI.

---

## 4.2 Message model

```ts
type MessageType =
  | 'text'
  | 'system'
  | 'image'
  | 'audio'
  | 'file';

interface ChatMessage {
  id: string;
  conversationId: string;
  rideId?: string;
  senderId: string;
  type: MessageType;
  content: string;
  attachment: MessageAttachment | null;
  createdAt: string;
}

interface MessageAttachment {
  url: string;
  thumbnailUrl: string | null;
  publicId: string;
  resourceType: 'image' | 'video' | 'raw';
  mimeType: string;
  originalName: string;
  size: number;
  width: number | null;
  height: number | null;
  durationSeconds: number | null;
}
```

Messages are immutable.

Always deduplicate messages by `id`.

---

## 4.3 Send text

### Socket

```ts
socket.emit(
  'send_message',
  { conversationId, content },
  (ack) => {}
);
```

Successful ack:

```json
{
  "success": true,
  "message": {}
}
```

### REST fallback

```http
POST /chat/conversations/:conversationId/messages
Content-Type: application/json
```

```json
{
  "content": "Hello"
}
```

Rules:

* Trim whitespace.
* Maximum 1000 characters.
* Content must not be empty.
* Server re-checks ride eligibility.
* Socket sends are rate limited.

When using REST while the socket is connected, the message may also arrive through `new_message`. Deduplicate by message ID.

---

## 4.4 Receive messages

```ts
chat.on('new_message', (message) => {
  addMessageIfNew(message);
});
```

`new_message` is used for:

* text
* images
* audio
* files
* system messages

If the conversation is open, mark it read. Otherwise update the unread badge.

---

## 4.5 Typing

```ts
chat.emit('typing_start', { conversationId });
chat.emit('typing_stop', { conversationId });

chat.on('user_typing', ...);
chat.on('user_stopped_typing', ...);
```

Do not emit on every keystroke.

Recommended behaviour:

* start once when typing begins
* stop after ~2 seconds of inactivity
* stop on blur or send
* hide automatically after ~3 seconds

---

## 4.6 Read receipts

```ts
chat.emit(
  'message_read',
  { conversationId },
  callback
);
```

Or:

```http
PATCH /chat/conversations/:conversationId/read
```

Receive:

```ts
chat.on('read_receipt', ({
  conversationId,
  userId,
  readAt
}) => {});
```

Messages created at or before `readAt` can be shown as read.

---

## 4.7 Unread count

```http
GET /chat/unread
```

Refresh unread counts:

* after connecting/reconnecting
* when returning to the foreground
* after opening a conversation

---

## 4.8 History

```http
GET /chat/conversations/:conversationId/messages?page=1&limit=30
```

Response:

```ts
interface Paginated<T> {
  items: T[];
  meta: {
    total: number;
    page: number;
    limit: number;
    totalPages: number;
    hasNextPage: boolean;
    hasPreviousPage: boolean;
  };
}
```

Messages are returned newest-first.

Reverse them for display.

Maximum `limit`: **100**.

History remains readable after chat closes.

---

## 4.9 Chat state events

### Chat closed

```ts
chat.on('chat_closed', ({
  rideId,
  code,
  message
}) => {});
```

Disable sending and calling.

### Post-ride window updated

```ts
chat.on('chat_window_updated', ({
  rideId,
  canSend,
  canCall,
  expiresAt
}) => {});
```

Typically:

* `canSend = true`
* `canCall = false`

The server remains authoritative even if the client has stale state.

---

# 5. Attachments & Voice Notes

Attachments use REST multipart uploads.

```http
POST /chat/conversations/:conversationId/attachments
Content-Type: multipart/form-data
```

Fields:

* `file` — required
* `caption` — optional, max 1000 characters

Do **not** manually set the multipart `Content-Type`; let the runtime add the boundary.

Successful response:

```text
201 ChatMessage
```

The returned message should be added to the conversation and deduplicated by ID.

## 5.1 Supported files

| Type     | Formats                                           | Message type |
| -------- | ------------------------------------------------- | ------------ |
| Image    | JPG, JPEG, PNG, WEBP, GIF, HEIC, HEIF             | `image`      |
| Audio    | M4A, MP4, AAC, MP3, OGG/Opus, WebM, WAV, AMR, 3GP | `audio`      |
| Document | PDF, DOC, DOCX, XLS, XLSX, PPT, PPTX, TXT, CSV    | `file`       |

Limits:

* Maximum file size: **10 MB**
* Maximum voice-note duration: **5 minutes**
* Video files are not supported.
* SVG, HTML, executables and archives are rejected.

The server validates the actual file contents.

## 5.2 Rendering

### Image

Use:

* `thumbnailUrl` for previews
* `url` for full-size display
* `width` / `height` to preserve layout

### Audio

Use:

* `url`
* `durationSeconds`

Load audio only when needed.

### File

Display:

* `originalName`
* `size`
* `url`

Escape all user-controlled text before rendering in HTML.

---

# 6. Calls

Calls use Socket.IO `/call` for signalling and WebRTC for media.

The server never handles the audio/video stream.

## 6.1 Call types

```ts
type CallType = 'voice' | 'video';
```

Voice:

```text
audio only
```

Video:

```text
audio + video
```

The call type is fixed when the call starts.

Voice → video upgrade is not supported. End the call and start a new video call.

---

## 6.2 Call flow

### Caller

```text
call_initiate
      ↓
call_incoming
      ↓
callee answers
      ↓
call_peer_joined
      ↓
caller creates SDP offer
      ↓
call_offer
      ↓
callee creates SDP answer
      ↓
call_answer
      ↓
ICE candidates exchanged
      ↓
call_active
      ↓
WebRTC media
```

### Start a call

```ts
socket.emit(
  'call_initiate',
  { rideId, type: 'voice' },
  callback
);
```

For video:

```ts
socket.emit(
  'call_initiate',
  { rideId, type: 'video' },
  callback
);
```

Successful response:

```ts
{
  success: true,
  callId: string,
  type: 'voice' | 'video',
  status: string,
  calleeOnline: boolean,
  iceServers: RTCIceServer[]
}
```

Use the returned `iceServers` for the `RTCPeerConnection`.

Do not hardcode TURN credentials.

---

## 6.3 Incoming call

```ts
socket.on('call_incoming', ({
  callId,
  rideId,
  type,
  caller,
  iceServers
}) => {});
```

The UI must follow `type`.

For a video call, request camera + microphone when answering.

---

## 6.4 Answer / decline

Create the WebRTC peer connection **before** joining:

```ts
socket.emit(
  'call_join',
  { callId },
  callback
);
```

Then process the incoming offer.

Decline:

```ts
socket.emit('call_decline', { callId });
```

---

## 6.5 WebRTC signalling

Caller:

```ts
const offer = await pc.createOffer();
await pc.setLocalDescription(offer);

socket.emit('call_offer', {
  callId,
  sdpType: 'offer',
  sdp: offer.sdp
});
```

Callee:

```ts
await pc.setRemoteDescription({
  type: 'offer',
  sdp
});

const answer = await pc.createAnswer();
await pc.setLocalDescription(answer);

socket.emit('call_answer', {
  callId,
  sdpType: 'answer',
  sdp: answer.sdp
});
```

ICE candidates:

```ts
pc.onicecandidate = ({ candidate }) => {
  if (!candidate) return;

  socket.emit('call_ice_candidate', {
    callId,
    candidate: candidate.candidate,
    sdpMid: candidate.sdpMid,
    sdpMLineIndex: candidate.sdpMLineIndex
  });
};
```

Buffer remote ICE candidates until the remote description has been applied.

---

## 6.6 Media

Voice:

```ts
navigator.mediaDevices.getUserMedia({
  audio: true,
  video: false
});
```

Video:

```ts
navigator.mediaDevices.getUserMedia({
  audio: true,
  video: true
});
```

Add local tracks to the peer connection before creating the offer.

For React Native, use `react-native-webrtc`.

---

## 6.7 Call state events

```ts
call_peer_joined
call_offer
call_answer
call_ice_candidate
call_active
call_ended
call_declined
call_cancelled
call_missed
call_failed
call_handled_elsewhere
```

Terminal events should return the UI to an idle state and release all media tracks.

---

## 6.8 Call controls

Mute:

```ts
audioTrack.enabled = false;
```

Camera:

```ts
videoTrack.enabled = false;
```

Hang up:

```ts
socket.emit('call_end', { callId });
```

Wait for the server's terminal event before treating the call as fully closed.

Start the call timer from:

```ts
call_active.startedAt
```

not from the local UI start time.

---

## 6.9 Call limits

| Rule                         |      Limit |
| ---------------------------- | ---------: |
| Ringing                      | 30 seconds |
| Media connection             | 20 seconds |
| Calls per minute             |          6 |
| Signalling events per minute |        400 |
| Concurrent calls             | 1 per user |

If the `/call` socket disconnects during a call, the server ends the call. There is no call resume.

---

# 7. Video

Video uses the same signalling flow as voice.

Client differences:

```text
voice → audio track
video → audio + video tracks
```

Recommended initial constraints:

```ts
{
  audio: true,
  video: {
    width: { ideal: 640 },
    height: { ideal: 480 },
    frameRate: { ideal: 24 }
  }
}
```

Web video elements must use:

```html
<video autoplay playsinline></video>
```

Local video should be muted.

Camera permission failure should be handled as a client-side error. The user may fall back to a voice call.

Video calls can be disabled server-side:

```text
VIDEO_CALLS_DISABLED
```

---

# 8. Call History

```http
GET /calls?page=1&limit=30
```

```http
GET /calls/:callId
```

```ts
interface CallView {
  id: string;
  rideId: string;
  direction: 'outgoing' | 'incoming';
  counterpart: PublicProfile | null;
  type: 'voice' | 'video';
  status:
    | 'initiated'
    | 'ringing'
    | 'active'
    | 'ended'
    | 'declined'
    | 'missed'
    | 'cancelled'
    | 'failed';
  startedAt: string | null;
  endedAt: string | null;
  durationSeconds: number | null;
  createdAt: string;
}
```

Completed calls also appear in chat as `system` messages.

Examples:

```text
📞 Voice call · 1 min 5 sec
📹 Video call · 45 sec
📞 Missed call
📹 Missed video call
📞 Call declined
📹 Video call declined
📞 Call failed
📹 Video call failed
```

Render system messages separately from normal message bubbles.

---

# 9. Push Notifications

Push notifications are sent when the relevant socket is not connected.

## Chat message

```json
{
  "type": "chat_message",
  "rideId": "...",
  "conversationId": "...",
  "senderId": "...",
  "messageType": "text"
}
```

Open the corresponding ride chat and refetch history.

## Incoming call

```json
{
  "type": "incoming_call",
  "callId": "...",
  "rideId": "...",
  "callType": "voice",
  "callerId": "...",
  "callerName": "..."
}
```

Use native incoming-call UI on mobile.

The server keeps the call ringing for **30 seconds** and re-delivers `call_incoming` when the app reconnects during that window.

## Missed call

```json
{
  "type": "missed_call",
  "callId": "...",
  "rideId": "...",
  "callType": "voice"
}
```

---

# 10. Errors

Always switch on `code`, not `message`.

## Chat

| Code                        | Meaning                       |
| --------------------------- | ----------------------------- |
| `RIDE_CHAT_NOT_MATCHED`     | No driver assigned            |
| `RIDE_CHAT_RIDE_CANCELLED`  | Ride cancelled                |
| `RIDE_CHAT_WINDOW_EXPIRED`  | Post-ride chat window expired |
| `RIDE_CHAT_NOT_PARTICIPANT` | User is not part of the ride  |
| `RIDE_CALL_NOT_ACTIVE`      | Calls are unavailable         |

## Calls

| Code                      | Meaning                                 |
| ------------------------- | --------------------------------------- |
| `CALL_BUSY`               | Either participant is already in a call |
| `CALL_ANSWERED_ELSEWHERE` | Another device answered                 |
| `CALL_NOT_ACTIVE`         | Call already ended                      |
| `NOT_PARTICIPANT`         | User is not part of the call            |
| `BAD_SIGNAL`              | Invalid SDP signalling role             |
| `VIDEO_CALLS_DISABLED`    | Video calls disabled server-side        |
| `RATE_LIMITED`            | Too many socket events                  |

## Attachments

| Code                          | Meaning                         |
| ----------------------------- | ------------------------------- |
| `ATTACHMENT_EMPTY`            | Empty file                      |
| `ATTACHMENT_TOO_LARGE`        | Over 10 MB                      |
| `ATTACHMENT_TYPE_NOT_ALLOWED` | Unsupported format              |
| `ATTACHMENT_CONTENT_MISMATCH` | File content doesn't match type |
| `ATTACHMENT_TOO_LONG`         | Voice note over 5 minutes       |
| `ATTACHMENT_REQUIRED`         | Missing file                    |

Also handle:

```text
503
INTERNAL_ERROR
Unauthorized
```

---

# 11. Reconnection

Socket events are **not replayed**.

After `/chat` reconnects, refetch:

1. `GET /chat/unread`
2. Open conversation history, if applicable
3. `GET /chat/rides/:rideId`

Merge messages by `id`.

Repeat the same refresh when the app returns to the foreground.

If `/call` disconnects during an active call, treat the call as failed and return to idle.

---
