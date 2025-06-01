<template>
  <div class="care-chat-app">
    <img class="logo" src="/logo.png" alt="">
    <div class="chat-container">
      <div class="messages-area" ref="messagesAreaRef">
        <div v-for="(msg, index) in chatMessages" :key="index" :class="['message-bubble', msg.sender]">
          <div class="sender-name">{{ msg.sender === 'user' ? 'You' : msg.sender === 'ai' ? 'Care AI' : 'System' }}</div>
          <div class="message-text">{{ msg.text }}</div>
          <div class="message-meta" v-if="msg.usage">
            Tokens: {{ msg.usage.total_tokens }}
          </div>
          <div class="message-meta" v-if="msg.timestamp">
            {{ new Date(msg.timestamp).toLocaleTimeString() }}
          </div>
        </div>
      </div>
      <div class="input-area">
        <input
          type="text"
          v-model="userInput"
          @keyup.enter="sendMessage"
          placeholder="Type your message to Care AI..."
          :disabled="!isConnected"
        />
        <button @click="sendMessage" :disabled="!isConnected || userInput.trim() === ''">
          Send
        </button>
      </div>
      <div class="status-area">
        <p>Status: <span :class="statusClass">{{ connectionStatus }}</span></p>
        <p v-if="userId">User ID: {{ userId }}</p>
      </div>
    </div>

    <div class="doctor-summary-section">
      <button @click="fetchDoctorSummary" :disabled="!userId">Fetch Doctor Summary</button>
      <div v-if="doctorSummaryLoading" class="loading">Loading summary...</div>
      <div v-if="doctorSummary && !doctorSummaryLoading" class="summary-display">
        <h3>Doctor Summary:</h3>
        <pre>{{ JSON.stringify(doctorSummary, null, 2) }}</pre>
      </div>
      <div v-if="doctorSummaryError" class="error-message">
        Error fetching summary: {{ doctorSummaryError }}
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick, computed } from 'vue';

const userInput = ref('');
const chatMessages = ref([]);
const doctorSummary = ref(null);
const doctorSummaryLoading = ref(false);
const doctorSummaryError = ref('');

let socket = null;
// Generate a somewhat unique userId for the session for demo purposes
const userId = ref('vue_user_' + Math.random().toString(16).substring(2, 10));
const connectionStatus = ref('Disconnected');
const isConnected = ref(false);
const messagesAreaRef = ref(null);


const statusClass = computed(() => {
  if (isConnected.value) return 'connected';
  if (connectionStatus.value === 'Connecting...') return 'connecting';
  return 'disconnected';
});

const scrollToBottom = () => {
  nextTick(() => {
    if (messagesAreaRef.value) {
      messagesAreaRef.value.scrollTop = messagesAreaRef.value.scrollHeight;
    }
  });
};

onMounted(() => {
  connectWebSocket();
});

onUnmounted(() => {
  if (socket) {
    socket.onopen = null;
    socket.onmessage = null;
    socket.onerror = null;
    socket.onclose = null;
    socket.close();
  }
});

const connectWebSocket = () => {
  connectionStatus.value = 'Connecting...';
  // Ensure your backend is running on localhost:3000 or update the URL
  socket = new WebSocket('ws://localhost:3000');

  socket.onopen = () => {
    isConnected.value = true;
    connectionStatus.value = 'Connected';
    console.log('WebSocket connection established.');
    addSystemMessage('Connection established with Care AI.');
    // Send an initial message to associate userId with this WebSocket connection
    socket.send(JSON.stringify({ type: 'init_connection', userId: userId.value }));
  };

  socket.onmessage = (event) => {
    try {
      const messageData = JSON.parse(event.data);
      console.log('Received from server:', messageData);

      if (messageData.type === 'ai_response' && messageData.payload) {
        chatMessages.value.push({
          sender: 'ai',
          text: messageData.payload.text || "AI didn't send text.",
          usage: messageData.payload.usage,
          timestamp: Date.now()
        });
      } else if (messageData.type === 'system_message') {
        addSystemMessage(messageData.text);
      } else if (messageData.type === 'error') {
        addSystemMessage(`Server Error: ${messageData.message}`, 'error');
      }
      scrollToBottom();
    } catch (error) {
      console.error('Error parsing message from server:', error, event.data);
      addSystemMessage('Received an unparseable message from the server.', 'error');
    }
  };

  socket.onerror = (error) => {
    isConnected.value = false;
    connectionStatus.value = 'Error';
    console.error('WebSocket error:', error);
    addSystemMessage('WebSocket connection error. Please try refreshing.', 'error');
  };

  socket.onclose = (event) => {
    isConnected.value = false;
    connectionStatus.value = `Disconnected (Code: ${event.code})`;
    console.log('WebSocket connection closed:', event.reason, event.code);
    addSystemMessage(`Connection closed. ${event.reason || ''}`, 'error');
    // Optional: attempt to reconnect
    // setTimeout(connectWebSocket, 5000); // Reconnect after 5 seconds
  };
};

const addSystemMessage = (text, type = 'info') => {
  chatMessages.value.push({
    sender: 'system',
    text: text,
    type: type, // 'info' or 'error'
    timestamp: Date.now()
  });
  scrollToBottom();
};

