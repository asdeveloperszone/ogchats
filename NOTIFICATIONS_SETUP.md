# ASChat — Push Notification Setup

How push notifications work and what you need to verify before deploying.

---

## Architecture

```
A sends message
      │
      ▼
Firebase Realtime DB (message written)
      │
      ▼
A's chat.js calls pushToReceiver()  ← SENDER fires this, not receiver
      │
      ▼
Railway backend /api/send
      │
      ▼
web-push library sends to B's push subscription
      │
      ▼
B's OS wakes sw.js even if browser/PWA is fully closed
      │
      ▼
OS notification shown on B's device
```

Two layers work together:

| Layer | Mechanism | Works when |
|-------|-----------|------------|
| Layer 1 | SW `postMessage` | App open or tab backgrounded |
| Layer 2 | Railway Web Push | App/browser **fully closed**, phone locked |

---

## Railway Backend Endpoints

Your backend at `https://aschatbackend-production.up.railway.app` must expose:

### `GET /api/vapid-public-key`
Returns `{ key: "BEl62iU..." }` — the VAPID public key.

### `POST /api/subscribe`
Body: `{ userID, subscription }` — stores user's push subscription.

### `POST /api/unsubscribe`
Body: `{ userID, subscription }` — removes subscription on logout.

### `POST /api/send`
Header: `x-api-secret: <your secret>`
Body:
```json
{
  "receiverID": "123456789",
  "senderID":   "987654321",
  "senderName": "Alice",
  "senderPhoto": "https://...",
  "type": "message",
  "text": "Hey!",
  "callType": "voice",
  "timestamp": 1234567890
}
```
`type` values: `message` | `photo` | `voice` | `call` | `missed_call` | `reaction`

The backend looks up `receiverID`'s stored subscription and sends the push.

### `GET /api/ice-servers`
Returns array of TURN/STUN server configs for WebRTC calls.

---

## Checklist Before Deploy

- [ ] Railway backend is deployed and running
- [ ] `PUSH_SERVER_URL` in `js/notifications.js` matches your Railway URL
- [ ] `API_SECRET` in `js/notifications.js` matches your Railway backend secret
- [ ] VAPID keys are generated and set on the backend
- [ ] Firebase hosting deployed with `firebase deploy --only hosting`
- [ ] Firebase DB rules deployed with `firebase deploy --only database`

---

## Testing

1. Install the PWA on two devices (Add to Home Screen)
2. On Device B: open app, grant notification permission, then **close the app completely**
3. On Device A: send a message to Device B
4. Device B should receive an OS push notification within 1–2 seconds

For calls: start a call from A → B receives a persistent notification with Accept/Decline buttons even with app closed.

---

## Troubleshooting

**No notification when app is closed**
→ Check Railway backend logs — is `/api/send` being called?
→ Check that B's subscription was saved (`/api/subscribe` called on login)
→ On Android: App info → Notifications → make sure ASChat is allowed

**Notifications work in browser but not installed PWA**
→ Android: Long-press PWA icon → App info → Notifications → Enable
→ iOS 16.4+: Must be added to Home Screen. Settings → Notifications → ASChat → Allow

**"Push subscription failed" in console**
→ VAPID key mismatch — regenerate on backend and redeploy
→ SW not registered yet — check browser console for SW errors

**Notification tapped but wrong chat opens**
→ `SW_BASE` in sw.js is derived from sw.js path — should resolve to `/ogchats` automatically

---

## Files Reference

| File | Purpose |
|------|---------|
| `js/notifications.js` | Push subscription, `pushToReceiver()` (sender-side), Layer 1 SW bridge |
| `js/chat.js` | Calls `pushToReceiver()` after every send |
| `js/call.js` | Calls `pushToReceiver()` when starting a call |
| `sw.js` | Handles `push` events, shows OS notifications, handles notification clicks |
| `firebase.json` | Hosting config — correct headers for sw.js and manifest.json |
| `database.rules.json` | Firebase security rules |
