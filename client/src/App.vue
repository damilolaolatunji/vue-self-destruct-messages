<template>
  <div class="App">
    <aside class="sidebar left-sidebar">
      <section v-if="!currentUser" class="join-chat">
        <h3>Join Chat</h3>
        <form @submit.prevent="addUser">
          <input
            placeholder="Enter your username"
            type="text"
            name="userId"
            v-model="userId"
            />
        </form>
      </section>

      <section v-if="currentUser" class="room-users">
        <h3>Room Users</h3>
        <ul>
          <li v-for="user in roomUsers" :key="user.id" class="user">
            <span class="presence" :class="user.presence.state" />
            <span>{{ user.name }}</span>
          </li>
        </ul>
      </section>
    </aside>

    <section class="chat-window">
      <header class="room-name">
        <h3 v-if="currentRoom">{{ currentRoom.name }}</h3>
        <h3 v-else>Chat</h3>
      </header>
      <section class="chat-session">
        <div v-for="message in messages" :key="message.id">
          <div v-if="message.type" class="status-message">
            <span>{{ message.text }}</span>
          </div>
          <div v-else class="message">
            <span class="user-id">{{ message.senderId }}</span>
            <span>{{ message.text }}</span>
          </div>
        </div>
      </section>
      <form @submit.prevent="sendMessage" class="message-form">
        <input
          class="message-input"
          autofocus
          placeholder="Compose your message and hit ENTER to send"
          v-model="newMessage"
          :disabled="!currentUser"
          name="newMessage"
          />
      </form>
    </section>

    <aside class="sidebar right-sidebar">
      <section v-if="currentUser" class="preferences">
        <h3>Self-destruct Timeout</h3>
        <select
          id="timeout"
          name="timeout"
          v-model="messageTimer"
          @change="updateMessageTimer"
          >
          <option value="0">Off</option>
          <option value="10000">10 seconds</option>
          <option value="20000">20 seconds</option>
          <option value="30000">30 seconds</option>
          <option value="60000">1 minute</option>
        </select>
      </section>
    </aside>
  </div>
</template>

<script>
import Chatkit from '@pusher/chatkit-client';
import axios from 'axios';

export default {
  name: 'app',
  data () {
    return {
      title: 'Chatkit',
      userId: '',
      currentUser: null,
      currentRoom: null,
      newMessage: '',
      messages: [],
      roomUsers: [],
      messageTimer: '0',
    }
  },
  methods: {
    addUser() {
      const { userId } = this;
      axios
        .post('http://localhost:5200/users', { userId })
        .then(() => {
          const tokenProvider = new Chatkit.TokenProvider({
            url: 'http://localhost:5200/authenticate',
          });

          const chatManager = new Chatkit.ChatManager({
            instanceLocator: '<your chatkit instance locator>',
            userId,
            tokenProvider,
          });

          return chatManager
            .connect({
              onRoomUpdated: room => {
                const { messageTimer } = room.customData;
                this.messageTimer = messageTimer;
                this.showStatusMessage();
              },
            })
            .then(currentUser => {
              this.currentUser = currentUser;
              this.connectToRoom()
            });
        })
        .catch(console.error);
    },

    sendMessage() {
      const { newMessage, currentUser, currentRoom } = this;

      if (newMessage.trim() === '') return;

      currentUser.sendMessage({
        text: newMessage,
        roomId: `${currentRoom.id}`,
      });

      this.newMessage = '';
    },

    connectToRoom() {
      const { currentUser } = this;

      return currentUser
        .subscribeToRoom({
          roomId: '<your room id>',
          messageLimit: 100,
          hooks: {
            onMessage: message => {
              const messages = [...this.messages];
              const index = messages.findIndex(item => item.id === message.id);
              if (index !== -1) {
                messages.splice(index, 1, message);
              } else {
                messages.push(message);
              }

              const { messageTimer } = this;
              if (message.text !== 'DELETED' && messageTimer !== '0') {
                this.deleteMessage(message.id);
              }

              this.messages = messages;
            },
            onPresenceChanged: () => {
              const { currentRoom } = this;
              this.roomUsers = currentRoom.users.sort(a => {
                if (a.presence.state === 'online') return -1;

                return 1;
              });
            },
          },
        })
        .then(currentRoom => {
          this.currentRoom = currentRoom;
          this.roomUsers = currentRoom.users;
        });
    },

    updateMessageTimer(event) {
      const { value } = event.target;
      const { currentRoom, currentUser } = this;

      currentUser.updateRoom({
        roomId: currentRoom.id,
        customData: { messageTimer: value },
      });
    },

    showStatusMessage() {
      const { messageTimer, messages } = this;
      const text = `The disappearing message timeout has been set to ${messageTimer /
          1000} seconds`;

      const statusMessage = {
        id: `${Date.now() + Math.random()}`,
        text,
        type: 'status',
      };
      messages.push(statusMessage);

      this.messages = messages;
    },

    deleteMessage(id) {
      const { messageTimer } = this;
      axios
        .post('http://localhost:5200/delete-message', {
          messageId: id,
          timer: Number(messageTimer),
        })
        .catch(console.error);
    },
  }
}
</script>
