<template>
  <div class="care-chat-app">
    <img class="logo" src="/logo.png" alt="Care AI Logo"> <!-- Make sure logo.png is in your public folder -->
    <div class="chat-container">
      <div class="messages-area" ref="messagesAreaRef">
        <div v-for="(msg, index) in chatMessages" :key="index" :class="['message-bubble', msg.sender, msg.type === 'error' ? 'error' : '']">
          <div class="sender-name">{{ msg.sender === 'user' ? 'You' : msg.sender === 'ai' ? 'Care AI' : 'System' }}</div>
          <div class="message-text">{{ msg.text }}</div>
          <div class="message-meta" v-if="msg.usage && msg.sender === 'ai'">
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
      <button @click="fetchDoctorSummary()" :disabled="!userId || doctorSummaryLoading">
        {{ doctorSummaryLoading ? 'Loading Summary...' : 'Fetch Doctor Summary' }}
      </button>
      <div v-if="doctorSummary && !doctorSummaryError && !doctorSummaryLoading" class="summary-display">
        <h3>Doctor Summary:</h3>
        <pre>{{ JSON.stringify(doctorSummary, null, 2) }}</pre>
      </div>
      <div v-if="doctorSummaryError && !doctorSummaryLoading" class="error-message">
        {{ doctorSummaryError }}
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
const userId = ref('vue_user_' + Math.random().toString(16).substring(2, 8)); // Shorter random part
const connectionStatus = ref('Disconnected');
const isConnected = ref(false);
const messagesAreaRef = ref(null);

const MAX_SUMMARY_RETRIES = 3; // Max number of retries for fetching summary
const SUMMARY_RETRY_DELAY = 3000; // Delay in milliseconds (3 seconds)

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
    // Clean up WebSocket event listeners to prevent memory leaks
    socket.onopen = null;
    socket.onmessage = null;
    socket.onerror = null;
    socket.onclose = null;
    if (socket.readyState === WebSocket.OPEN || socket.readyState === WebSocket.CONNECTING) {
      socket.close();
    }
    socket = null; // Ensure socket is nullified
  }
});

