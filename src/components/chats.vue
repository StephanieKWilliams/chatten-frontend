<template>
    <div class="container">
      <div class="row">
        <div class="col-sm-6 offset-3">
  
          <div v-if="sessionStarted" id="chat-container" class="card">
            <div class="card-header text-white text-center font-weight-bold subtle-blue-gradient">
              Share the page URL to invite new friends
            </div>
  
            <div class="card-body">
              <div class="container chat-body">
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
              <form>
                <div class="row">
                  <div class="col-sm-10">
                    <input type="text" placeholder="Type a message" />
                  </div>
                  <div class="col-sm-2">
                    <button class="btn btn-primary">Send</button>
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
  
  const $ = window.jQuery
  
  export default {
    data () {
      return {
        sessionStarted: false,
        messages: [
          {"status":"SUCCESS","uri":"040213b14a02451","message":"Hello!","user":{"id":1,"username":"danidee","email":"osaetindaniel@gmail.com","first_name":"","last_name":""}},
          {"status":"SUCCESS","uri":"040213b14a02451","message":"Hey whatsup! i dey","user":{"id":2,"username":"daniel","email":"","first_name":"","last_name":""}}
        ]
      }
    },
  
    created () {
      this.username = sessionStorage.getItem('username')
  
     
    },
  
    methods: {
      async startChatSession() {
  try {
    const token = sessionStorage.getItem('authToken');
    if (!token) {
      alert('Please log in again. Token not found.');
      return;
    }

    console.log("JWT Token: ", token);  // Log the token to debug

    const response = await fetch('http://localhost:8000/api/chats/', {
      method: 'POST',
      headers: {
        'Authorization': `Token ${token}`,  // Correct the Authorization header format
        'Content-Type': 'application/json',
      },
      credentials: 'include',  // Send cookies with the request if needed (e.g., session cookies)
    });

    const data = await response.json();

    if (response.ok) {
      console.log("Chat session created: ", data); // Log the data to debug
      alert("A new session has been created, you'll be redirected automatically.");
      this.sessionStarted = true;
      this.$router.push(`/chats/${data.uri}/`);
    } else {
      console.error("Error starting chat session:", data); // Log the error data
      alert(data.message || 'An error occurred while starting the chat session.');
    }
  } catch (error) {
    console.error('An error occurred:', error);  // Log any errors that occur during the request
    alert('An error occurred: ' + error.message);
  }
}


    }
  }
  </script>