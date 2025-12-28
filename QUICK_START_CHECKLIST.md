# Quick Start Implementation Checklist

Use this checklist to track your implementation progress.

## Prerequisites
- [ ] Backend is running and accessible
- [ ] User authentication is working
- [ ] Basic chat functionality (send/receive messages) is working

---

## Phase 1: Delete Message (Estimated: 2-3 hours)

### Step 1: Update Types
- [ ] Add `isDeleted?` and `deletedFor?` to `ChatMessage` interface
- [ ] File: `homexa-mobile/src/features/chat/types/index.ts`

### Step 2: Add Service Method
- [ ] Add `deleteMessage()` method to `chatService`
- [ ] File: `homexa-mobile/src/services/chat.service.ts`
- [ ] Test: Call method manually in console to verify endpoint

### Step 3: Update MessageBubble
- [ ] Add `messageId`, `isDeleted`, `onDelete` props
- [ ] Add long press handler
- [ ] Show delete menu (Alert.alert with options)
- [ ] Display "Message deleted" placeholder
- [ ] File: `homexa-mobile/src/features/chat/components/MessageBubble.tsx`

### Step 4: Update ChatDetailScreen
- [ ] Add `handleDeleteMessage` function
- [ ] Implement optimistic UI update
- [ ] Add error handling with rollback
- [ ] Pass props to MessageBubble
- [ ] File: `homexa-mobile/src/features/chat/screens/ChatDetailScreen.tsx`

### Step 5: Testing
- [ ] Long press on sent message shows menu
- [ ] "Delete for Me" works correctly
- [ ] "Delete for Everyone" works correctly
- [ ] Error handling works (test with network off)
- [ ] Cannot delete received messages

---

## Phase 2: File Attachments (Estimated: 4-5 hours)

### Step 1: Update Types
- [ ] Create `MessageAttachment` interface
- [ ] Add `attachments?` to `ChatMessage` interface
- [ ] File: `homexa-mobile/src/features/chat/types/index.ts`

### Step 2: Create File Picker Utility
- [ ] Create `homexa-mobile/src/utils/filePicker.ts`
- [ ] Implement `pickImage()` using expo-image-picker
- [ ] Implement `validateFile()` for size/type checks
- [ ] Handle permissions
- [ ] Test: Verify image selection works

### Step 3: Update ChatInputBar
- [ ] Add `selectedFiles` state
- [ ] Add file picker button handler
- [ ] Add file preview UI (scrollable row of images)
- [ ] Add remove file functionality
- [ ] Update `onSendMessage` signature to accept attachments
- [ ] File: `homexa-mobile/src/features/chat/components/ChatInputBar.tsx`

### Step 4: Update Chat Service
- [ ] Update `sendMessage()` to use FormData
- [ ] Append files, product_id, content, chatId to FormData
- [ ] Set Content-Type header for multipart/form-data
- [ ] File: `homexa-mobile/src/services/chat.service.ts`
- [ ] Test: Upload file manually to verify endpoint

### Step 5: Create AttachmentDisplay Component
- [ ] Create `homexa-mobile/src/features/chat/components/AttachmentDisplay.tsx`
- [ ] Display images with preview
- [ ] Display documents with icons
- [ ] Add fullscreen image modal
- [ ] Test: Component renders correctly

### Step 6: Update MessageBubble
- [ ] Add `attachments?` prop
- [ ] Render `AttachmentDisplay` component
- [ ] Adjust layout for messages with attachments
- [ ] File: `homexa-mobile/src/features/chat/components/MessageBubble.tsx`

### Step 7: Update ChatDetailScreen
- [ ] Update `handleSendMessage` to accept attachments
- [ ] Pass attachments to `chatService.sendMessage`
- [ ] Map attachments from API response
- [ ] Pass attachments to MessageBubble
- [ ] File: `homexa-mobile/src/features/chat/screens/ChatDetailScreen.tsx`

### Step 8: Testing
- [ ] Image picker opens and allows selection
- [ ] Multiple images can be selected
- [ ] Preview shows selected images
- [ ] Can remove images before sending
- [ ] Files upload successfully
- [ ] Images display in message bubbles
- [ ] Fullscreen image view works
- [ ] File size validation works
- [ ] Error handling works

---

## Phase 3: Socket.IO Integration (Estimated: 5-6 hours)

### Step 1: Install Dependencies
- [ ] Run `npm install socket.io-client`
- [ ] Verify package.json updated
- [ ] No conflicts with existing dependencies

