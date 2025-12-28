# Chat Features Implementation Guide

This guide provides step-by-step instructions for implementing:
1. Delete Message Functionality
2. File Attachments (Send & Display)
3. Socket.IO Real-time Communication

---

## 📋 Table of Contents

1. [Delete Message Implementation](#1-delete-message-implementation)
2. [File Attachments Implementation](#2-file-attachments-implementation)
3. [Socket.IO Integration](#3-socketio-integration)

---

## 1. Delete Message Implementation

### Subtask 1.1: Update ChatMessage Type
**File:** `homexa-mobile/src/features/chat/types/index.ts`

Add `isDeleted` and `deletedFor` fields to track deletion state:
```typescript
export interface ChatMessage {
  // ... existing fields
  isDeleted?: boolean; // Frontend flag for deleted messages
  deletedFor?: string[]; // Users who deleted this message
}
```

### Subtask 1.2: Add Delete Message to Chat Service
**File:** `homexa-mobile/src/services/chat.service.ts`

Add the delete message method:
```typescript
async deleteMessage(messageId: string, deleteForAll: boolean = false) {
  try {
    const response = await api.delete(`/api/chats/messages/${messageId}`, {
      params: { deleteForAll }
    });
    return response.data;
  } catch (error) {
    throw error;
  }
}
```

### Subtask 1.3: Update MessageBubble Component
**File:** `homexa-mobile/src/features/chat/components/MessageBubble.tsx`

Add:
- Long press handler for delete menu
- `isDeleted` prop to show "Message deleted" placeholder
- Delete button/action sheet UI

### Subtask 1.4: Add Delete Handler in ChatDetailScreen
**File:** `homexa-mobile/src/features/chat/screens/ChatDetailScreen.tsx`

Add:
- `handleDeleteMessage` function
- Optimistic UI update (remove message immediately)
- Error handling with rollback on failure

---

## 2. File Attachments Implementation

### Subtask 2.1: Update ChatMessage Type for Attachments
**File:** `homexa-mobile/src/features/chat/types/index.ts`

Add attachments field:
```typescript
export interface MessageAttachment {
  id: string;
  messageId: string;
  fileUrl: string;
  fileType: string;
}

export interface ChatMessage {
  // ... existing fields
  attachments?: MessageAttachment[];
}
```

### Subtask 2.2: Create File Picker Utility
**File:** `homexa-mobile/src/utils/filePicker.ts` (NEW FILE)

Create utility using `expo-image-picker` for:
- Image selection
- File type validation
- File size limits
- Multiple file selection

### Subtask 2.3: Update ChatInputBar Component
**File:** `homexa-mobile/src/features/chat/components/ChatInputBar.tsx`

Add:
- File attachment button handler
- Selected files preview
- Remove file from selection
- Update `onSendMessage` to accept attachments

### Subtask 2.4: Update Chat Service for File Upload
**File:** `homexa-mobile/src/services/chat.service.ts`

Update `sendMessage` to support FormData:
- Use `FormData` for multipart/form-data requests
- Append files, product_id, content, chatId
- Set proper headers for file upload

### Subtask 2.5: Create AttachmentDisplay Component
**File:** `homexa-mobile/src/features/chat/components/AttachmentDisplay.tsx` (NEW FILE)

Component to display:
- Images (with preview/fullscreen)
- Documents (with download/open)
- File icons based on type
- File names and sizes

### Subtask 2.6: Update MessageBubble Component
**File:** `homexa-mobile/src/features/chat/components/MessageBubble.tsx`

Add:
- `attachments` prop
- Render `AttachmentDisplay` component
- Layout adjustments for messages with attachments

### Subtask 2.7: Update ChatDetailScreen
**File:** `homexa-mobile/src/features/chat/screens/ChatDetailScreen.tsx`

Update:
- `handleSendMessage` to handle attachments
- Message mapping to include attachments from API
- Pass attachments to MessageBubble

---

## 3. Socket.IO Integration

### Subtask 3.1: Install Socket.IO Client
**Command:**
```bash
cd homexa-mobile
npm install socket.io-client
```

### Subtask 3.2: Create Socket Service
**File:** `homexa-mobile/src/services/socket.service.ts` (NEW FILE)

Create singleton service with:
- Connection management
- Authentication (pass userId in handshake)
- Event listeners (new_message, message_read, chat_updated, etc.)
- Connection state management (connected, disconnected, error)
- Auto-reconnect logic

### Subtask 3.3: Create Socket Context
**File:** `homexa-mobile/src/contexts/SocketContext.tsx` (NEW FILE)

React context to:
- Provide socket instance to components
- Manage connection lifecycle
- Handle authentication on mount
- Cleanup on unmount

### Subtask 3.4: Initialize Socket in App Root
**File:** `homexa-mobile/app/_layout.tsx`

Wrap app with SocketProvider:
- Initialize socket when user is authenticated
- Cleanup when user logs out

### Subtask 3.5: Add Socket Listeners in ChatDetailScreen
**File:** `homexa-mobile/src/features/chat/screens/ChatDetailScreen.tsx`

Add:
- `new_message` listener to add messages in real-time
- `message_read` listener to update read status
- `message_deleted_for_all` listener to remove deleted messages
- `message_deleted_for_me` listener for local deletion
- Cleanup listeners on unmount

### Subtask 3.6: Add Socket Listeners in ChatListScreen
**File:** `homexa-mobile/src/features/chat/screens/ChatListScreen.tsx`

Add:
- `chat_updated` listener to refresh chat list
- `new_chat` listener for new chat notifications
- Update last message and unread count in real-time

### Subtask 3.7: Update Message Sending
**File:** `homexa-mobile/src/features/chat/screens/ChatDetailScreen.tsx`

Optimize `handleSendMessage`:
- Keep optimistic UI update
- Remove duplicate message if socket receives it
- Handle socket events for sent messages

### Subtask 3.8: Add Connection Status Indicator (Optional)
**File:** `homexa-mobile/src/components/SocketStatusIndicator.tsx` (NEW FILE)

Optional component to show:
- Connected/disconnected status
- Reconnecting indicator
- Network status feedback

---

## 🔧 Implementation Order Recommendation

1. **Phase 1: Delete Message** (Simplest, no new dependencies)
   - Subtasks 1.1 → 1.2 → 1.3 → 1.4

2. **Phase 2: File Attachments** (Requires expo-image-picker)
   - Subtasks 2.1 → 2.2 → 2.3 → 2.4 → 2.5 → 2.6 → 2.7

3. **Phase 3: Socket.IO** (Most complex, requires all dependencies)
   - Subtasks 3.1 → 3.2 → 3.3 → 3.4 → 3.5 → 3.6 → 3.7 → 3.8

---

## 🐛 Debugging Tips

### Delete Message
- Check network tab for DELETE request
- Verify messageId is correct
- Check deleteForAll parameter
- Handle 404 (message already deleted)

### File Attachments
- Check file size limits (backend may have limits)
- Verify FormData structure
- Test with different file types
- Check CORS settings for file uploads
- Verify file URLs in response

### Socket.IO
- Check connection in socket service logs
- Verify userId is passed in handshake auth
- Test reconnection logic
- Check backend socket event names match frontend
- Verify socket rooms (userId-based rooms)
- Test with multiple clients simultaneously

---

## 📝 Notes

- Backend already supports all these features
- Socket.IO server is already configured in backend
- File uploads use multipart/form-data
- Socket authentication uses userId in handshake.auth
- All socket events are userId-based (rooms)

