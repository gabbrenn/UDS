# Implementation Summary

## Overview

This guide explains how to implement three key chat features:
1. **Delete Message** - Allow users to delete their messages
2. **File Attachments** - Send and display images/files in messages
3. **Socket.IO Integration** - Real-time message updates

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Mobile App (React Native)                │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ ChatDetail   │  │ ChatList     │  │ ChatInputBar │      │
│  │ Screen       │  │ Screen       │  │ Component    │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                  │                  │               │
│         │                  │                  │               │
│  ┌──────▼──────────────────▼──────────────────▼───────┐     │
│  │           Socket Context (Socket.IO)                │     │
│  │  - Real-time events (new_message, message_read)    │     │
│  └─────────────────────────────────────────────────────┘     │
│         │                                                      │
│  ┌──────▼──────────────────────────────────────┐             │
│  │         Chat Service (API Client)            │             │
│  │  - getChats()                                │             │
│  │  - getChatById()                             │             │
│  │  - sendMessage() (with FormData for files)  │             │
│  │  - deleteMessage()                           │             │
│  └──────────────────────────────────────────────┘             │
│                                                               │
└───────────────────────┬───────────────────────────────────────┘
                        │
                        │ HTTP REST API
                        │ WebSocket (Socket.IO)
                        │
┌───────────────────────▼───────────────────────────────────────┐
│                    Backend (Fastify + Socket.IO)               │
├───────────────────────────────────────────────────────────────┤
│  - Chat Controller    - Socket.IO Server                      │
│  - Chat Service       - User-specific rooms (by userId)       │
│  - Chat Repository    - Events: new_message, message_read,    │
│                         message_deleted, chat_updated         │
└───────────────────────────────────────────────────────────────┘
```

---

## Feature Breakdown

### 1. Delete Message

**Flow:**
```
User long-presses message
  → Alert menu appears
  → User selects "Delete for Me" or "Delete for Everyone"
  → Optimistic UI update (message removed/marked as deleted)
  → API call: DELETE /api/chats/messages/:messageId
  → On success: UI already updated
  → On error: Rollback to original state
```

**Key Components:**
- `MessageBubble`: Long press handler, delete menu
- `ChatDetailScreen`: Delete handler with optimistic updates
- `chatService.deleteMessage()`: API call

**Backend Support:** ✅ Already implemented

---

### 2. File Attachments

**Flow:**
```
User taps attach button
  → Image picker opens (expo-image-picker)
  → User selects images
  → Images show in preview
  → User can remove before sending
  → On send: Convert to FormData
  → API call: POST /api/chats/messages (multipart/form-data)
  → Backend saves files and returns URLs
  → Display attachments in MessageBubble
```

**Key Components:**
- `filePicker.ts`: Image selection utility
- `ChatInputBar`: File selection UI and preview
- `AttachmentDisplay`: Display attachments in messages
- `MessageBubble`: Render attachments
- `chatService.sendMessage()`: FormData upload

**Backend Support:** ✅ Already implemented (multipart/form-data)

---

### 3. Socket.IO Integration

**Flow:**
```
App starts (user logged in)
  → Socket connects with userId in handshake.auth
  → Backend joins socket to userId-specific room
  → Event listeners registered in components
  → Real-time events received and UI updates
  → On logout: Socket disconnects
```

**Socket Events:**
- `new_message`: New message received → Add to chat
- `message_read`: Messages read → Update read status
- `message_deleted_for_all`: Message deleted for everyone → Remove from chat
- `message_deleted_for_me`: Message deleted for me → Mark as deleted
- `chat_updated`: Chat updated → Refresh chat list
- `new_chat`: New chat created → Refresh chat list

**Key Components:**
- `socket.service.ts`: Singleton socket instance
- `SocketContext`: React context for socket access
- `ChatDetailScreen`: Message event listeners
- `ChatListScreen`: Chat list event listeners

**Backend Support:** ✅ Already implemented

---

## Implementation Order

### Recommended Order:
1. **Delete Message** (Simplest, no new dependencies)
2. **File Attachments** (Requires expo-image-picker)
3. **Socket.IO** (Most complex, benefits from other features)

### Why This Order?
- Start simple to build confidence
- Each feature builds on previous knowledge
- Socket.IO benefits from having delete/attachments working first
- Easier debugging when features are implemented incrementally

---

## Key Files to Modify/Create

### Existing Files to Modify:
1. `homexa-mobile/src/features/chat/types/index.ts` - Add types
2. `homexa-mobile/src/services/chat.service.ts` - Add deleteMessage, update sendMessage
3. `homexa-mobile/src/features/chat/components/MessageBubble.tsx` - Delete UI, attachments
4. `homexa-mobile/src/features/chat/components/ChatInputBar.tsx` - File picker UI
5. `homexa-mobile/src/features/chat/screens/ChatDetailScreen.tsx` - Delete handler, socket listeners, attachments
6. `homexa-mobile/src/features/chat/screens/ChatListScreen.tsx` - Socket listeners
7. `homexa-mobile/app/_layout.tsx` - Add SocketProvider

### New Files to Create:
1. `homexa-mobile/src/utils/filePicker.ts` - Image picker utility
2. `homexa-mobile/src/features/chat/components/AttachmentDisplay.tsx` - Attachment UI
3. `homexa-mobile/src/services/socket.service.ts` - Socket singleton
4. `homexa-mobile/src/contexts/SocketContext.tsx` - Socket React context

---

## Dependencies to Install

```bash
# Required for file attachments
npm install expo-image-picker

