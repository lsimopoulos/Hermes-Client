<template>
  <div class="chat-window-container" v-if="contactName">
    <div class="chat-header">
      <div class="contact-info">
        <span class="contact-name">{{ contactName }}</span>
      </div>
      <div class="header-icons">
        <button class="history-button" @click="fetchOldMessages" :disabled="fetchedAllHistoryMessages">
          <font-awesome-icon icon="history" />
        </button>
      </div>
    </div>
    <div class="chatcontainer" @scroll="onScroll">

      <div v-for="(msg, index) in currentMessages" :key="index" ref="chat_container"
        :class="msg.isSelf ? 'my-message' : 'other-message'">
        <ChatMessage :from="msg.name" :time="msg.time" :isSelf="msg.isSelf" :message="msg.message" />

      </div>
      <div class="typing-indicator" v-if="otherTyping">
        <span class="dot"></span>
        <span class="dot"></span>
        <span class="dot"></span>
      </div>
      <button v-if="!isBottom" type="submit" class="btn btn-primary float" @click="scrollToBottom">
        <font-awesome-icon icon="angle-double-down" />
      </button>
    </div>

    <div class="chat-input-area">
      <input type="text" class="form-control text-input" v-model="chatMsg" v-on:keyup.enter="sendMessage"
        @keypress='handleKeyPressed' />
      <button type="submit"
        class="inline-block align-middle text-center select-none border font-normal whitespace-no-wrap rounded py-1 px-3 leading-normal no-underline bg-blue-600 text-white hover:bg-blue-600 sendmessage-btn"
        @click="sendMessage">
        <font-awesome-icon icon="paper-plane" />
      </button>
    </div>
  </div>
</template>
<script>
import ChatMessage from "./ChatMessage.vue";
import { chatService } from "../services/chat.service";

