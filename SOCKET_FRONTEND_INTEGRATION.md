# Socket.IO Frontend Integration Guide

This guide explains how to integrate the chat socket functionality on the frontend using Socket.IO.

## Table of Contents

1. [Installation](#installation)
2. [Connection Setup](#connection-setup)
3. [Socket Events](#socket-events)
4. [Complete Example](#complete-example)
5. [Best Practices](#best-practices)
6. [Error Handling](#error-handling)
7. [TypeScript Types](#typescript-types)

---

## Installation

### React/Next.js Example

```bash
npm install socket.io-client
# or
yarn add socket.io-client
```

### Vanilla JavaScript

```html
<script src="https://cdn.socket.io/4.5.4/socket.io.min.js"></script>
```

---

## Connection Setup

### Basic Connection

The socket server requires authentication via `userId` in the handshake auth object.

```javascript
import { io } from 'socket.io-client';

// Replace with your backend URL
const SOCKET_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3000';

// Get userId from your auth system (JWT token, user context, etc.)
const userId = getCurrentUserId(); // Your function to get current user ID

const socket = io(SOCKET_URL, {
  auth: {
    userId: userId, // Required: Server uses this to join user-specific room
  },
  transports: ['websocket', 'polling'], // Fallback to polling if websocket fails
  reconnection: true,
  reconnectionDelay: 1000,
  reconnectionAttempts: 5,
});
```

### React Hook Example

```javascript
import { useEffect, useState } from 'react';
import { io, Socket } from 'socket.io-client';

const useSocket = (userId: string | null) => {
  const [socket, setSocket] = useState<Socket | null>(null);
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    if (!userId) return;

    const SOCKET_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3000';

    const newSocket = io(SOCKET_URL, {
      auth: {
        userId: userId,
      },
      transports: ['websocket', 'polling'],
      reconnection: true,
      reconnectionDelay: 1000,
      reconnectionAttempts: 5,
    });

    newSocket.on('connect', () => {
      console.log('Socket connected:', newSocket.id);
      setIsConnected(true);
    });

    newSocket.on('disconnect', () => {
      console.log('Socket disconnected');
      setIsConnected(false);
    });

    newSocket.on('connect_error', (error) => {
      console.error('Socket connection error:', error);
      setIsConnected(false);
    });

    setSocket(newSocket);

    // Cleanup on unmount
    return () => {
      newSocket.close();
    };
  }, [userId]);

  return { socket, isConnected };
};
```

---

## Socket Events

The server emits the following events to connected clients:

### 1. `new_message`

Emitted when a new message is received in a chat.

**When it fires:** When someone sends you a message

**Payload:**
```typescript
{
  chatId: string;
  message: {
    id: string;
    chatId: string;
    senderId: string;
    content: string;
    createdAt: Date;
    readAt: Date | null;
    attachments: Array<{
      id: string;
      messageId: string;
      fileUrl: string;
      fileType: string;
    }>;
  };
}
```

**Usage:**
```javascript
socket.on('new_message', (data) => {
  console.log('New message received:', data);

  const { chatId, message } = data;

  // Update your chat UI with the new message
  if (currentChatId === chatId) {
    // Add message to current chat
    setMessages(prev => [...prev, message]);

    // Play notification sound
    playNotificationSound();
  } else {
    // Update chat list with new message indicator
    updateChatUnreadCount(chatId, 1);
  }
});
```

---

### 2. `chat_updated`

Emitted when a chat needs to be refreshed in the chat list.

**When it fires:** After a message is sent (both sender and receiver get this)

**Payload:**
```typescript
{
  chatId: string;
}
```

**Usage:**
```javascript
socket.on('chat_updated', (data) => {
  console.log('Chat updated:', data.chatId);

  // Refetch chat list or update specific chat in the list
  refreshChatList();

  // Or update specific chat
  updateChatInList(data.chatId);
});
```

---

### 3. `new_chat`

Emitted when a new chat is created (only sent to the seller when customer initiates).

**When it fires:** When a customer sends the first message to a seller

**Payload:**
```typescript
{
  chatId: string;
  productId: string;
}
```

**Usage:**
```javascript
socket.on('new_chat', (data) => {
  console.log('New chat created:', data);

  const { chatId, productId } = data;

  // Add new chat to chat list
  addNewChatToList({
    chatId,
    productId,
    // Fetch full chat details from API
  });

  // Show notification to seller
  showNotification('You have a new chat!');
});
```

---

### 4. `message_read`

Emitted when messages in a chat are marked as read.

**When it fires:** When the other participant reads your messages

**Payload:**
```typescript
{
  chatId: string;
  readerId: string; // ID of the user who read the messages
}
```

**Usage:**
```javascript
socket.on('message_read', (data) => {
  console.log('Messages read:', data);

  const { chatId, readerId } = data;

  // Update message read status in UI
  if (currentChatId === chatId) {
    // Mark all your messages as read
    updateMessageReadStatus(chatId, readerId);
  }
});
```

---

### 5. `message_deleted_for_all`

Emitted when a message is deleted for everyone (both participants).

**When it fires:** When someone deletes a message with `deleteForAll: true`

**Payload:**
```typescript
{
  messageId: string;
  chatId: string;
}
```

**Usage:**
```javascript
socket.on('message_deleted_for_all', (data) => {
  console.log('Message deleted for all:', data);

  const { messageId, chatId } = data;

  // Remove message from UI
  if (currentChatId === chatId) {
    removeMessageFromUI(messageId);
  }
});
```

---

### 6. `message_deleted_for_me`

Emitted when you delete a message for yourself only.

**When it fires:** When you delete a message with `deleteForAll: false`

**Payload:**
```typescript
{
  messageId: string;
  chatId: string;
}
```

**Usage:**
```javascript
socket.on('message_deleted_for_me', (data) => {
  console.log('Message deleted for me:', data);

  const { messageId, chatId } = data;

  // Hide message in UI (soft delete)
  if (currentChatId === chatId) {
    hideMessageInUI(messageId);
  }
});
```

---

## Complete Example

### React Component with Socket Integration

```javascript
import { useEffect, useState, useRef } from 'react';
import { io, Socket } from 'socket.io-client';

const ChatComponent = ({ userId, currentChatId }) => {
  const [socket, setSocket] = useState<Socket | null>(null);
  const [messages, setMessages] = useState([]);
  const [chats, setChats] = useState([]);
  const socketRef = useRef<Socket | null>(null);

  useEffect(() => {
    if (!userId) return;

    const SOCKET_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3000';

    const newSocket = io(SOCKET_URL, {
      auth: { userId },
      transports: ['websocket', 'polling'],
      reconnection: true,
    });

    // Connection events
    newSocket.on('connect', () => {
      console.log('Connected to socket server');
    });

    newSocket.on('disconnect', () => {
      console.log('Disconnected from socket server');
    });

    // Chat events
    newSocket.on('new_message', (data) => {
      const { chatId, message } = data;

      if (currentChatId === chatId) {
        setMessages(prev => [...prev, message]);
        // Auto-scroll to bottom
        scrollToBottom();
      }

      // Update chat list
      updateChatInList(chatId, { lastMessage: message.content });
    });

    newSocket.on('chat_updated', ({ chatId }) => {
      // Refresh chat list
      fetchChatList();
    });

    newSocket.on('new_chat', ({ chatId, productId }) => {
      // Add new chat to list
      fetchChatList();
      showNotification('New chat started!');
    });

    newSocket.on('message_read', ({ chatId, readerId }) => {
      if (currentChatId === chatId) {
        // Update read receipts
        updateReadReceipts(chatId, readerId);
      }
    });

    newSocket.on('message_deleted_for_all', ({ messageId, chatId }) => {
      if (currentChatId === chatId) {
        setMessages(prev => prev.filter(msg => msg.id !== messageId));
      }
    });

    newSocket.on('message_deleted_for_me', ({ messageId, chatId }) => {
      if (currentChatId === chatId) {
        // Hide message (soft delete)
        setMessages(prev =>
          prev.map(msg =>
            msg.id === messageId ? { ...msg, deleted: true } : msg
          )
        );
      }
    });

    socketRef.current = newSocket;
    setSocket(newSocket);

    return () => {
      newSocket.close();
    };
  }, [userId]);

  const updateChatInList = (chatId, updates) => {
    setChats(prev =>
      prev.map(chat =>
        chat.id === chatId ? { ...chat, ...updates } : chat
      )
    );
  };

  const fetchChatList = async () => {
    // Fetch from your API
    const response = await fetch('/api/chats', {
      headers: {
        Authorization: `Bearer ${getToken()}`,
      },
    });
    const data = await response.json();
    setChats(data);
  };

  return (
    <div>
      {/* Your chat UI */}
    </div>
  );
};
```

---

## Best Practices

### 1. **Connection Management**

- Only connect when user is authenticated
- Disconnect on logout
- Handle reconnection gracefully

```javascript
// Connect on login
const handleLogin = async () => {
  await loginUser();
  initializeSocket(userId);
};

// Disconnect on logout
const handleLogout = () => {
  socket?.disconnect();
  setSocket(null);
};
```

### 2. **Event Cleanup**

Always remove event listeners to prevent memory leaks:

```javascript
useEffect(() => {
  const handleNewMessage = (data) => {
    // Handle message
  };

  socket?.on('new_message', handleNewMessage);

  return () => {
    socket?.off('new_message', handleNewMessage);
  };
}, [socket]);
```

### 3. **State Management**

Consider using a state management library (Redux, Zustand) for socket state:

```javascript
// Zustand example
import create from 'zustand';

const useSocketStore = create((set) => ({
  socket: null,
  isConnected: false,
  setSocket: (socket) => set({ socket }),
  setIsConnected: (isConnected) => set({ isConnected }),
}));
```

### 4. **Error Handling**

Always handle connection errors:

```javascript
socket.on('connect_error', (error) => {
  console.error('Connection error:', error);
  // Show user-friendly error message
  showError('Unable to connect to chat server. Please refresh.');
});

socket.on('error', (error) => {
  console.error('Socket error:', error);
});
```

### 5. **Reconnection Strategy**

Configure reconnection with exponential backoff:

```javascript
const socket = io(SOCKET_URL, {
  auth: { userId },
  reconnection: true,
  reconnectionDelay: 1000,
  reconnectionDelayMax: 5000,
  reconnectionAttempts: Infinity,
});
```

---

## Error Handling

### Common Errors

1. **Connection Refused**
   ```javascript
   socket.on('connect_error', (error) => {
     if (error.message === 'xhr poll error') {
       // Server is down or unreachable
       showError('Chat service unavailable');
     }
   });
   ```

2. **Authentication Failed**
   - Ensure `userId` is correctly passed in `auth`
   - Verify user is authenticated before connecting

3. **Reconnection Failures**
   ```javascript
   let reconnectAttempts = 0;

   socket.on('reconnect_attempt', () => {
     reconnectAttempts++;
     if (reconnectAttempts > 5) {
       showError('Connection lost. Please refresh the page.');
     }
   });

   socket.on('reconnect', () => {
     reconnectAttempts = 0;
     showSuccess('Reconnected to chat');
   });
   ```

---

## TypeScript Types

Create type definitions for better type safety:

```typescript
// types/socket.ts

export interface SocketAuth {
  userId: string;
}

export interface NewMessageEvent {
  chatId: string;
  message: {
    id: string;
    chatId: string;
    senderId: string;
    content: string;
    createdAt: Date;
    readAt: Date | null;
    attachments: Array<{
      id: string;
      messageId: string;
      fileUrl: string;
      fileType: string;
    }>;
  };
}

export interface ChatUpdatedEvent {
  chatId: string;
}

export interface NewChatEvent {
  chatId: string;
  productId: string;
}

export interface MessageReadEvent {
  chatId: string;
  readerId: string;
}

export interface MessageDeletedEvent {
  messageId: string;
  chatId: string;
}

// Socket event map
export interface ServerToClientEvents {
  new_message: (data: NewMessageEvent) => void;
  chat_updated: (data: ChatUpdatedEvent) => void;
  new_chat: (data: NewChatEvent) => void;
  message_read: (data: MessageReadEvent) => void;
  message_deleted_for_all: (data: MessageDeletedEvent) => void;
  message_deleted_for_me: (data: MessageDeletedEvent) => void;
}

export interface ClientToServerEvents {
  // Add client-to-server events if needed
}
```

Usage with types:

```typescript
import { io, Socket } from 'socket.io-client';
import { ServerToClientEvents, ClientToServerEvents, SocketAuth } from './types/socket';

const socket: Socket<ServerToClientEvents, ClientToServerEvents> = io(SOCKET_URL, {
  auth: { userId } as SocketAuth,
});

socket.on('new_message', (data) => {
  // data is typed as NewMessageEvent
  console.log(data.chatId, data.message);
});
```

---

## Testing Socket Connection

### Test Connection

```javascript
socket.on('connect', () => {
  console.log('✅ Socket connected:', socket.id);
});

socket.on('disconnect', () => {
  console.log('❌ Socket disconnected');
});

// Test emit (if server supports it)
socket.emit('ping', (response) => {
  console.log('Server response:', response);
});
```

### Debug Mode

Enable debug logs in development:

```javascript
// In browser console or add to your code
localStorage.debug = 'socket.io-client:*';
```

---

## API Endpoints Reference

The socket events work alongside these REST API endpoints:

- `GET /api/chats` - Get user's chat list
- `GET /api/chats/messages?chatId=xxx` - Get chat messages
- `POST /api/chats/messages` - Send a message
- `POST /api/chats/messages/read` - Mark messages as read
- `DELETE /api/chats/messages/:messageId` - Delete a message

---

## Support

For issues or questions:
1. Check server logs for socket connection errors
2. Verify `userId` is correctly passed in auth
3. Ensure CORS is configured on the server
4. Check network tab for WebSocket connection status

---

## Quick Reference

| Event | When Fired | Payload |
|-------|------------|---------|
| `new_message` | New message received | `{ chatId, message }` |
| `chat_updated` | Chat list needs refresh | `{ chatId }` |
| `new_chat` | New chat created (seller only) | `{ chatId, productId }` |
| `message_read` | Messages marked as read | `{ chatId, readerId }` |
| `message_deleted_for_all` | Message deleted for everyone | `{ messageId, chatId }` |
| `message_deleted_for_me` | Message deleted for me | `{ messageId, chatId }` |

---

**Last Updated:** 2024
**Server URL:** Configure via `NEXT_PUBLIC_API_URL` or use `http://localhost:3000` by default
