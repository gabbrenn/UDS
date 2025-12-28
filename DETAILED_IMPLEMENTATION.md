# Detailed Implementation Guide with Code Examples

This document provides detailed code examples and explanations for each subtask.

---

## 1. Delete Message Implementation

### Why This Approach?
- **Optimistic UI**: Remove message immediately for better UX
- **Error Handling**: Rollback if deletion fails
- **Two Modes**: Delete for self only (default) or delete for everyone (if sender)

### Subtask 1.1: Update Types

```typescript
// homexa-mobile/src/features/chat/types/index.ts
export interface ChatMessage {
  id: string;
  senderId: string;
  senderName: string;
  senderAvatar?: string;
  message: string;
  timestamp: Date;
  isRead: boolean;
  productId?: string;
  productImage?: string;
  productName?: string;
  isDeleted?: boolean; // NEW: Frontend flag
  deletedFor?: string[]; // NEW: Users who deleted this message
}
```

**Explanation**: These fields help track deletion state on the frontend and match backend response structure.

---

### Subtask 1.2: Add Delete Service Method

```typescript
// homexa-mobile/src/services/chat.service.ts
export const chatService = {
  // ... existing methods

  async deleteMessage(messageId: string, deleteForAll: boolean = false) {
    try {
      console.log('=== DELETE MESSAGE REQUEST ===');
      console.log('Endpoint: DELETE /api/chats/messages/:messageId');
      console.log('Parameters:', { messageId, deleteForAll });

      const response = await api.delete(`/api/chats/messages/${messageId}`, {
        params: { deleteForAll }
      });

      console.log('=== DELETE MESSAGE RESPONSE ===');
      console.log('Status: Success');
      console.log('Response:', JSON.stringify(response, null, 2));
      return response.data;
    } catch (error) {
      throw error;
    }
  }
};
```