export default {
  name: "ChatWindowComponent",
  props: {
    messages: Array,
    unreadMessages: Number,
    contactName: String
  },
  created() {
    document.addEventListener('isTypingEvent', this.isTypingEvent);
    document.addEventListener('messagedAddedEvent', this.messageAddedEvent);
    document.addEventListener('changeChatContentEvent', this.handleChangeOfChatContentEvent);

  },
  emits: ["update-unread"],
  computed: {
    currentMessages: {
      get() {
        return this.messages;
      },
    }
  },
  components: {
    // eslint-disable-next-line
    ChatMessage,
  },
  methods: {
    fetchOldMessages() {
      let id = this.$store.getters['auth/user_id'];
      let self = this;
      if (this.messages.length > 0) {
        let argId;
        if (this.messages[0].id !== this.lastMessageId) {
          argId = this.messages[0].id;
        } else {
          argId = this.lastMessageId;
        }

        chatService.fetchOldMessages(argId).then((response) => {
          self.lastMessageId = response.lastid;
          if (self.lastMessageId === "") {
            self.fetchedAllHistoryMessages = true;
          }
          response.messages.forEach((c) => {
            self.messages.splice(0, 0, this.updateMessageIfSelf(c, id));
          });
        });
      }
      else {
        this.lastMessageId = "";
        chatService.fetchOldMessages().then((response) => {
          this.lastMessageId = response.lastid;
          console.log(this.lastMessageId);
          response.messages.forEach((c) => {
            self.messages.splice(0, 0, this.updateMessageIfSelf(c, id));
          });
        });
      }
    },
    updateMessageIfSelf(message, id) {
      if (message.from === id) {
        message.isSelf = true;
      }
      return message;
    },
    sendMessage() {
      if (this.chatMsg) {
        this.stopMeTyping();
        chatService.sendMessage(this.chatMsg);
        this.chatMsg = "";

        this.$nextTick(() => {
          this.scrollToBottom();
        });
      }
    },
    handleChangeOfChatContentEvent(event) {
   
      if (event.detail.sourceChanged || (this.isBottom && event.detail.newMessage)) {
        this.$nextTick(() => {
          this.scrollToBottom();
        });
      }
    },
    handleKeyPressed() {
      if (!this.meTyping) {
        this.meTyping = true;
        const customEvent = new CustomEvent('sendIsTypingEvent', {});
        document.dispatchEvent(customEvent);
        this.meTypingTimer = setTimeout(() => this.stopMeTyping(), 5000);
      }
    },
    onScroll(e) {
      if (!this.isScrollingFromCode) {
        const { scrollTop, offsetHeight, scrollHeight } = e.target;
        this.isBottom = scrollTop + offsetHeight + 50 > scrollHeight;
        if (this.isBottom && this.unreadMessages > 0) {
          this.$emit('update-unread', 0);
        }
      }
    },
    scrollToBottom() {
      this.isScrollingFromCode = true;
      const chatcontainer = this.$refs.chat_container;
      if (chatcontainer && chatcontainer.length > 0) {
        let divNumber =
          this.unreadMessages > 0
            ? chatcontainer.length - this.unreadMessages - 1
            : chatcontainer.length - 1;
        if (divNumber < 0) divNumber = 0;
        chatcontainer[divNumber].scrollIntoView({ behavior: "smooth" });
      }
      setTimeout(
        function () {
          this.isScrollingFromCode = false;
          if (this.isBottom && this.unreadMessages > 0) {
            this.$emit('update-unread', 0);
          }
        }.bind(this),
        500
      );
    },
    messageAddedEvent(event) {
      const contactId = event.detail.id;
      const selected_contact = this.$store.getters["user/selected_contact"];
      if (selected_contact.id === contactId) {
        this.otherTyping = false;
        clearTimeout(this.otherTypingTimer);
      }
    },
    isTypingEvent(event) {
      const status = event.detail;
      const selected_contact = this.$store.getters["user/selected_contact"];
      if (selected_contact.id === status.from) {
        if (this.otherTyping) {
          clearTimeout(this.otherTypingTimer);
        }
        this.otherTyping = true;
        this.otherTypingTimer = setTimeout(() => this.stopOtherTyping(), 5000);
      }
    },
    stopMeTyping() {
      this.meTyping = false;

      clearTimeout(this.meTypingTimer);
    },
    stopOtherTyping() {
      this.otherTyping = false;
      clearTimeout(this.otherTypingTimer);
    }
  },
  beforeUnmount() {
    document.removeEventListener('isTypingEvent', this.isTypingEvent);
    document.removeEventListener('messagedAddedEvent', this.messageAddedEvent);
    document.removeEventListener('changeChatContentEvent', this.handleChangeOfChatContentEvent);
  },
  data() {
    return {
      chatMsg: "",
      isBottom: true,
      isScrollingFromCode: false,
      meTypingTimer: "",
      metyping: false,
      otherTypingTimer: "",
      otherTyping: false,
      lastMessageId: null,
      fetchedAllHistoryMessages: false
    };
  },
};
</script>
<style scoped>
.typing-indicator {
  display: flex;
  justify-content: flex-start;
  align-items: center;
  margin-top: 10px;
  height: 20px;
}

.dot {
  width: 8px;
  height: 8px;
  background-color: #ccc;
  border-radius: 50%;
  margin-right: 4px;
  animation: dot-pulse 1.5s infinite ease-in-out;
}

@keyframes dot-pulse {

  0%,
  40%,
  100% {
    opacity: 0.3;
  }

  20% {
    opacity: 1;
  }
}

.chat-window-container {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  width: 100%;
  height: 90vh;
}

.chatcontainer {
  padding: 0 20px;
  margin: 0 auto;
  min-height: 92%;
  width: 100%;
  border: 3px solid #f1f1f1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
}

.chat-input-area {
  width: 100%;
  margin-bottom: 10px;
  display: flex;
  gap: 10px;
}

.sendmessage-btn {
  border-radius: 50%;
}

.text-input {
  width: 90%;
}

.float {
  position: fixed;
  width: 40px;
  height: 35px;
  bottom: 80px;
  right: 30px;
  border-radius: 50px;
  text-align: center;
  box-shadow: 2px 2px 3px #999;
}

.my-message {
  display: flex;
  justify-content: flex-end;
}

.other-message {
  display: flex;
  justify-content: flex-start;
}

.chat-header {
  background-color: #d1d7db;
  color: white;
  padding: 10px 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.header-icons {
  display: flex;
  gap: 10px;
  margin-left: auto;
}
</style>