const connectWebSocket = () => {
  if (socket && (socket.readyState === WebSocket.OPEN || socket.readyState === WebSocket.CONNECTING)) {
    console.log("WebSocket already open or connecting.");
    return;
  }

  connectionStatus.value = 'Connecting...';
  socket = new WebSocket('ws://localhost:3000'); // Ensure backend is on this address

  socket.onopen = () => {
    isConnected.value = true;
    connectionStatus.value = 'Connected';
    console.log('WebSocket connection established.');
    addSystemMessage('Connection established with Care AI.');
    socket.send(JSON.stringify({ type: 'init_connection', userId: userId.value }));
  };

  socket.onmessage = (event) => {
    try {
      const messageData = JSON.parse(event.data);
      console.log('Received from server:', messageData);

      if (messageData.type === 'ai_response' && messageData.payload) {
        chatMessages.value.push({
          sender: 'ai',
          text: messageData.payload.text || "AI response did not contain text.",
          usage: messageData.payload.usage,
          timestamp: Date.now()
        });
      } else if (messageData.type === 'system_message') {
        addSystemMessage(messageData.text);
      } else if (messageData.type === 'error') {
        addSystemMessage(`Server Error: ${messageData.message}`, 'error');
      } else if (messageData.type === 'summary_ready' && messageData.userId === userId.value) {
        // Optional: If backend implements pushing 'summary_ready'
        addSystemMessage('Doctor summary is now ready to be fetched!', 'info');
        fetchDoctorSummary(); // Automatically fetch if backend signals readiness
      }
      scrollToBottom();
    } catch (error) {
      console.error('Error parsing message from server:', error, event.data);
      addSystemMessage('Received an unparseable message from the server.', 'error');
    }
  };

  socket.onerror = (error) => {
    isConnected.value = false;
    connectionStatus.value = 'Connection Error';
    console.error('WebSocket error:', error);
    addSystemMessage('WebSocket connection error. Please check server and refresh.', 'error');
    // No automatic reconnect here to avoid infinite loops if server is down.
    // User can manually refresh or you can add a reconnect button.
  };

  socket.onclose = (event) => {
    isConnected.value = false;
    connectionStatus.value = `Disconnected (Code: ${event.code})`;
    console.log('WebSocket connection closed:', event.reason, event.code);
    let closeReason = event.reason || 'Connection closed.';
    if (event.code === 1006) { // Abnormal closure
        closeReason = 'Connection lost abnormally. Please check server status.';
    }
    addSystemMessage(closeReason, 'error');
    socket = null; // Ensure socket is nullified
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
  if (userInput.value.trim() === '' || !socket || !isConnected.value || socket.readyState !== WebSocket.OPEN) {
    addSystemMessage('Cannot send message. Not connected or input is empty.', 'error');
    if (!isConnected.value || (socket && socket.readyState !== WebSocket.OPEN)) {
        console.log("Attempting to reconnect before sending...");
        connectWebSocket(); // Try to reconnect if not connected
    }
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

const fetchDoctorSummary = async (retryAttempt = 0) => {
  if (!userId.value) {
    doctorSummaryError.value = "User ID is not set. Cannot fetch summary.";
    addSystemMessage(doctorSummaryError.value, 'error');
    return;
  }

  if (retryAttempt === 0) { // Only set loading true on the initial user-triggered attempt
    doctorSummaryLoading.value = true;
    doctorSummary.value = null;
    doctorSummaryError.value = ''; // Clear previous errors
  }

  try {
    const response = await fetch(`http://localhost:3000/chat/${userId.value}/doctorsummary`);

    if (!response.ok) {
      const errorData = await response.json().catch(() => ({ message: `HTTP error! Status: ${response.status} - ${response.statusText}` }));
      // Check for the specific "not yet available" message for retrying
      if (response.status === 404 && errorData.message === "Doctor summary not yet available." && retryAttempt < MAX_SUMMARY_RETRIES) {
        addSystemMessage(`Summary not ready yet. Retrying in ${SUMMARY_RETRY_DELAY / 1000}s (Attempt ${retryAttempt + 1}/${MAX_SUMMARY_RETRIES})...`, 'info');
        // Keep doctorSummaryLoading true while retrying
        setTimeout(() => fetchDoctorSummary(retryAttempt + 1), SUMMARY_RETRY_DELAY);
        return; // Exit current attempt, loading remains true
      }
      // For other errors or if max retries reached
      throw new Error(errorData.message || `Failed to fetch summary. Status: ${response.status}`);
    }

    const fetchedSummary = await response.json();
    if (fetchedSummary && fetchedSummary.error) { // If the fetched "summary" is actually an error object from the backend
      doctorSummaryError.value = `Server-side summary generation issue: ${fetchedSummary.error}`;
      addSystemMessage(doctorSummaryError.value, 'error');
      doctorSummary.value = null; // Don't display the error object as a valid summary
    } else {
      doctorSummary.value = fetchedSummary;
      addSystemMessage('Doctor summary loaded successfully.', 'info');
      doctorSummaryError.value = ''; // Clear any previous error messages
    }
    doctorSummaryLoading.value = false; // Success or final non-retryable error

  } catch (error) {
    console.error('Failed to fetch doctor summary:', error);
    doctorSummaryError.value = error.message;
    addSystemMessage(`Error fetching summary: ${error.message}`, 'error');
    doctorSummaryLoading.value = false; // Error, stop loading
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
  display: flex;
  flex-direction: column;
}

.logo {
  display: block;
  margin: 0 auto 20px auto; /* Center logo and add bottom margin */
  max-height: 60px; /* Adjust as needed */
}


.chat-container {
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  background-color: #fff;
  padding: 15px;
  margin-bottom: 20px;
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
  border-radius: 18px; /* More rounded */
  max-width: 75%;
  word-wrap: break-word;
  clear: both;
  box-shadow: 0 1px 3px rgba(0,0,0,0.05);
}

.sender-name {
  font-weight: bold;
  font-size: 0.85em;
  margin-bottom: 4px;
}

.message-bubble.user {
  background-color: #07E585; /* Your green */
  color: #fff; /* White text for better contrast */
  margin-left: auto;
  float: right;
}
.message-bubble.user .sender-name { color: #f0f0f0; } /* Lighter name for dark bubble */
.message-bubble.user .message-text { color: #fff; }


.message-bubble.ai {
  background-color: #e9ecef; /* Light grey */
  color: #212529; /* Darker text */
  margin-right: auto;
  float: left;
}
.message-bubble.ai .sender-name { color: #07A05E; } /* Darker green for AI name */

.message-bubble.system {
  background-color: #fff3cd; /* Light yellow for info */
  color: #856404;
  font-style: italic;
  font-size: 0.9em;
  text-align: center;
  max-width: 100%;
  clear: both;
  float: none;
  margin: 10px auto; /* Center system messages */
  padding: 8px 12px;
  border-radius: 8px;
}
.message-bubble.system.error { /* Distinct style for system errors */
  background-color: #f8d7da; /* Light red */
  color: #721c24; /* Dark red text */
  font-style: normal;
  font-weight: bold;
}


.message-meta {
  font-size: 0.75em;
  margin-top: 5px;
  text-align: right;
}
.message-bubble.user .message-meta { color: #e6ffe6; } /* Lighter meta for user */
.message-bubble.ai .message-meta { color: #6c757d; }


.input-area {
  display: flex;
  margin-top: 10px;
}

.input-area input {
  flex-grow: 1;
  padding: 12px 18px; /* More padding */
  border: 1px solid #ccc;
  border-radius: 25px; /* More rounded */
  margin-right: 10px;
  font-size: 1em;
}
.input-area input:focus {
  border-color: #07E585;
  box-shadow: 0 0 0 0.2rem rgba(7, 229, 133, 0.25);
  outline: none;
}

.input-area button {
  padding: 10px 20px;
  background-color: #07E585;
  color: white;
  border: none;
  border-radius: 25px; /* More rounded */
  cursor: pointer;
  font-size: 1em;
  font-weight: bold;
  transition: background-color 0.2s ease-in-out;
}

.message-text {
  text-align: left;
}

.input-area button:hover:not(:disabled) {
  background-color: #06c775; /* Slightly darker green on hover */
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
.status-area .connected { color: #28a745; font-weight: bold; } /* Bootstrap success green */
.status-area .disconnected, .status-area .error { color: #dc3545; font-weight: bold; } /* Bootstrap danger red */
.status-area .connecting { color: #ffc107; font-weight: bold; } /* Bootstrap warning yellow */

.doctor-summary-section {
  margin-top: 30px;
  padding: 20px;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  background-color: #fff;
}
.doctor-summary-section h3 { margin-top: 0; color: #333; }
.doctor-summary-section button {
  background-color: #007bff; /* Bootstrap primary blue for this button */
  color: white;
  padding: 10px 15px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  margin-bottom: 15px;
  transition: background-color 0.2s;
}
.doctor-summary-section button:hover:not(:disabled) {
  background-color: #0056b3;
}
.doctor-summary-section button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.summary-display pre {
  background-color: #f8f9fa;
  padding: 15px;
  border-radius: 4px;
  border: 1px solid #eee;
  max-height: 300px;
  overflow-y: auto;
  white-space: pre-wrap;
  word-break: break-all;
  font-size: 0.9em;
  color: #212529;
}
.loading, .error-message {
  padding: 10px;
  margin-top: 10px;
  border-radius: 4px;
  text-align: center;
}
.loading { background-color: #e2e3e5; color: #333; }
.error-message { background-color: #f8d7da; color: #721c24; font-weight: bold; }
</style>