**Explanation**: 
- `deleteForAll: false` = Delete for yourself only (default)
- `deleteForAll: true` = Delete for everyone (only if you're the sender)
- Backend endpoint: `DELETE /api/chats/messages/:messageId?deleteForAll=true/false`

---

### Subtask 1.3: Update MessageBubble Component

```typescript
// homexa-mobile/src/features/chat/components/MessageBubble.tsx
import React from 'react';
import { View, Text, StyleSheet, Image, TouchableOpacity, Alert } from 'react-native';
// ... other imports

interface MessageBubbleProps {
  message: string;
  isSent: boolean;
  timestamp: Date;
  isRead?: boolean;
  senderAvatar?: string;
  senderName?: string;
  messageId?: string; // NEW: Needed for delete
  isDeleted?: boolean; // NEW: Flag for deleted messages
  onDelete?: (messageId: string, deleteForAll: boolean) => void; // NEW: Delete handler
}

export const MessageBubble: React.FC<MessageBubbleProps> = ({
  message,
  isSent,
  timestamp,
  isRead,
  senderAvatar,
  senderName,
  messageId,
  isDeleted = false,
  onDelete,
}) => {
  // ... existing code

  const handleLongPress = () => {
    if (!isSent || !messageId || isDeleted) return; // Only sender can delete

    Alert.alert(
      'Delete Message',
      'Do you want to delete this message?',
      [
        {
          text: 'Cancel',
          style: 'cancel',
        },
        {
          text: 'Delete for Me',
          style: 'destructive',
          onPress: () => onDelete?.(messageId, false),
        },
        {
          text: 'Delete for Everyone',
          style: 'destructive',
          onPress: () => onDelete?.(messageId, true),
        },
      ],
      { cancelable: true }
    );
  };

  return (
    <TouchableOpacity
      activeOpacity={0.7}
      onLongPress={handleLongPress}
      disabled={!isSent || isDeleted}
    >
      <View style={[styles.container, isSent ? styles.sentContainer : styles.receivedContainer]}>
        {/* ... existing avatar code */}

        <View style={[styles.bubble, /* ... existing styles */]}>
          {isDeleted ? (
            <Text style={[styles.deletedMessage, { color: themeColors.textSecondary }]}>
              Message deleted
            </Text>
          ) : (
            <>
              {/* ... existing message content */}
            </>
          )}
        </View>
      </View>
    </TouchableOpacity>
  );
};

const styles = StyleSheet.create({
  // ... existing styles
  deletedMessage: {
    fontStyle: 'italic',
    fontSize: theme.fontSize.sm,
  },
});
```

**Explanation**:
- Long press triggers delete menu (iOS/Android standard)
- Only sender can delete (check `isSent`)
- Two options: Delete for me vs Delete for everyone
- Show "Message deleted" placeholder if `isDeleted` is true

---

### Subtask 1.4: Add Delete Handler in ChatDetailScreen

```typescript
// homexa-mobile/src/features/chat/screens/ChatDetailScreen.tsx
// ... existing imports

export default function ChatDetailScreen() {
  // ... existing state and hooks

  const handleDeleteMessage = async (messageId: string, deleteForAll: boolean) => {
    // Optimistic UI: Remove message immediately
    const messageToDelete = messages.find(msg => msg.id === messageId);
    if (!messageToDelete) return;

    // Store original state for rollback
    const originalMessages = [...messages];

    // Update UI optimistically
    if (deleteForAll) {
      // Remove from list completely
      setMessages(prevMessages => prevMessages.filter(msg => msg.id !== messageId));
    } else {
      // Mark as deleted (still visible but shows "Message deleted")
      setMessages(prevMessages =>
        prevMessages.map(msg =>
          msg.id === messageId ? { ...msg, isDeleted: true } : msg
        )
      );
    }

    try {
      await chatService.deleteMessage(messageId, deleteForAll);
      // Success - UI already updated
    } catch (error) {
      console.error('Error deleting message:', error);
      
      // Rollback on error
      setMessages(originalMessages);
      
      Alert.alert('Error', 'Failed to delete message. Please try again.');
    }
  };

  // Update MessageBubble rendering:
  return (
    // ... existing JSX
    <MessageBubble
      key={msg.id}
      messageId={msg.id} // NEW
      message={msg.message}
      isSent={msg.senderId === user?.id}
      timestamp={msg.timestamp}
      isRead={msg.isRead}
      isDeleted={msg.isDeleted} // NEW
      onDelete={handleDeleteMessage} // NEW
      senderAvatar={/* ... */}
      senderName={/* ... */}
    />
  );
}
```

**Explanation**:
- **Optimistic UI**: Update immediately for better UX
- **Rollback**: Restore original state if API call fails
- **Two deletion modes**: 
  - `deleteForAll: false` → Show "Message deleted" (soft delete)
  - `deleteForAll: true` → Remove completely (hard delete)

---

## 2. File Attachments Implementation

### Why This Approach?
- **FormData**: Required for multipart file uploads
- **Preview**: Show selected files before sending
- **Type Safety**: Proper TypeScript interfaces
- **Progressive Enhancement**: Works with or without attachments

### Subtask 2.1: Update Types

```typescript
// homexa-mobile/src/features/chat/types/index.ts
export interface MessageAttachment {
  id: string;
  messageId: string;
  fileUrl: string;
  fileType: string;
  fileName?: string; // Optional: for display
  fileSize?: number; // Optional: for display
}

export interface ChatMessage {
  // ... existing fields
  attachments?: MessageAttachment[];
}
```

---

### Subtask 2.2: Create File Picker Utility

```typescript
// homexa-mobile/src/utils/filePicker.ts (NEW FILE)
import * as ImagePicker from 'expo-image-picker';
import * as DocumentPicker from 'expo-document-picker'; // If you want document support

export interface PickedFile {
  uri: string;
  type: string;
  name: string;
  size?: number;
}

const MAX_FILE_SIZE = 10 * 1024 * 1024; // 10MB
const ALLOWED_IMAGE_TYPES = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];

export const pickImage = async (): Promise<PickedFile | null> => {
  try {
    // Request permissions
    const { status } = await ImagePicker.requestMediaLibraryPermissionsAsync();
    if (status !== 'granted') {
      // Note: Import Alert from 'react-native' if using in this file
      // For now, return null and handle alert in component
      return null;
    }

    // Launch image picker
    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      allowsEditing: false,
      quality: 0.8,
      allowsMultipleSelection: true, // Enable multiple selection
    });

    if (result.canceled) return null;

    // Handle multiple images
    if (result.assets && result.assets.length > 0) {
      return result.assets.map(asset => ({
        uri: asset.uri,
        type: asset.mimeType || 'image/jpeg',
        name: asset.fileName || `image_${Date.now()}.jpg`,
        size: asset.fileSize,
      }));
    }

    return null;
  } catch (error) {
    console.error('Error picking image:', error);
    return null;
  }
};

export const validateFile = (file: PickedFile): string | null => {
  if (file.size && file.size > MAX_FILE_SIZE) {
    return 'File size exceeds 10MB limit';
  }

  if (file.type && !ALLOWED_IMAGE_TYPES.includes(file.type)) {
    return 'Unsupported file type. Please use JPEG, PNG, GIF, or WebP';
  }

  return null;
};
```

**Note**: If you want document support (PDF, DOCX, etc.), you'll need `expo-document-picker`:
```bash
npm install expo-document-picker
```

---

### Subtask 2.3: Update ChatInputBar Component

```typescript
// homexa-mobile/src/features/chat/components/ChatInputBar.tsx
import React, { useState } from 'react';
import { View, TextInput, TouchableOpacity, StyleSheet, Image, ScrollView } from 'react-native';
import { pickImage, validateFile, PickedFile } from '@/src/utils/filePicker';

interface ChatInputBarProps {
  onSendMessage: (message: string, attachments?: PickedFile[]) => void; // Updated signature
  placeholder?: string;
  isLoading?: boolean;
}

export const ChatInputBar: React.FC<ChatInputBarProps> = ({
  onSendMessage,
  placeholder = 'Type a message...',
  isLoading = false,
}) => {
  const [message, setMessage] = useState('');
  const [selectedFiles, setSelectedFiles] = useState<PickedFile[]>([]); // NEW

  const handleAttach = async () => {
    const files = await pickImage();
    if (!files) return;

    // Validate files
    const filesArray = Array.isArray(files) ? files : [files];
    const validFiles: PickedFile[] = [];
    
    for (const file of filesArray) {
      const error = validateFile(file);
      if (error) {
        // Show error in component (import Alert from 'react-native')
        console.error('Invalid file:', error);
        continue;
      }
      validFiles.push(file);
    }

    setSelectedFiles(prev => [...prev, ...validFiles]);
  };

  const removeFile = (index: number) => {
    setSelectedFiles(prev => prev.filter((_, i) => i !== index));
  };

  const handleSend = () => {
    if (!message.trim() && selectedFiles.length === 0) return;

    onSendMessage(message.trim(), selectedFiles.length > 0 ? selectedFiles : undefined);
    setMessage('');
    setSelectedFiles([]); // Clear attachments after sending
  };

  return (
    <View style={[styles.container, { backgroundColor: themeColors.bg }]}>
      {/* File Preview Section */}
      {selectedFiles.length > 0 && (
        <ScrollView horizontal style={styles.previewContainer}>
          {selectedFiles.map((file, index) => (
            <View key={index} style={styles.previewItem}>
              <Image source={{ uri: file.uri }} style={styles.previewImage} />
              <TouchableOpacity
                style={styles.removeButton}
                onPress={() => removeFile(index)}
              >
                <Ionicons name="close-circle" size={20} color={themeColors.red} />
              </TouchableOpacity>
            </View>
          ))}
        </ScrollView>
      )}

      <View style={[styles.inputContainer, { backgroundColor: themeColors.card }]}>
        <TextInput
          // ... existing props
        />

        <TouchableOpacity
          style={styles.attachButton}
          onPress={handleAttach}
          disabled={isLoading}
        >
          <Ionicons name="attach" size={20} color={themeColors.blue} />
        </TouchableOpacity>
      </View>

      <TouchableOpacity
        style={[styles.sendButton, {
          backgroundColor: themeColors.blue,
          opacity: (message.trim() || selectedFiles.length > 0) && !isLoading ? 1 : 0.5,
        }]}
        onPress={handleSend}
        disabled={(!message.trim() && selectedFiles.length === 0) || isLoading}
      >
        {/* ... existing send button */}
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  // ... existing styles
  previewContainer: {
    paddingHorizontal: 12,
    paddingTop: 8,
    maxHeight: 100,
  },
  previewItem: {
    marginRight: 8,
    position: 'relative',
  },
  previewImage: {
    width: 80,
    height: 80,
    borderRadius: 8,
  },
  removeButton: {
    position: 'absolute',
    top: -8,
    right: -8,
    backgroundColor: themeColors.bg,
    borderRadius: 12,
  },
});
```

---

### Subtask 2.4: Update Chat Service for File Upload

```typescript
// homexa-mobile/src/services/chat.service.ts
import { PickedFile } from '@/src/utils/filePicker';

export const chatService = {
  // ... existing methods

  async sendMessage(
    productId: string,
    content: string,
    attachments?: PickedFile[],
    chatId?: string
  ) {
    try {
      const formData = new FormData();

      // Append text fields
      if (chatId) {
        formData.append('chatId', chatId);
      }
      if (productId) {
        formData.append('product_id', productId);
      }
      if (content) {
        formData.append('content', content);
      }

      // Append files
      if (attachments && attachments.length > 0) {
        attachments.forEach((file, index) => {
          // React Native FormData expects specific format
          formData.append('file', {
            uri: file.uri,
            type: file.type,
            name: file.name,
          } as any);
        });
      }

      const response = await api.post('/api/chats/messages', formData, {
        headers: {
          'Content-Type': 'multipart/form-data',
        },
      });

      return response.data;
    } catch (error) {
      throw error;
    }
  },
};
```

**Important Notes**:
- React Native FormData format is different from web
- Backend expects `file` field name (check backend controller)
- Content-Type header is set automatically by axios for FormData, but explicitly setting it helps

---

### Subtask 2.5: Create AttachmentDisplay Component

```typescript
// homexa-mobile/src/features/chat/components/AttachmentDisplay.tsx (NEW FILE)
import React, { useState } from 'react';
import { View, Image, TouchableOpacity, StyleSheet, Modal, Text } from 'react-native';
import { MessageAttachment } from '../types';
import { Ionicons } from '@expo/vector-icons';
import { getThemeColors } from '@/src/constants/colors';
import { useTheme } from '@/src/contexts/ThemeContext';

interface AttachmentDisplayProps {
  attachments: MessageAttachment[];
}

export const AttachmentDisplay: React.FC<AttachmentDisplayProps> = ({ attachments }) => {
  const { isDark } = useTheme();
  const themeColors = getThemeColors(isDark);
  const [selectedImage, setSelectedImage] = useState<string | null>(null);

  const isImage = (fileType: string) => {
    return fileType.startsWith('image/');
  };

  const getFileIcon = (fileType: string) => {
    if (fileType.includes('pdf')) return 'document-text';
    if (fileType.includes('word')) return 'document';
    return 'document-attach';
  };

  if (!attachments || attachments.length === 0) return null;

  return (
    <>
      <View style={styles.container}>
        {attachments.map((attachment) => {
          if (isImage(attachment.fileType)) {
            return (
              <TouchableOpacity
                key={attachment.id}
                onPress={() => setSelectedImage(attachment.fileUrl)}
                style={styles.imageContainer}
              >
                <Image
                  source={{ uri: attachment.fileUrl }}
                  style={styles.image}
                  resizeMode="cover"
                />
              </TouchableOpacity>
            );
          }

          return (
            <TouchableOpacity
              key={attachment.id}
              style={[styles.documentContainer, { backgroundColor: themeColors.card }]}
            >
              <Ionicons
                name={getFileIcon(attachment.fileType)}
                size={32}
                color={themeColors.blue}
              />
              {attachment.fileName && (
                <Text style={[styles.fileName, { color: themeColors.text }]} numberOfLines={1}>
                  {attachment.fileName}
                </Text>
              )}
            </TouchableOpacity>
          );
        })}
      </View>

      {/* Fullscreen Image Modal */}
      <Modal
        visible={!!selectedImage}
        transparent
        animationType="fade"
        onRequestClose={() => setSelectedImage(null)}
      >
        <TouchableOpacity
          style={styles.modalContainer}
          activeOpacity={1}
          onPress={() => setSelectedImage(null)}
        >
          {selectedImage && (
            <Image
              source={{ uri: selectedImage }}
              style={styles.fullscreenImage}
              resizeMode="contain"
            />
          )}
        </TouchableOpacity>
      </Modal>
    </>
  );
};

const styles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    gap: 8,
    marginTop: 8,
  },
  imageContainer: {
    width: 150,
    height: 150,
    borderRadius: 8,
    overflow: 'hidden',
  },
  image: {
    width: '100%',
    height: '100%',
  },
  documentContainer: {
    width: 150,
    height: 150,
    borderRadius: 8,
    alignItems: 'center',
    justifyContent: 'center',
    padding: 12,
  },
  fileName: {
    marginTop: 8,
    fontSize: 12,
  },
  modalContainer: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.9)',
    justifyContent: 'center',
    alignItems: 'center',
  },
  fullscreenImage: {
    width: '100%',
    height: '100%',
  },
});
```

---

### Subtask 2.6 & 2.7: Update MessageBubble and ChatDetailScreen

```typescript
// MessageBubble.tsx - Add attachments prop
interface MessageBubbleProps {
  // ... existing props
  attachments?: MessageAttachment[];
}

export const MessageBubble: React.FC<MessageBubbleProps> = ({
  // ... existing props
  attachments,
}) => {
  return (
    <View style={styles.bubble}>
      {/* ... existing message content */}
      
      {attachments && attachments.length > 0 && (
        <AttachmentDisplay attachments={attachments} />
      )}
      
      {/* ... existing footer */}
    </View>
  );
};

// ChatDetailScreen.tsx - Update message handling
const handleSendMessage = async (messageText: string, attachments?: PickedFile[]) => {
  // ... existing code
  
  const response = await chatService.sendMessage(
    conversation.productId,
    messageText,
    attachments,
    conversation.id
  );
  
  // ... rest of the code
};

// Update message mapping to include attachments
const mappedMessages = messages.map((msg: any) => ({
  ...msg,
  timestamp: typeof msg.timestamp === 'string' ? new Date(msg.timestamp) : msg.timestamp,
  attachments: msg.attachments || [], // Include attachments
})).reverse();

// Pass attachments to MessageBubble
<MessageBubble
  // ... existing props
  attachments={msg.attachments}
/>
```

---

## 3. Socket.IO Integration

### Why This Approach?
- **Singleton Pattern**: Single socket instance across app
- **Context API**: Easy access from any component
- **Auto-reconnect**: Handles network issues
- **Room-based**: User-specific rooms for targeted messages

### Subtask 3.1: Install Socket.IO Client

```bash
cd homexa-mobile
npm install socket.io-client
```

---

### Subtask 3.2: Create Socket Service

```typescript
// homexa-mobile/src/services/socket.service.ts (NEW FILE)
import { io, Socket } from 'socket.io-client';
import { API_BASE_URL } from '@/src/config/env';

class SocketService {
  private socket: Socket | null = null;
  private userId: string | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;

  connect(userId: string, token: string) {
    if (this.socket?.connected && this.userId === userId) {
      console.log('Socket already connected');
      return;
    }

    // Disconnect existing connection if different user
    if (this.socket) {
      this.disconnect();
    }

    this.userId = userId;

    console.log('🔌 Connecting to socket server...');

    // Socket.IO connects to base URL (same server as API, but no /api path)
    // If API_BASE_URL is "https://example.com/api", socket URL should be "https://example.com"
    const socketUrl = API_BASE_URL.replace(/\/api\/?$/, '') || API_BASE_URL;

    this.socket = io(socketUrl, {
      auth: {
        userId: userId,
        token: token, // Optional: if backend validates token
      },
      transports: ['websocket', 'polling'], // Fallback to polling if websocket fails
      reconnection: true,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 5000,
      reconnectionAttempts: this.maxReconnectAttempts,
    });

    this.setupEventHandlers();
  }

  private setupEventHandlers() {
    if (!this.socket) return;

    this.socket.on('connect', () => {
      console.log('✅ Socket connected:', this.socket?.id);
      this.reconnectAttempts = 0;
    });

    this.socket.on('disconnect', (reason) => {
      console.log('❌ Socket disconnected:', reason);
    });

    this.socket.on('connect_error', (error) => {
      console.error('🔴 Socket connection error:', error);
      this.reconnectAttempts++;
      
      if (this.reconnectAttempts >= this.maxReconnectAttempts) {
        console.error('Max reconnection attempts reached');
      }
    });

    this.socket.on('reconnect', (attemptNumber) => {
      console.log('🔄 Socket reconnected after', attemptNumber, 'attempts');
      this.reconnectAttempts = 0;
    });

    this.socket.on('reconnect_attempt', (attemptNumber) => {
      console.log('🔄 Reconnection attempt:', attemptNumber);
    });
  }

  disconnect() {
    if (this.socket) {
      console.log('🔌 Disconnecting socket...');
      this.socket.disconnect();
      this.socket = null;
      this.userId = null;
      this.reconnectAttempts = 0;
    }
  }

  // Event Listeners
  on(event: string, callback: (...args: any[]) => void) {
    if (this.socket) {
      this.socket.on(event, callback);
    }
  }

  off(event: string, callback?: (...args: any[]) => void) {
    if (this.socket) {
      this.socket.off(event, callback);
    }
  }

  emit(event: string, data: any) {
    if (this.socket?.connected) {
      this.socket.emit(event, data);
    } else {
      console.warn('Socket not connected, cannot emit:', event);
    }
  }

  isConnected(): boolean {
    return this.socket?.connected || false;
  }

  getSocket(): Socket | null {
    return this.socket;
  }
}

export const socketService = new SocketService();
```

**Explanation**:
- **Singleton**: One instance for entire app
- **Authentication**: Pass userId in handshake.auth (backend expects this)
- **Auto-reconnect**: Built-in reconnection with limits
- **Event Management**: Easy on/off for event listeners

---

### Subtask 3.3: Create Socket Context

```typescript
// homexa-mobile/src/contexts/SocketContext.tsx (NEW FILE)
import React, { createContext, useContext, useEffect, useState, ReactNode } from 'react';
import { socketService } from '@/src/services/socket.service';
import { useAuthStore } from '@/src/stores/authStore';

interface SocketContextType {
  isConnected: boolean;
  socket: any; // Socket instance if needed
}

const SocketContext = createContext<SocketContextType | undefined>(undefined);

export const SocketProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
  const [isConnected, setIsConnected] = useState(false);
  const { user, token } = useAuthStore();

  useEffect(() => {
    if (user?.id && token) {
      console.log('🔌 Initializing socket for user:', user.id);
      
      // Connect socket
      socketService.connect(user.id, token);

      // Listen for connection status
      const updateConnectionStatus = () => {
        setIsConnected(socketService.isConnected());
      };

      socketService.on('connect', updateConnectionStatus);
      socketService.on('disconnect', updateConnectionStatus);

      // Initial status check
      updateConnectionStatus();

      // Cleanup on unmount
      return () => {
        console.log('🧹 Cleaning up socket connection');
        socketService.off('connect', updateConnectionStatus);
        socketService.off('disconnect', updateConnectionStatus);
        socketService.disconnect();
      };
    } else {
      // Disconnect if user logs out
      socketService.disconnect();
      setIsConnected(false);
    }
  }, [user?.id, token]);

  return (
    <SocketContext.Provider
      value={{
        isConnected,
        socket: socketService.getSocket(),
      }}
    >
      {children}
    </SocketContext.Provider>
  );
};

export const useSocket = () => {
  const context = useContext(SocketContext);
  if (!context) {
    throw new Error('useSocket must be used within SocketProvider');
  }
  return context;
};
```

---

### Subtask 3.4: Initialize Socket in App Root

```typescript
// homexa-mobile/app/_layout.tsx
import { SocketProvider } from '@/src/contexts/SocketContext';

export default function RootLayout() {
  return (
    <ThemeProvider>
      <ToastProvider>
        <SocketProvider> {/* NEW: Wrap app with SocketProvider */}
          {/* ... rest of app */}
        </SocketProvider>
      </ToastProvider>
    </ThemeProvider>
  );
}
```

---

### Subtask 3.5: Add Socket Listeners in ChatDetailScreen

```typescript
// homexa-mobile/src/features/chat/screens/ChatDetailScreen.tsx
import { socketService } from '@/src/services/socket.service';
import { useSocket } from '@/src/contexts/SocketContext';

export default function ChatDetailScreen() {
  const { isConnected } = useSocket();
  // ... existing code

  useEffect(() => {
    if (!conversationId || !conversation?.id) return;

    // Socket event listeners
    const handleNewMessage = (data: { chatId: string; message: any }) => {
      if (data.chatId !== conversation.id) return; // Only for current chat

      // Avoid duplicates (check if message already exists)
      setMessages(prevMessages => {
        const exists = prevMessages.some(msg => msg.id === data.message.id);
        if (exists) return prevMessages;

        // Add new message
        const newMessage: ChatMessage = {
          id: data.message.id,
          senderId: data.message.senderId,
          senderName: data.message.senderName || 'Unknown',
          message: data.message.content || '',
          timestamp: new Date(data.message.createdAt || Date.now()),
          isRead: false,
          attachments: data.message.attachments || [],
        };

        return [...prevMessages, newMessage];
      });

      // Scroll to bottom
      setTimeout(() => scrollViewRef.current?.scrollToEnd({ animated: true }), 100);
    };

    const handleMessageRead = (data: { chatId: string; readerId: string }) => {
      if (data.chatId !== conversation.id) return;
      
      // Update read status for messages sent by current user
      setMessages(prevMessages =>
        prevMessages.map(msg =>
          msg.senderId === user?.id ? { ...msg, isRead: true } : msg
        )
      );
    };

    const handleMessageDeletedForAll = (data: { messageId: string; chatId: string }) => {
      if (data.chatId !== conversation.id) return;
      
      // Remove message completely
      setMessages(prevMessages =>
        prevMessages.filter(msg => msg.id !== data.messageId)
      );
    };

    const handleMessageDeletedForMe = (data: { messageId: string; chatId: string }) => {
      if (data.chatId !== conversation.id) return;
      
      // Mark as deleted
      setMessages(prevMessages =>
        prevMessages.map(msg =>
          msg.id === data.messageId ? { ...msg, isDeleted: true } : msg
        )
      );
    };

    // Register listeners
    socketService.on('new_message', handleNewMessage);
    socketService.on('message_read', handleMessageRead);
    socketService.on('message_deleted_for_all', handleMessageDeletedForAll);
    socketService.on('message_deleted_for_me', handleMessageDeletedForMe);

    // Cleanup listeners on unmount
    return () => {
      socketService.off('new_message', handleNewMessage);
      socketService.off('message_read', handleMessageRead);
      socketService.off('message_deleted_for_all', handleMessageDeletedForAll);
      socketService.off('message_deleted_for_me', handleMessageDeletedForMe);
    };
  }, [conversationId, conversation?.id, user?.id]);

  // ... rest of component
}
```

**Key Points**:
- Check `chatId` to only handle events for current chat
- Prevent duplicate messages (check if message ID exists)
- Clean up listeners to prevent memory leaks
- Update UI optimistically while handling socket events

---

### Subtask 3.6: Update ChatListScreen (Similar Pattern)

```typescript
// homexa-mobile/src/features/chat/screens/ChatListScreen.tsx
import { socketService } from '@/src/services/socket.service';

export default function ChatListScreen() {
  // ... existing code

  useEffect(() => {
    const handleChatUpdated = (data: { chatId: string }) => {
      // Refresh chat list
      fetchChats();
    };

    const handleNewChat = (data: { chatId: string; productId: string }) => {
      // Refresh chat list to show new chat
      fetchChats();
    };

    socketService.on('chat_updated', handleChatUpdated);
    socketService.on('new_chat', handleNewChat);

    return () => {
      socketService.off('chat_updated', handleChatUpdated);
      socketService.off('new_chat', handleNewChat);
    };
  }, []);

  // ... rest of component
}
```

---

## 🎯 Testing Checklist

### Delete Message
- [ ] Long press on sent message shows delete menu
- [ ] "Delete for Me" marks message as deleted
- [ ] "Delete for Everyone" removes message completely
- [ ] Error handling works (rollback on failure)
- [ ] Cannot delete received messages

### File Attachments
- [ ] Image picker opens and allows selection
- [ ] Selected images show preview
- [ ] Can remove images before sending
- [ ] Files upload successfully
- [ ] Images display in message bubbles
- [ ] Can view fullscreen image
- [ ] File size validation works

### Socket.IO
- [ ] Socket connects on app start (if logged in)
- [ ] New messages appear in real-time
- [ ] Read receipts update in real-time
- [ ] Deleted messages update in real-time
- [ ] Chat list updates on new chats
- [ ] Reconnection works after network loss
- [ ] No duplicate messages on reconnect

---

## 🚨 Common Issues & Solutions

### Issue: Socket not connecting
**Solution**: 
- Check API_BASE_URL (should be base URL, not /api endpoint)
- Verify userId is passed in handshake.auth
- Check backend socket initialization
- Test with multiple clients

### Issue: Files not uploading
**Solution**:
- Verify FormData format (React Native specific)
- Check file size limits
- Ensure Content-Type header is set
- Test with different file types

### Issue: Duplicate messages
**Solution**:
- Check message IDs before adding
- Remove optimistic UI if socket receives message
- Use message ID as key in map()

---

## 📚 Additional Resources

- [Socket.IO Client Docs](https://socket.io/docs/v4/client-api/)
- [Expo Image Picker](https://docs.expo.dev/versions/latest/sdk/image-picker/)
- [React Native FormData](https://reactnative.dev/docs/network#using-fetch)

