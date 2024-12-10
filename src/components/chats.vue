<template>
  <div class="container">
    <div class="row">
      <div class="col-sm-6 offset-3">
        <div v-if="sessionStarted" id="chat-container" class="card">
          <div class="card-header text-white text-center font-weight-bold subtle-blue-gradient">
            Share the page URL to invite new friends
          </div>

          <div class="card-body">
            <div class="container chat-body" ref="chatBody">
              <div v-for="message in messages" :key="message.id" class="row chat-section">
                <template v-if="username === message.user.username">
                  <div class="col-sm-7 offset-3">
                    <span class="card-text speech-bubble speech-bubble-user float-right text-white subtle-blue-gradient">
                      {{ message.message }}
                    </span>
                  </div>
                  <div class="col-sm-2">
                    <img class="rounded-circle" :src="`http://placehold.it/40/007bff/fff&text=${message.user.username[0].toUpperCase()}`" />
                  </div>
                </template>
                <template v-else>
                  <div class="col-sm-2">
                    <img class="rounded-circle" :src="`http://placehold.it/40/333333/fff&text=${message.user.username[0].toUpperCase()}`" />
                  </div>
                  <div class="col-sm-7">
                    <span class="card-text speech-bubble speech-bubble-peer">
                      {{ message.message }}
                    </span>
                  </div>
                </template>
              </div>
            </div>
          </div>

          <div class="card-footer text-muted">
            <form @submit.prevent="postMessage">
              <div class="row">
                <div class="col-sm-10">
                  <input type="text" v-model="message" placeholder="Type a message" />
                </div>
                <div class="col-sm-2">
                  <button class="btn btn-primary" type="submit">Send</button>
                </div>
              </div>
            </form>
          </div>
        </div>

        <div v-else>
          <h3 class="text-center">Welcome !</h3>
          <br />
          <p class="text-center">
            To start chatting with friends click on the button below, it'll start a new chat session
            and then you can invite your friends over to chat!
          </p>
          <br />
          <button @click="startChatSession" class="btn btn-primary btn-lg btn-block">Start Chatting</button>
        </div>
      </div>
    </div>
  </div>
</template>


<script>
export default {
  data() {
    return {
      sessionStarted: false,
      messages: [],
      message: '',
      username: sessionStorage.getItem('username'),
      socket: null,
    };
  },

  created() {
    // Check if the chat session URI exists in the route params and join the session
    if (this.$route.params.uri) {
      this.joinChatSession();
    }
    setInterval(this.fetchChatSessionHistory, 3000);  // Fetch history periodically

    // Initialize WebSocket connection
    this.initializeWebSocket();
  },

  methods: {
    // Scroll to the bottom of the chat container
    scrollToBottom() {
      this.$nextTick(() => {
        const chatContainer = this.$refs.chatBody;
        chatContainer.scrollTop = chatContainer.scrollHeight;
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

        if (response.ok) {
          this.sessionStarted = true;
          this.fetchChatSessionHistory();
        } else {
          alert('Failed to join the chat session.');
        }
      } catch (error) {
        alert('An error occurred while joining the session.');
      }
    },

    // Fetch the chat session history
    async fetchChatSessionHistory() {
      const uri = this.$route.params.uri;
      const token = sessionStorage.getItem('authToken');
      try {
        const response = await fetch(`http://localhost:8000/api/chats/${uri}/messages/`, {
          method: 'GET',
          headers: {
            'Authorization': `Token ${token}`,
            'Content-Type': 'application/json',
          },
        });

        const data = await response.json();
        if (response.ok) {
          this.messages = data.messages;
          this.scrollToBottom(); // Scroll to bottom when history is fetched
        } else {
          alert('Failed to fetch messages.');
        }
      } catch (error) {
        alert('An error occurred while fetching messages.');
      }
    },

    // Post a new message to the chat
    async postMessage() {
      if (!this.message.trim()) {
        return; // Avoid sending empty messages
      }

      const data = { message: this.message };
      const uri = this.$route.params.uri;
      const token = sessionStorage.getItem('authToken');
      try {
        const response = await fetch(`http://localhost:8000/api/chats/${uri}/messages/`, {
          method: 'POST',
          headers: {
            'Authorization': `Token ${token}`,
            'Content-Type': 'application/json',
          },
          body: JSON.stringify(data),
        });

        const messageData = await response.json();
        if (response.ok) {
          this.messages.push(messageData);
          this.message = ''; // Clear input after sending
          this.scrollToBottom(); // Scroll to bottom after sending
        } else {
          alert(messageData.message || 'An error occurred while sending the message.');
        }
      } catch (error) {
        alert('An error occurred: ' + error.message);
      }
    },

    // Initialize the WebSocket connection
    async initializeWebSocket() {
      const chatSessionId = this.$route.params.uri;
      const socket = new WebSocket(`ws://localhost:8001/chat/${chatSessionId}/`);

      socket.onopen = () => {
        console.log("WebSocket connection established.");
      };

      socket.onmessage = (event) => {
        const data = JSON.parse(event.data);
        const newMessage = data.message;
        
        // Add the new message to the messages array
        this.messages.push(newMessage);
        this.scrollToBottom(); // Scroll to bottom when a new message arrives
      };

      socket.onerror = (error) => console.log('WebSocket Error: ', error);
      socket.onclose = () => {
        console.log("WebSocket connection closed.");
      };

      this.socket = socket;
    }
  }
};
</script>

  
  <style scoped>
  h1,
  h2 {
    font-weight: normal;
  }
  ul {
    list-style-type: none;
    padding: 0;
  }
  li {
    display: inline-block;
    margin: 0 10px;
  }
  
  .btn {
    border-radius: 0 !important;
  }
  
  .card-footer input[type="text"] {
    background-color: #ffffff;
    color: #444444;
    padding: 7px;
    font-size: 13px;
    border: 2px solid #cccccc;
    width: 100%;
    height: 38px;
  }
  
  .card-header a {
    text-decoration: underline;
  }
  
  .card-body {
    background-color: #ddd;
  }
  
  .chat-body {
    margin-top: -15px;
    margin-bottom: -5px;
    height: 380px;
    overflow-y: auto;
  }
  
  .speech-bubble {
    display: inline-block;
    position: relative;
    border-radius: 0.4em;
    padding: 10px;
    background-color: #fff;
    font-size: 14px;
  }
  
  .subtle-blue-gradient {
    background: linear-gradient(45deg,#004bff, #007bff);
  }
  
  .speech-bubble-user:after {
    content: "";
    position: absolute;
    right: 4px;
    top: 10px;
    width: 0;
    height: 0;
    border: 20px solid transparent;
    border-left-color: #007bff;
    border-right: 0;
    border-top: 0;
    margin-top: -10px;
    margin-right: -20px;
  }
  
  .speech-bubble-peer:after {
    content: "";
    position: absolute;
    left: 3px;
    top: 10px;
    width: 0;
    height: 0;
    border: 20px solid transparent;
    border-right-color: #ffffff;
    border-top: 0;
    border-left: 0;
    margin-top: -10px;
    margin-left: -20px;
  }
  
  .chat-section:first-child {
    margin-top: 10px;
  }
  
  .chat-section {
    margin-top: 15px;
  }
  
  .send-section {
    margin-bottom: -20px;
    padding-bottom: 10px;
  }
  </style>