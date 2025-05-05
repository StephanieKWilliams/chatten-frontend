<template>
  <div class="chat-application">
    <!-- Welcome screen -->
    <div v-if="!sessionStarted" class="welcome-screen">
      <div class="welcome-container">
        <img src="/api/placeholder/80/80" alt="Chat logo" class="welcome-logo" />
        <h2 class="welcome-title">Welcome to ChatApp</h2>
        <p class="welcome-text">
          Connect with friends in real-time! Start a new chat session and share the link to invite others.
        </p>
        <button @click="startChatSession" class="btn-start">
          <span class="btn-icon">💬</span>
          Start Chatting
        </button>
      </div>
    </div>

    <!-- Chat interface -->
    <div v-else class="chat-interface">
      <!-- Header with share options -->
      <header class="chat-header">
        <div class="header-content">
          <h3 class="room-title">Chat Room</h3>
          <div class="active-users">
            <div class="user-avatars">
              <div v-for="(participant, index) in uniqueParticipants" :key="participant"
                   class="avatar" :style="{ zIndex: uniqueParticipants.length - index }">
                {{ participant[0].toUpperCase() }}
              </div>
            </div>
            <span>{{ uniqueParticipants.length }} online</span>
          </div>
        </div>
        <div class="share-section">
          <input
            ref="shareInput"
            type="text"
            :value="currentUrl"
            class="share-input"
            readonly
          />
          <button @click="copyShareLink" class="btn-share">
            {{ linkCopied ? 'Copied!' : 'Copy Link' }}
          </button>
        </div>
      </header>

      <!-- Messages area -->
      <div class="messages-container" ref="chatBody">
        <div class="date-divider" v-if="messages.length > 0">
          {{ formatDate(messages[0].created_at || new Date()) }}
        </div>

        <transition-group name="message-transition">
          <div
            v-for="(message, index) in messages"
            :key="message.id || `msg-${index}`"
            class="message-item"
            :class="{'user-message': username === message.user.username, 'peer-message': username !== message.user.username}"
          >
            <!-- Show date divider when date changes -->
            <div class="date-divider" v-if="index > 0 && showDateDivider(messages[index-1], message)">
              {{ formatDate(message.created_at || new Date()) }}
            </div>

            <!-- Message with avatar -->
            <div class="message-content">
              <div
                class="avatar"
                :class="{'user-avatar': username === message.user.username, 'peer-avatar': username !== message.user.username}"
              >
                {{ message.user.username[0].toUpperCase() }}
              </div>
              <div class="message-bubble-container">
                <span v-if="username !== message.user.username" class="sender-name">{{ message.user.username }}</span>
                <div class="message-bubble">
                  <p>{{ message.message }}</p>
                  <span class="message-time">{{ formatTime(message.created_at || new Date()) }}</span>
                </div>
              </div>
            </div>
          </div>
        </transition-group>

        <div v-if="isTyping" class="typing-indicator">
          <div class="typing-avatar">
            <span>{{ typingUser[0].toUpperCase() }}</span>
          </div>
          <div class="typing-bubble">
            <span class="dot"></span>
            <span class="dot"></span>
            <span class="dot"></span>
          </div>
        </div>

        <div v-if="messages.length === 0" class="empty-chat">
          <img src="/api/placeholder/120/120" alt="Empty chat" class="empty-chat-image" />
          <p>No messages yet. Be the first to say hello!</p>
        </div>
      </div>

      <!-- Message input area -->
      <div class="message-input-container">
        <div class="input-wrapper">
          <textarea
            v-model="message"
            @keydown.enter.prevent="postMessage"
            @input="handleInput"
            ref="messageInput"
            placeholder="Type your message..."
            class="message-input"
          ></textarea>
          <div class="input-actions">
            <button class="emoji-btn" @click="showEmojiPicker = !showEmojiPicker">
              😊
            </button>
            <div v-if="showEmojiPicker" class="emoji-picker">
              <div class="emoji-list">
                <span v-for="emoji in commonEmojis" :key="emoji" @click="addEmoji(emoji)" class="emoji">{{ emoji }}</span>
              </div>
            </div>
          </div>
        </div>
        <button
          class="send-button"
          :class="{ 'send-active': message.trim().length > 0 }"
          @click="postMessage"
          :disabled="!message.trim()"
        >
          <span class="send-icon">➤</span>
        </button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    // Generate a default username if none exists
    const storedUsername = sessionStorage.getItem('username');
    const defaultUsername = storedUsername || 'user_' + Math.floor(Math.random() * 10000);

    // If no username was stored, set the generated one
    if (!storedUsername) {
      sessionStorage.setItem('username', defaultUsername);
    }

    return {
      sessionStarted: false,
      messages: [],
      message: '',
      username: defaultUsername,
      socket: null,
      socketConnected: false,
      showEmojiPicker: false,
      linkCopied: false,
      isTyping: false,
      typingUser: '',
      typingTimeout: null,
      historyPollingInterval: null,
      socketReconnectAttempts: 0,
      lastMessageId: null,
      commonEmojis: ['😊', '👍', '❤️', '😂', '🎉', '🙏', '👏', '🔥', '😍', '🤔'],
      currentUrl: window.location.href,
      debug: true, // Set to true to show debug messages in the console
      processedMessageIds: new Set(), // Track processed message IDs to prevent duplicates
    };
  },

  computed: {
    uniqueParticipants() {
      // Get unique usernames from messages
      const participants = new Set();
      this.messages.forEach(msg => {
        if (msg.user && msg.user.username) {
          participants.add(msg.user.username);
        }
      });
      return Array.from(participants);
    }
  },

  created() {
    // Check if the chat session URI exists in the route params and join the session
    if (this.$route.params.uri) {
      this.joinChatSession();
      this.currentUrl = window.location.href;
    }

    // Initialize WebSocket connection
    this.initializeWebSocket();
  },

  mounted() {
    // Add event listener for clicks outside emoji picker
    document.addEventListener('click', this.handleClickOutside);

    // Set up polling for chat history
    this.startPollingHistory();

    // Log initial state
    console.log('Component mounted, sessionStarted:', this.sessionStarted);
    console.log('Current route URI:', this.$route.params.uri);
  },

  beforeDestroy() {
    // Clean up event listeners and intervals
    document.removeEventListener('click', this.handleClickOutside);
    clearInterval(this.historyPollingInterval);

    // Close WebSocket connection
    if (this.socket && this.socket.readyState === WebSocket.OPEN) {
      this.socket.close();
    }
  },

  methods: {
    // Format date for dividers
    formatDate(dateString) {
      const date = new Date(dateString);
      const today = new Date();
      const yesterday = new Date(today);
      yesterday.setDate(yesterday.getDate() - 1);

      if (date.toDateString() === today.toDateString()) {
        return 'Today';
      } else if (date.toDateString() === yesterday.toDateString()) {
        return 'Yesterday';
      } else {
        return date.toLocaleDateString('en-US', {
          weekday: 'long',
          year: 'numeric',
          month: 'long',
          day: 'numeric'
        });
      }
    },

    // Format time for messages
    formatTime(dateString) {
      const date = new Date(dateString);
      return date.toLocaleTimeString('en-US', {
        hour: '2-digit',
        minute: '2-digit'
      });
    },

    // Check if we need to show a date divider between messages
    showDateDivider(prevMessage, currentMessage) {
      const prevDate = new Date(prevMessage.created_at || new Date());
      const currentDate = new Date(currentMessage.created_at || new Date());

      return prevDate.toDateString() !== currentDate.toDateString();
    },

    // Handle input in the message textarea
    handleInput() {
      // Auto-resize the textarea
      this.$nextTick(() => {
        const textarea = this.$refs.messageInput;
        textarea.style.height = 'auto';
        textarea.style.height = `${Math.min(textarea.scrollHeight, 150)}px`;
      });

      // Send typing indicator through WebSocket
      if (this.socket && this.socket.readyState === WebSocket.OPEN) {
        this.socket.send(JSON.stringify({
          type: 'typing',
          username: this.username
        }));
      }
    },

    // Add emoji to message
    addEmoji(emoji) {
      this.message += emoji;
      this.showEmojiPicker = false;
      this.$refs.messageInput.focus();
    },

    // Handle clicks outside the emoji picker
    handleClickOutside(event) {
      if (this.$el.querySelector('.emoji-picker') &&
          !this.$el.querySelector('.emoji-picker').contains(event.target) &&
          !this.$el.querySelector('.emoji-btn').contains(event.target)) {
        this.showEmojiPicker = false;
      }
    },

    // Copy share link to clipboard
    copyShareLink() {
      const shareInput = this.$refs.shareInput;
      shareInput.select();
      document.execCommand('copy');

      this.linkCopied = true;
      setTimeout(() => {
        this.linkCopied = false;
      }, 2000);
    },

    // Start polling for chat history
    startPollingHistory() {
      // Fetch immediately and then set up interval
      this.fetchChatSessionHistory();

      // Set up polling interval - fetch every 30 seconds
      this.historyPollingInterval = setInterval(() => {
        console.log('Polling for new messages...');
        this.fetchChatSessionHistory();
      }, 30000);

      console.log('Message polling started');
    },

    // Scroll to the bottom of the chat container
    scrollToBottom() {
      this.$nextTick(() => {
        const chatBody = this.$refs.chatBody;
        if (chatBody) {
          chatBody.scrollTop = chatBody.scrollHeight;
        }
      });
    },

    // Start a new chat session
    async startChatSession() {
      try {
        const token = sessionStorage.getItem('authToken');
        if (!token) {
          alert('Please log in again. Token not found.');
          return;
        }

        const response = await fetch('http://localhost:8000/api/chats/', {
          method: 'POST',
          headers: {
            'Authorization': `Token ${token}`,
            'Content-Type': 'application/json',
          },
          credentials: 'include',
        });

        const data = await response.json();

        if (response.ok) {
          this.sessionStarted = true;
          this.$router.push(`/chats/${data.uri}/`);
          this.currentUrl = window.location.href;
        } else {
          alert(data.message || 'An error occurred while starting the chat session.');
        }
      } catch (error) {
        alert('An error occurred: ' + error.message);
      }
    },

    // Join an existing chat session
    async joinChatSession() {
      const uri = this.$route.params.uri;
      const token = sessionStorage.getItem('authToken');

      console.log('Joining chat session with URI:', uri);

      try {
        const response = await fetch(`http://localhost:8000/api/chats/${uri}/`, {
          method: 'PATCH',
          headers: {
            'Authorization': `Token ${token}`,
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({ username: this.username }),
        });

        const data = await response.json();
        console.log('Join session response:', data);

        if (response.ok) {
          this.sessionStarted = true;
          console.log('Successfully joined session, sessionStarted set to true');
          await this.fetchChatSessionHistory();
        } else {
          console.error('Failed to join chat session:', data);
          alert('Failed to join the chat session: ' + (data.message || 'Unknown error'));
        }
      } catch (error) {
        console.error('Exception while joining session:', error);
        alert('An error occurred while joining the session: ' + error.message);
      }
    },

    // Fetch the chat session history - FIXED VERSION
    async fetchChatSessionHistory() {
      if (!this.$route.params.uri) {
        console.log('No URI found, cannot fetch chat history');
        return;
      }

      const uri = this.$route.params.uri;
      const token = sessionStorage.getItem('authToken');

      if (!token) {
        console.error('No authentication token available');
        return;
      }

      console.log('Fetching chat history...');

      try {
        const response = await fetch(`http://localhost:8000/api/chats/${uri}/messages/`, {
          method: 'GET',
          headers: {
            'Authorization': `Token ${token}`,
            'Content-Type': 'application/json',
          },
          credentials: 'include',
        });

        const responseText = await response.text();

        let data;
        try {
          data = JSON.parse(responseText);
        } catch (e) {
          console.error('Failed to parse history response as JSON:', e);
          return;
        }

        if (response.ok) {
          if (!data.messages || !Array.isArray(data.messages)) {
            console.error('Invalid messages format in history response');
            return;
          }

          // Create a map of existing messages by ID for quick lookup
          const existingMessagesMap = {};
          this.messages.forEach(msg => {
            if (msg.id && !msg.id.toString().startsWith('temp_')) {
              existingMessagesMap[msg.id] = msg;
            }
          });

          // Find temporary messages that need to be preserved
          const tempMessages = this.messages.filter(msg =>
            msg.id && msg.id.toString().startsWith('temp_')
          );

          // Create a new messages array with server messages
          const newMessages = [];

          // Add all server messages that aren't already in our list
          data.messages.forEach(serverMsg => {
            // Add to processed IDs to prevent duplicates
            this.processedMessageIds.add(serverMsg.id);
            newMessages.push(serverMsg);
          });

          // Add any temporary messages that aren't in the server response
          tempMessages.forEach(tempMsg => {
            // Check if this temp message has a corresponding server message
            const hasServerVersion = data.messages.some(serverMsg =>
              serverMsg.message === tempMsg.message &&
              serverMsg.user.username === tempMsg.user.username
            );

            if (!hasServerVersion) {
              newMessages.push(tempMsg);
            }
          });

          // Sort messages by timestamp
          newMessages.sort((a, b) => {
            const dateA = new Date(a.created_at || Date.now());
            const dateB = new Date(b.created_at || Date.now());
            return dateA - dateB;
          });

          // Update messages array
          this.messages = newMessages;
          this.scrollToBottom();
        } else {
          console.error('Failed to fetch messages:', data);
        }
      } catch (error) {
        console.error('An error occurred while fetching messages:', error);
      }
    },

    // Post a new message to the chat - FIXED VERSION
    async postMessage() {
      if (!this.message.trim()) {
        return; // Avoid sending empty messages
      }

      const messageContent = this.message.trim();
      const uri = this.$route.params.uri;
      const token = sessionStorage.getItem('authToken');

      // Clear the input immediately for better UX
      const messageToSend = messageContent;
      this.message = '';

      // Reset textarea height
      if (this.$refs.messageInput) {
        this.$refs.messageInput.style.height = 'auto';
      }

      // Create temporary message with unique ID
      const tempId = 'temp_' + Date.now();
      const tempMessage = {
        id: tempId,
        message: messageToSend,
        user: {
          username: this.username
        },
        created_at: new Date().toISOString()
      };

      // Add temporary message to UI
      this.messages = [...this.messages, tempMessage];
      this.scrollToBottom();

      try {
        const response = await fetch(`http://localhost:8000/api/chats/${uri}/messages/`, {
          method: 'POST',
          headers: {
            'Authorization': `Token ${token}`,
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({ message: messageToSend }),
          credentials: 'include',
        });

        const responseText = await response.text();

        let messageData;
        try {
          messageData = JSON.parse(responseText);
        } catch (e) {
          console.error('Failed to parse response as JSON:', e);
          throw new Error('Invalid JSON response from server');
        }

        if (response.ok) {
          // Add the server message ID to our processed list
          if (messageData.id) {
            this.processedMessageIds.add(messageData.id);
          }

          // Replace temp message with the real one from the server
          const updatedMessages = this.messages.map(msg =>
            msg.id === tempId ? messageData : msg
          );

          this.messages = updatedMessages;
        } else {
          // Remove the temp message on error
          this.messages = this.messages.filter(msg => msg.id !== tempId);
          alert(messageData.message || 'An error occurred while sending the message.');
          this.message = messageToSend; // Restore the message to input
        }
      } catch (error) {
        // Remove the temp message on error
        this.messages = this.messages.filter(msg => msg.id !== tempId);
        alert('An error occurred: ' + error.message);
        this.message = messageToSend; // Restore the message to input
      }
    },

    // Initialize the WebSocket connection
    async initializeWebSocket() {
      if (!this.$route.params.uri) {
        console.log('No URI found, cannot initialize WebSocket');
        return;
      }

      const chatSessionId = this.$route.params.uri;
      const token = sessionStorage.getItem('authToken');
      const username = this.username;

      if (!token) {
        console.error('No authentication token available for WebSocket');
        return;
      }

      try {
        // Close existing connection if any
        if (this.socket) {
          console.log('Closing existing WebSocket connection');
          this.socket.close();
        }

        console.log(`Connecting to WebSocket: ws://localhost:8001/ws/chats/${chatSessionId}/`);
        const socket = new WebSocket(`ws://localhost:8001/ws/chats/${chatSessionId}/`);

        // Set WebSocket timeout - the connection should establish within 5 seconds
        const connectionTimeout = setTimeout(() => {
          if (socket.readyState !== WebSocket.OPEN) {
            console.error('WebSocket connection timeout');
            socket.close();
          }
        }, 5000);

        socket.onopen = () => {
          clearTimeout(connectionTimeout);
          console.log("WebSocket connection established successfully");
          this.socketConnected = true;

          // Send authentication message immediately after connection
          if (token) {
            console.log('Sending authentication message with token and username');
            socket.send(JSON.stringify({
              type: 'authenticate',
              token: token,
              username: username
            }));
          }
        };

        socket.onmessage = (event) => {
          try {
            const data = JSON.parse(event.data);
            console.log('WebSocket message received:', data);

            if (data.type === 'typing') {
              // Handle typing indicator
              this.isTyping = true;
              this.typingUser = data.username;

              // Clear previous timeout if exists
              if (this.typingTimeout) {
                clearTimeout(this.typingTimeout);
              }

              // Hide typing indicator after 3 seconds
              this.typingTimeout = setTimeout(() => {
                this.isTyping = false;
              }, 3000);
            }
            else if (data.type === 'message' && data.message) {
              // Only add the message if we haven't processed it before
              if (!this.processedMessageIds.has(data.message.id)) {
                this.processedMessageIds.add(data.message.id);

                // Check if this is a replacement for a temp message
                const tempIndex = this.messages.findIndex(msg =>
                  msg.id.toString().startsWith('temp_') &&
                  msg.message === data.message.message &&
                  msg.user.username === data.message.user.username
                );

                if (tempIndex !== -1) {
                  // Replace the temp message
                  const updatedMessages = [...this.messages];
                  updatedMessages[tempIndex] = data.message;
                  this.messages = updatedMessages;
                } else {
                  // Add as a new message
                  this.messages = [...this.messages, data.message];
                }

                this.scrollToBottom();
              }
            }
          } catch (error) {
            console.error('Error processing WebSocket message:', error);
          }
        };

        socket.onerror = (error) => {
          console.error('WebSocket Error:', error);
          this.socketConnected = false;
        };

        socket.onclose = (event) => {
          console.log("WebSocket connection closed:", event.code, event.reason);
          this.socketConnected = false;

          // Try to reconnect after a delay, unless we're intentionally closing
          if (event.code !== 1000) {
            setTimeout(() => {
              if (this.sessionStarted) {
                console.log("Attempting to reconnect WebSocket...");
                this.initializeWebSocket();
              }
            }, 5000);
          }
        };

        this.socket = socket;
      } catch (error) {
        console.error("Failed to initialize WebSocket:", error);
      }
    }
  }
};
</script>

<style scoped>
/* Base styles */
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

.chat-application {
  height: 100vh;
  width: 100%;
  background-color: #f5f7fb;
  display: flex;
  flex-direction: column;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
}

/* Welcome screen */
.welcome-screen {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  background: linear-gradient(135deg, #6e8efb, #a777e3);
  color: white;
}

.welcome-container {
  text-align: center;
  max-width: 500px;
  padding: 2rem;
  background-color: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(10px);
  border-radius: 1rem;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
}

.welcome-logo {
  width: 80px;
  height: 80px;
  border-radius: 50%;
  margin-bottom: 1.5rem;
  background-color: white;
  padding: 1rem;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.welcome-title {
  font-size: 2rem;
  margin-bottom: 1rem;
  font-weight: 700;
}

.welcome-text {
  margin-bottom: 2rem;
  line-height: 1.6;
  font-size: 1.1rem;
  opacity: 0.9;
}

.btn-start {
  background-color: white;
  color: #6e8efb;
  border: none;
  padding: 0.8rem 2rem;
  font-size: 1.1rem;
  font-weight: 600;
  border-radius: 2rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.btn-start:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.15);
}

.btn-icon {
  margin-right: 0.5rem;
  font-size: 1.2rem;
}

/* Chat interface */
.chat-interface {
  display: flex;
  flex-direction: column;
  height: 100%;
  background-color: #f5f7fb;
}

/* Header */
.chat-header {
  background: linear-gradient(135deg, #6e8efb, #a777e3);
  color: white;
  padding: 1rem;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  z-index: 100;
}

.header-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
}

.room-title {
  font-weight: 600;
  font-size: 1.25rem;
}

.active-users {
  display: flex;
  align-items: center;
  font-size: 0.85rem;
}

.user-avatars {
  display: flex;
  margin-right: 0.5rem;
}

.user-avatars .avatar {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  background-color: #4b6cb7;
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  font-size: 0.8rem;
  margin-left: -10px;
  border: 2px solid white;
}

.user-avatars .avatar:first-child {
  margin-left: 0;
}

.share-section {
  display: flex;
  align-items: center;
}

.share-input {
  flex: 1;
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 1.5rem 0 0 1.5rem;
  font-size: 0.9rem;
  outline: none;
  background-color: rgba(255, 255, 255, 0.2);
  color: white;
}

.share-input::placeholder {
  color: rgba(255, 255, 255, 0.7);
}

.btn-share {
  background-color: rgba(255, 255, 255, 0.25);
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 0 1.5rem 1.5rem 0;
  font-size: 0.9rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
}

.btn-share:hover {
  background-color: rgba(255, 255, 255, 0.35);
}

/* Messages container */
.messages-container {
  flex: 1;
  overflow-y: auto;
  padding: 1rem;
  display: flex;
  flex-direction: column;
}

.date-divider {
  text-align: center;
  padding: 0.5rem 0;
  margin: 0.75rem 0;
  font-size: 0.85rem;
  color: #8795a1;
  position: relative;
}

.date-divider::before,
.date-divider::after {
  content: "";
  position: absolute;
  top: 50%;
  width: 40%;
  height: 1px;
  background-color: #e2e8f0;
}

.date-divider::before {
  left: 0;
}

.date-divider::after {
  right: 0;
}

.message-item {
  margin-bottom: 1rem;
  animation: fadeIn 0.3s ease-out;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

.message-transition-enter-active,
.message-transition-leave-active {
  transition: all 0.3s;
}

.message-transition-enter-from,
.message-transition-leave-to {
  opacity: 0;
  transform: translateY(30px);
}

.message-content {
  display: flex;
  align-items: flex-start;
  max-width: 85%;
}

.user-message .message-content {
  flex-direction: row-reverse;
  margin-left: auto;
}

.avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background-color: #4b6cb7;
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  flex-shrink: 0;
}

.user-avatar {
  background: linear-gradient(135deg, #6e8efb, #a777e3);
  margin-left: 0.75rem;
}

.peer-avatar {
  background-color: #64748b;
  margin-right: 0.75rem;
}

.message-bubble-container {
  display: flex;
  flex-direction: column;
}

.sender-name {
  font-size: 0.8rem;
  font-weight: 600;
  margin-bottom: 0.2rem;
  color: #64748b;
}

.message-bubble {
  padding: 0.75rem 1rem;
  border-radius: 1.25rem;
  position: relative;
  max-width: 100%;
  word-wrap: break-word;
}

.user-message .message-bubble {
  background: linear-gradient(135deg, #6e8efb, #a777e3);
  color: white;
  border-bottom-right-radius: 0.25rem;
}

.peer-message .message-bubble {
  background-color: white;
  color: #334155;
  border-bottom-left-radius: 0.25rem;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
}

.message-bubble p {
  margin: 0;
  line-height: 1.5;
}

.message-time {
  font-size: 0.7rem;
  margin-top: 0.3rem;
  display: block;
  text-align: right;
  opacity: 0.7;
}

.typing-indicator {
  display: flex;
  align-items: flex-start;
  margin-top: 0.5rem;
  margin-bottom: 0.5rem;
}

.typing-avatar {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background-color: #64748b;
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  font-size: 0.8rem;
  margin-right: 0.5rem;
}

.typing-bubble {
  background-color: white;
  padding: 0.5rem 1rem;
  border-radius: 1.25rem;
  border-bottom-left-radius: 0.25rem;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
  display: flex;
  align-items: center;
}

.dot {
  background-color: #64748b;
  width: 6px;
  height: 6px;
  border-radius: 50%;
  margin: 0 3px;
  animation: bounce 1.5s infinite ease-in-out;
}

.dot:nth-child(2) {
  animation-delay: 0.2s;
}

.dot:nth-child(3) {
  animation-delay: 0.4s;
}

@keyframes bounce {
  0%, 80%, 100% { transform: translateY(0); }
  40% { transform: translateY(-8px); }
}

.empty-chat {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: #94a3b8;
  text-align: center;
}

.empty-chat-image {
  opacity: 0.7;
  margin-bottom: 1rem;
}

/* Input area */
.message-input-container {
  padding: 1rem;
  background-color: white;
  border-top: 1px solid #e2e8f0;
  display: flex;
  align-items: flex-end;
  box-shadow: 0 -2px 10px rgba(0, 0, 0, 0.05);
}

.input-wrapper {
  position: relative;
  flex: 1;
  margin-right: 0.75rem;
}

.message-input {
  width: 100%;
  padding: 0.75rem 3rem 0.75rem 1rem;
  border: 1px solid #e2e8f0;
  border-radius: 1.5rem;
  font-family: inherit;
  font-size: 1rem;
  resize: none;
  overflow-y: auto;
  max-height: 150px;
  outline: none;
  transition: all 0.2s ease;
}

.message-input:focus {
  border-color: #a777e3;
  box-shadow: 0 0 0 2px rgba(167, 119, 227, 0.2);
}

.input-actions {
  position: absolute;
  right: 1rem;
  bottom: 0.75rem;
  display: flex;
  align-items: center;
}

.emoji-btn {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 1.2rem;
  padding: 0;
  opacity: 0.7;
  transition: all 0.2s ease;
}

.emoji-btn:hover {
  opacity: 1;
  transform: scale(1.1);
}

.emoji-picker {
  position: absolute;
  bottom: calc(100% + 10px);
  right: -10px;
  background-color: white;
  border-radius: 0.75rem;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15);
  padding: 0.75rem;
  z-index: 10;
  width: 250px;
}

.emoji-list {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 0.5rem;
}

.emoji {
  font-size: 1.5rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0.25rem;
  border-radius: 0.25rem;
  transition: all 0.2s ease;
}

.emoji:hover {
  background-color: #f1f5f9;
  transform: scale(1.1);
}

.send-button {
  width: 50px;
  height: 50px;
  border-radius: 50%;
  background-color: #e2e8f0;
  border: none;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s ease;
}

.send-active {
  background: linear-gradient(135deg, #6e8efb, #a777e3);
  color: white;
  box-shadow: 0 4px 12px rgba(110, 142, 251, 0.4);
}

.send-button:hover.send-active {
  transform: scale(1.05);
}

.send-icon {
  font-size: 1.2rem;
}

/* Responsive design */
@media (max-width: 768px) {
  .chat-header {
    padding: 0.75rem;
  }

  .header-content {
    flex-direction: column;
    align-items: flex-start;
  }

  .active-users {
    margin-top: 0.5rem;
  }

  .share-section {
    margin-top: 0.5rem;
    width: 100%;
  }

  .share-input {
    font-size: 0.8rem;
    padding: 0.4rem 0.75rem;
  }

  .btn-share {
    font-size: 0.8rem;
    padding: 0.4rem 0.75rem;
  }

  .message-content {
    max-width: 95%;
  }

  .avatar {
    width: 34px;
    height: 34px;
    font-size: 0.8rem;
  }

  .message-bubble {
    padding: 0.6rem 0.8rem;
    font-size: 0.9rem;
  }

  .message-input {
    padding: 0.6rem 2.5rem 0.6rem 0.8rem;
    font-size: 0.9rem;
  }

  .input-actions {
    right: 0.75rem;
    bottom: 0.6rem;
  }

  .emoji-btn {
    font-size: 1rem;
  }

  .emoji-picker {
    width: 220px;
  }

  .emoji-list {
    grid-template-columns: repeat(4, 1fr);
  }

  .emoji {
    font-size: 1.2rem;
  }

  .send-button {
    width: 40px;
    height: 40px;
  }
}

/* Dark mode support */
@media (prefers-color-scheme: dark) {
  .chat-application {
    background-color: #1a202c;
    color: #e2e8f0;
  }

  .chat-interface {
    background-color: #1a202c;
  }

  .peer-message .message-bubble {
    background-color: #2d3748;
    color: #e2e8f0;
  }

  .sender-name {
    color: #a0aec0;
  }

  .date-divider {
    color: #a0aec0;
  }

  .date-divider::before,
  .date-divider::after {
    background-color: #4a5568;
  }

  .typing-bubble {
    background-color: #2d3748;
  }

  .dot {
    background-color: #a0aec0;
  }

  .empty-chat {
    color: #a0aec0;
  }

  .message-input-container {
    background-color: #2d3748;
    border-top: 1px solid #4a5568;
  }

  .message-input {
    background-color: #1a202c;
    border-color: #4a5568;
    color: #e2e8f0;
  }

  .message-input::placeholder {
    color: #a0aec0;
  }

  .emoji-picker {
    background-color: #2d3748;
  }

  .emoji:hover {
    background-color: #4a5568;
  }

  .send-button {
    background-color: #4a5568;
    color: #e2e8f0;
  }
}
</style>

