<template>
  <div class="container">
    <h1 class="text-center">Welcome to Chatten!</h1>
    <div id="auth-container" class="row">
      <div class="col-sm-4 offset-sm-4">
        <ul class="nav nav-tabs nav-justified" id="myTab" role="tablist">
          <li class="nav-item">
            <a class="nav-link" 
               :class="{ active: isSignUp }" 
               id="signup-tab" 
               data-toggle="tab" 
               href="#signup" 
               role="tab" 
               aria-controls="signup" 
               aria-selected="true" 
               @click="isSignUp = true">Sign Up</a>
          </li>
          <li class="nav-item">
            <a class="nav-link" 
               :class="{ active: !isSignUp }" 
               id="signin-tab" 
               data-toggle="tab" 
               href="#signin" 
               role="tab" 
               aria-controls="signin" 
               aria-selected="false" 
               @click="isSignUp = false">Sign In</a>
          </li>
        </ul>

        <div class="tab-content" id="myTabContent">
          <!-- Sign Up Tab -->
          <div class="tab-pane fade" :class="{ 'show active': isSignUp }" id="signup" role="tabpanel" aria-labelledby="signup-tab">
            <form @submit.prevent="signUp">
              <div class="form-group">
                <input v-model="email" type="email" class="form-control" id="email" placeholder="Email Address" required>
              </div>
              <div class="form-row">
                <div class="form-group col-md-6">
                  <input v-model="username" type="text" class="form-control" id="username" placeholder="Username" required>
                </div>
                <div class="form-group col-md-6">
                  <input v-model="password" type="password" class="form-control" id="password" placeholder="Password" required>
                </div>
              </div>
              <div class="form-group">
                <div class="form-check">
                  <input v-model="toc" class="form-check-input" type="checkbox" id="toc" required>
                  <label class="form-check-label" for="toc">
                    Accept terms and Conditions
                  </label>
                </div>
              </div>
              <button type="submit" class="btn btn-block btn-primary" :disabled="loading">
                {{ loading ? 'Loading...' : 'Sign up' }}
              </button>
            </form>
          </div>

          <!-- Sign In Tab -->
          <div class="tab-pane fade" :class="{ 'show active': !isSignUp }" id="signin" role="tabpanel" aria-labelledby="signin-tab">
            <form @submit.prevent="signIn">
              <div class="form-group">
                <input v-model="username" type="text" class="form-control" id="username" placeholder="Username" required>
              </div>
              <div class="form-group">
                <input v-model="password" type="password" class="form-control" id="password" placeholder="Password" required>
              </div>
              <button type="submit" class="btn btn-block btn-primary" :disabled="loading">
                {{ loading ? 'Loading...' : 'Sign in' }}
              </button>
            </form>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
export default {
  data() {
    return {
      email: '',
      username: '',
      password: '',
      toc: false,
      loading: false,
      isSignUp: true  // Controls the active tab
    };
  },
  
  methods: {
    // Sign Up method
    signUp() {
      const userData = {
        email: this.email,
        username: this.username,
        password: this.password,
      };

      this.loading = true;
      console.log("Sending sign up request...");

      fetch('http://localhost:8000/auth/users/', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        credentials: 'include',
        body: JSON.stringify(userData),
      })
        .then(response => {
          if (!response.ok) {
            throw new Error('Sign Up failed: ' + response.statusText);
          }
          return response.json();
        })
        .then(data => {
          alert("Your account has been created.");
          this.isSignUp = false; // Switch to the sign-in tab after successful sign-up
        })
        .catch(error => {
          console.error(error);
          alert(error.message);
        })
        .finally(() => {
          this.loading = false;
          console.log("Sign up request completed.");
        });
    },

    // Sign In method
    signIn() {
      const credentials = {
        username: this.username,
        password: this.password,
      };

      this.loading = true;
      console.log("Sending sign in request...");

      fetch('http://localhost:8000/auth/token/login/', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        credentials: 'include',
        body: JSON.stringify(credentials),
      })
        .then(response => {
          if (!response.ok) {
            throw new Error('Login failed: ' + response.statusText);
          }
          return response.json();
        })
        .then(data => {
          sessionStorage.setItem('authToken', data.auth_token);
          sessionStorage.setItem('username', this.username);
          alert("Logged in successfully!");
          this.$router.push('/chats');  // Redirect to chats
        })
        .catch(error => {
          console.error(error);
          alert(error.message);
        })
        .finally(() => {
          this.loading = false;
          console.log("Sign in request completed.");
        });
    },
  }
};
</script>