const sendMessage = () => {
  if (userInput.value.trim() === '' || !socket || !isConnected.value) {
    addSystemMessage('Cannot send message. Not connected or input is empty.', 'error');
    return;
  }
  const messagePayload = {
    type: 'chat_message',
    userId: userId.value,
    message: userInput.value,
  };
  socket.send(JSON.stringify(messagePayload));
  chatMessages.value.push({ sender: 'user', text: userInput.value, timestamp: Date.now() });
  userInput.value = '';
  scrollToBottom();
};

const fetchDoctorSummary = async () => {
  if (!userId.value) {
    doctorSummaryError.value = "User ID is not set.";
    return;
  }
  doctorSummaryLoading.value = true;
  doctorSummary.value = null;
  doctorSummaryError.value = '';
  try {
    // Ensure backend is running on localhost:3000 or update the URL
    const response = await fetch(`http://localhost:3000/chat/${userId.value}/doctorsummary`);
    if (!response.ok) {
      const errorData = await response.json().catch(() => ({ message: `HTTP error! Status: ${response.status}` }));
      throw new Error(errorData.message || `Failed to fetch summary with status: ${response.status}`);
    }
    doctorSummary.value = await response.json();
  } catch (error) {
    console.error('Failed to fetch doctor summary:', error);
    doctorSummaryError.value = error.message;
  } finally {
    doctorSummaryLoading.value = false;
  }
};

</script>

<style>
.care-chat-app {
  max-width: 800px;
  margin: 20px auto;
  padding: 20px;
  border: 1px solid #ddd;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  background-color: #f9f9f9;
}

h1 {
  text-align: center;
  color: #333;
  margin-bottom: 20px;
}

.chat-container {
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  background-color: #fff;
  padding: 15px;
}

.messages-area {
  height: 450px;
  overflow-y: auto;
  margin-bottom: 15px;
  padding: 10px;
  border: 1px solid #eee;
  background-color: #fdfdfd;
  border-radius: 4px;
}

.message-bubble {
  margin-bottom: 12px;
  padding: 10px 15px;
  border-radius: 18px;
  max-width: 75%;
  word-wrap: break-word;
  clear: both; /* Ensure bubbles don't overlap incorrectly */
}

.sender-name {
  font-weight: bold;
  font-size: 0.85em;
  margin-bottom: 4px;
  color: #555;
}

.message-bubble.user {
  background-color: #07E585;
  color: white;
  margin-left: auto; /* Aligns to the right */
  float: right; /* Fallback for some browsers or if parent isn't flex */
}
.message-bubble.user .sender-name { color: #e0e0e0; }


.message-bubble.ai {
  background-color: #e9ecef;
  color: #333;
  margin-right: auto; /* Aligns to the left */
  float: left; /* Fallback */
}
.message-bubble.ai .sender-name { color: #07E585; }

.message-bubble.system {
  background-color: #fff3cd;
  color: #856404;
  font-style: italic;
  font-size: 0.9em;
  text-align: center;
  max-width: 100%;
  clear: both;
  float: none;
  margin: 10px 0;
}
.message-bubble.system.error {
  background-color: #f8d7da;
  color: #721c24;
}


.message-meta {
  font-size: 0.75em;
  color: #777;
  margin-top: 5px;
  text-align: right;
}
.message-bubble.user .message-meta { color: #d1e7ff; }
.message-bubble.ai .message-meta { color: #6c757d; }


.input-area {
  display: flex;
  margin-top: 10px;
}

.logo {
  margin-bottom: 20px;
}

.input-area input {
  flex-grow: 1;
  padding: 12px;
  border: 1px solid #ccc;
  border-radius: 20px;
  margin-right: 10px;
  font-size: 1em;
}
.input-area input:focus {
  border-color: #07E585;
  box-shadow: 0 0 0 0.2rem rgba(0,123,255,.25);
  outline: none;
}

.input-area button {
  padding: 10px 20px;
  background-color: #07E585;
  color: white;
  border: none;
  border-radius: 20px;
  cursor: pointer;
  font-size: 1em;
  transition: background-color 0.2s;
}

.message-text {
  text-align: left;
}

.input-area button:hover {
  background-color: #07e585;
}
.input-area button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.status-area {
  margin-top: 15px;
  font-size: 0.9em;
  color: #666;
  text-align: center;
}
.status-area .connected { color: green; font-weight: bold; }
.status-area .disconnected { color: red; font-weight: bold; }
.status-area .connecting { color: orange; font-weight: bold; }

.doctor-summary-section {
  margin-top: 30px;
  padding: 15px;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  background-color: #fff;
}
.doctor-summary-section h3 { margin-top: 0;}
.summary-display pre {
  background-color: #f8f9fa;
  padding: 10px;
  border-radius: 4px;
  border: 1px solid #eee;
  max-height: 300px;
  overflow-y: auto;
  white-space: pre-wrap; /* Allows text to wrap */
  word-break: break-all; /* Breaks long words if necessary */
}
.loading, .error-message {
  padding: 10px;
  margin-top: 10px;
  border-radius: 4px;
}
.loading { background-color: #e2e3e5; }
.error-message { background-color: #f8d7da; color: #721c24; }
</style>