### Step 2: Create Socket Service
- [ ] Create `homexa-mobile/src/services/socket.service.ts`
- [ ] Implement singleton pattern
- [ ] Add `connect()` method with userId and token
- [ ] Add `disconnect()` method
- [ ] Add `on()`, `off()`, `emit()` methods
- [ ] Add connection status tracking
- [ ] Add auto-reconnect logic
- [ ] Test: Service initializes correctly

### Step 3: Create Socket Context
- [ ] Create `homexa-mobile/src/contexts/SocketContext.tsx`
- [ ] Implement SocketProvider component
- [ ] Initialize socket when user is authenticated
- [ ] Cleanup on logout/unmount
- [ ] Export `useSocket` hook
- [ ] Test: Context provides socket instance

### Step 4: Initialize in App Root
- [ ] Wrap app with `<SocketProvider>` in `_layout.tsx`
- [ ] Verify socket connects on app start (if logged in)
- [ ] File: `homexa-mobile/app/_layout.tsx`

### Step 5: Add Listeners in ChatDetailScreen
- [ ] Import `socketService` and `useSocket`
- [ ] Add `new_message` listener
- [ ] Add `message_read` listener
- [ ] Add `message_deleted_for_all` listener
- [ ] Add `message_deleted_for_me` listener
- [ ] Implement handler functions
- [ ] Cleanup listeners on unmount
- [ ] Prevent duplicate messages
- [ ] File: `homexa-mobile/src/features/chat/screens/ChatDetailScreen.tsx`

### Step 6: Add Listeners in ChatListScreen
- [ ] Import `socketService`
- [ ] Add `chat_updated` listener
- [ ] Add `new_chat` listener
- [ ] Refresh chat list on events
- [ ] Cleanup listeners on unmount
- [ ] File: `homexa-mobile/src/features/chat/screens/ChatListScreen.tsx`

### Step 7: Optimize Message Sending
- [ ] Update `handleSendMessage` in ChatDetailScreen
- [ ] Handle socket events for sent messages
- [ ] Avoid duplicate messages from socket
- [ ] File: `homexa-mobile/src/features/chat/screens/ChatDetailScreen.tsx`

### Step 8: Testing
- [ ] Socket connects on app start (check logs)
- [ ] New messages appear in real-time (test with 2 devices/clients)
- [ ] Read receipts update in real-time
- [ ] Deleted messages update in real-time
- [ ] Chat list updates on new chats
- [ ] Reconnection works after network loss
- [ ] No duplicate messages on reconnect
- [ ] Socket disconnects on logout
- [ ] Socket reconnects on login

---

## Configuration Checklist

### Environment Variables
- [ ] Verify `API_BASE_URL` in `homexa-mobile/src/config/env.ts`
- [ ] Socket URL should be base URL (same as API, without `/api`)
- [ ] Test connection with correct URL

### Backend Verification
- [ ] Backend socket server is running
- [ ] CORS allows mobile app origin
- [ ] Socket authentication works (userId in handshake.auth)
- [ ] Socket events are being emitted correctly

---

## Final Testing

### Integration Tests
- [ ] All three features work together
- [ ] Delete message works with attachments
- [ ] Socket updates work with file attachments
- [ ] No conflicts between features

### Performance Tests
- [ ] No memory leaks (socket listeners cleaned up)
- [ ] No excessive reconnections
- [ ] File uploads don't block UI
- [ ] Large file handling works correctly

### Edge Cases
- [ ] Network loss/recovery
- [ ] App backgrounding/foregrounding
- [ ] Multiple rapid messages
- [ ] Concurrent file uploads
- [ ] Very large files (test size limits)

---

## Documentation
- [ ] Update README if needed
- [ ] Add code comments for complex logic
- [ ] Document socket events in code
- [ ] Add error handling documentation

---

## 🎉 Completion

Once all checkboxes are checked, you should have:
- ✅ Delete message functionality
- ✅ File attachment support (send & display)
- ✅ Real-time updates via Socket.IO

---

## Troubleshooting Tips

### Socket Not Connecting?
- Check `API_BASE_URL` format (should be base URL, not `/api` endpoint)
- Verify `userId` is passed in `handshake.auth`
- Check backend logs for connection errors
- Test with browser console if needed

### Files Not Uploading?
- Verify FormData format (React Native specific)
- Check file size limits (backend may have limits)
- Ensure Content-Type header is correct
- Test endpoint with Postman first

### Messages Duplicating?
- Check message ID before adding to state
- Remove optimistic UI update if socket receives message
- Use message ID as key in React list

### Performance Issues?
- Verify socket listeners are cleaned up
- Check for memory leaks (use React DevTools)
- Limit file preview sizes
- Optimize image rendering