# Required for Socket.IO
npm install socket.io-client
```

**Note:** `expo-image-picker` is already in your package.json, so only `socket.io-client` needs to be installed.

---

## Important Configuration Notes

### Socket URL Configuration
The socket connects to the same server as your API but without the `/api` path:
- API URL: `https://example.com/api`
- Socket URL: `https://example.com`

The socket service automatically handles this by stripping `/api` from `API_BASE_URL`.

### Authentication
- REST API: Uses `Bearer` token in `Authorization` header
- Socket.IO: Uses `userId` in `handshake.auth.userId` (no token needed for basic setup)

### File Upload Format
React Native FormData format is different from web:
```typescript
formData.append('file', {
  uri: file.uri,      // Local file URI
  type: file.type,    // MIME type
  name: file.name,    // Filename
} as any);
```

---

## Testing Strategy

### Unit Testing
- Test file validation logic
- Test socket event handlers
- Test delete message logic

### Integration Testing
- Test full message flow (send → receive → delete)
- Test file upload → display flow
- Test socket events with real backend

### Manual Testing Checklist
- [ ] Delete message works (both modes)
- [ ] File attachments upload and display
- [ ] Socket events update UI in real-time
- [ ] Network loss/recovery works
- [ ] No memory leaks (check listeners cleanup)
- [ ] Multiple devices work simultaneously

---

## Common Pitfalls & Solutions

### 1. Socket Not Connecting
**Problem:** Socket.IO client can't connect to server

**Solutions:**
- Verify `API_BASE_URL` format (should strip `/api`)
- Check `userId` is passed in `handshake.auth`
- Verify backend socket server is running
- Check CORS settings on backend
- Test with browser console first

### 2. Duplicate Messages
**Problem:** Messages appear twice (optimistic UI + socket event)

**Solutions:**
- Check message ID before adding to state
- Remove optimistic UI if socket receives same message
- Use message ID as React key

### 3. Files Not Uploading
**Problem:** FormData not formatted correctly for React Native

**Solutions:**
- Use React Native FormData format (see code example)
- Verify Content-Type header (should be `multipart/form-data`)
- Check file size limits on backend
- Test endpoint with Postman first

### 4. Memory Leaks
**Problem:** Socket listeners not cleaned up

**Solutions:**
- Always return cleanup function in `useEffect`
- Remove all listeners in cleanup
- Disconnect socket on logout/unmount

---

## Next Steps

1. **Read the Detailed Guide**: `DETAILED_IMPLEMENTATION.md` for code examples
2. **Follow the Checklist**: `QUICK_START_CHECKLIST.md` for step-by-step tracking
3. **Start with Phase 1**: Delete Message (simplest)
4. **Test Thoroughly**: Each feature before moving to next
5. **Iterate**: Refine and optimize after all features work

---

## Support Resources

- **Backend Socket Events**: Check `homexa-bn/SOCKET_FRONTEND_INTEGRATION.md`
- **Expo Image Picker**: https://docs.expo.dev/versions/latest/sdk/image-picker/
- **Socket.IO Client**: https://socket.io/docs/v4/client-api/

---

## Summary

All three features are well-supported by the backend. The implementation is straightforward:

- **Delete Message**: Simple UI + API call
- **File Attachments**: FormData upload + display component
- **Socket.IO**: Connect + listen to events + update UI

Follow the guides step-by-step, and you'll have a fully-featured real-time chat system! 🚀

