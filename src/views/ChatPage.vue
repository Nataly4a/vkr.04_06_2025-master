<template>
  <div class="chat-page">
    <AppNavbar />
    <div class="chat-header">
      <h2>{{ chat.name }}</h2>
      <div class="trip-info">
        <span>Поездка: {{ chat.trip }}</span>
        <span>Дата: {{ chat.date }}</span>
      </div>
    </div>
    <div v-if="isLoadingChat" class="loading-state">
      <span class="loading-icon">⏳</span>
      <p>Загрузка чата...</p>
    </div>
    <div v-else-if="errorLoadingChat" class="error-state">
      <span class="error-icon">⚠️</span>
      <p>Не удалось загрузить чат. Попробуйте снова.</p>
      <button class="retry-button" @click="loadChat($route.params.id)" aria-label="Попробовать снова">
        Попробовать снова
      </button>
    </div>
    <div v-else class="chat-messages" ref="messagesContainer">
      <div
        v-for="(message, index) in messages"
        :key="index"
        :class="['message', message.isCurrentUser ? 'message-you' : 'message-other']"
      >
        <div class="message-avatar">
          <img
            :src="message.isCurrentUser ? userAvatar : companionAvatar"
            alt="Avatar"
            :aria-label="message.isCurrentUser ? 'Ваш аватар' : 'Аватар собеседника'"
            @error="handleImageError"
          />
        </div>
        <div class="message-content">
          <span class="message-sender">{{ message.sender_name }} {{ message.sender_surname }}</span>
          <span class="message-text">{{ message.content }}</span>
          <span class="message-time">{{ formatTime(message.sent_at) }}</span>
          <span v-if="message.isSending" class="message-status">Отправка...</span>
        </div>
      </div>
    </div>
    <button
      v-if="showScrollButton"
      class="scroll-bottom-btn"
      @click="scrollToBottom"
      aria-label="Прокрутить вниз"
    >
      <span class="scroll-icon">⬇</span>
    </button>
    <div class="chat-input">
      <input
        v-model="newMessage"
        @keyup.enter="sendMessage"
        placeholder="Введите сообщение..."
        aria-label="Введите сообщение"
        :disabled="isSendingMessage"
      />
      <button
        @click="sendMessage"
        :disabled="isSendingMessage || !newMessage.trim()"
        aria-label="Отправить сообщение"
        class="action-button send"
      >
        <span v-if="isSendingMessage" class="loading-icon">⏳</span>
        <span v-else class="send-text">Отправить</span>
        <span class="send-icon">➤</span>
      </button>
      <button
        class="action-button clear-input"
        @click="clearInput"
        aria-label="Очистить поле ввода"
        :disabled="!newMessage.trim()"
      >
        Очистить
      </button>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import AppNavbar from "@/components/AppNavbar.vue";
import Cookies from 'js-cookie';
import { API_CONFIG } from '@/config/api';
import { debounce } from 'lodash';
import leoProfanity from 'leo-profanity';

// Настройка leo-profanity для клиента
leoProfanity.loadDictionary('ru');
// Добавляем пользовательский список запрещённых слов
const customProfanityList = [
  'жопа', 'Жопа', 'вагина', 'Вагина', 'Пидорасы', 'пидорасы', 'Тупой', 'Тупая', 'тупой',
  'тупая', 'сучка', 'сука', 'Сучка', 'шлюха', 'Шлюха', 'Дура', 'дура', 'Дурак', 'дурак',
];
leoProfanity.add(customProfanityList);

// Функция для поиска конкретного матерного слова
function findProfanityWord(text, profanityList) {
  const lowerText = text.toLowerCase();
  for (const word of profanityList) {
    const lowerWord = word.toLowerCase();
    if (lowerText.includes(lowerWord)) {
      return word;
    }
  }
  return null;
}

export default {
  components: {
    AppNavbar,
  },
  data() {
    return {
      chat: {},
      messages: [],
      newMessage: "",
      socket: null,
      userAvatar: '/pngwing.com (5).png',
      companionAvatar: '/pngwing.com (5).png',
      currentUserId: null,
      companionId: null,
      showScrollButton: false,
      isLoadingChat: false,
      errorLoadingChat: false,
      isSendingMessage: false,
      reconnectAttempts: 0,
      maxReconnectAttempts: 5,
      reconnectDelay: 3000,
    };
  },
  async created() {
    this.currentUserId = parseInt(Cookies.get('userId') || 1);
    const chatId = this.$route.params.id;
    await this.loadChat(chatId);
    await this.loadMessages(chatId);
    this.initWebSocket(chatId);
    this.setupScrollListener();
  },
  beforeUnmount() {
    if (this.socket) {
      this.socket.close(1000, 'Component unmounted');
    }
  },
  methods: {
    async loadChat(chatId) {
      try {
        this.isLoadingChat = true;
        this.errorLoadingChat = false;
        const response = await axios.get(
          `${API_CONFIG.BASE_URL}/chat/${chatId}`,
          {
            headers: {
              'Authorization': `Bearer ${Cookies.get("token")}`
            }
          }
        );
        this.chat = response.data;
        this.companionId = this.chat.passenger_id === this.currentUserId 
          ? this.chat.driver_id 
          : this.chat.passenger_id;
      } catch (error) {
        console.error("Ошибка при загрузке данных о чате:", error);
        this.errorLoadingChat = true;
        this.$toast.error('Не удалось загрузить данные чата.');
      } finally {
        this.isLoadingChat = false;
      }
    },
    async loadMessages(chatId) {
      try {
        const response = await axios.get(
          `${API_CONFIG.BASE_URL}/chat/${chatId}/messages`,
          {
            headers: {
              'Authorization': `Bearer ${Cookies.get("token")}`
            }
          }
        );
        this.messages = response.data.map(msg => ({
          ...msg,
          isCurrentUser: msg.sender_id === this.currentUserId,
          sender_name: msg.sender_name,
          sender_surname: msg.sender_surname
        }));
        this.$nextTick(() => this.scrollToBottom());
      } catch (error) {
        console.error("Ошибка при загрузке сообщений:", error);
        this.$toast.error('Не удалось загрузить сообщения.');
      }
    },
    initWebSocket(chatId) {
      const connect = () => {
        try {
          this.socket = new WebSocket(`${API_CONFIG.WS_URL}/chat/${chatId}`);

          this.socket.onopen = () => {
            console.log('WebSocket соединение установлено');
            this.reconnectAttempts = 0;
            this.socket.send(JSON.stringify({
              type: 'auth',
              token: Cookies.get('token'),
              user_id: this.currentUserId
            }));
            this.$toast.success('Подключение к чату установлено.');
          };

          this.socket.onmessage = (event) => {
            try {
              const message = JSON.parse(event.data);
              if (message.type === 'message') {
                this.handleNewMessage(message);
              } else if (message.type === 'profanity_detected') {
                this.$toast.error(message.error);
              } else if (message.type === 'error') {
                this.$toast.error(`Ошибка: ${message.error}`);
              } else if (message.type === 'auth_success') {
                console.log('Аутентификация успешна');
              }
            } catch (error) {
              console.error('Ошибка обработки WebSocket сообщения:', error);
              this.$toast.error('Ошибка обработки сообщения.');
            }
          };

          this.socket.onclose = (event) => {
            console.log('WebSocket соединение закрыто:', event);
            if (event.code !== 1000 && this.reconnectAttempts < this.maxReconnectAttempts) {
              const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts);
              setTimeout(() => {
                console.log(`Попытка переподключения ${this.reconnectAttempts + 1}...`);
                this.reconnectAttempts++;
                connect();
              }, delay);
            } else if (event.code !== 1000) {
              console.error('Достигнуто максимальное количество попыток переподключения.');
              this.$toast.error('Не удалось подключиться к чату. Используется резервный режим.');
            }
          };

          this.socket.onerror = (error) => {
            console.error('WebSocket ошибка:', error);
          };
        } catch (error) {
          console.error('Ошибка инициализации WebSocket:', error);
          this.$toast.error('Ошибка подключения к чату.');
        }
      };

      connect();
    },
    handleNewMessage(message) {
      this.messages.push({
        ...message,
        isCurrentUser: message.sender_id === this.currentUserId,
        sender_name: message.sender_name,
        sender_surname: message.sender_surname
      });
      this.$nextTick(() => this.scrollToBottom());
    },
    sendMessage: debounce(async function () {
      if (!this.newMessage.trim()) {
        this.$toast.warning('Сообщение не может быть пустым.');
        return;
      }

      const messageData = {
        type: 'message',
        chat_id: parseInt(this.$route.params.id),
        sender_id: this.currentUserId,
        content: this.newMessage,
        sent_at: new Date().toISOString()
      };

      // Оптимистическое добавление сообщения
      const tempMessageIndex = this.messages.push({
        ...messageData,
        isCurrentUser: true,
        sender_name: 'Вы',
        sender_surname: '',
        isSending: true
      }) - 1;
      this.$nextTick(() => this.scrollToBottom());

      // Проверка на мат на клиенте после попытки отправки
      const profanityWord = findProfanityWord(this.newMessage, customProfanityList);
      if (profanityWord) {
        // Удаляем оптимистическое сообщение
        this.messages.splice(tempMessageIndex, 1);
        this.$toast.error(`Сообщение не отправлено: содержит нецензурную лексику "${profanityWord}".`);
        this.newMessage = ""; // Очищаем поле ввода
        return; // Выходим без изменения isSendingMessage
      }

      this.isSendingMessage = true; // Устанавливаем индикатор только после проверки на мат
      try {
        // Попытка отправки через WebSocket
        if (this.socket && this.socket.readyState === WebSocket.OPEN) {
          this.socket.send(JSON.stringify(messageData));
        } else {
          // Фоллбек на HTTP
          await axios.post(
            `${API_CONFIG.BASE_URL}/chat/${this.$route.params.id}/messages`,
            {
              content: messageData.content,
              sender_id: messageData.sender_id
            },
            {
              headers: {
                'Authorization': `Bearer ${Cookies.get("token")}`
              }
            }
          );
        }

        // Обновляем статус сообщения
        this.messages = this.messages.map(msg =>
          msg.sent_at === messageData.sent_at && msg.isSending
            ? { ...msg, isSending: false }
            : msg
        );
        this.$toast.success('Сообщение отправлено.');
      } catch (error) {
        console.error("Ошибка при отправке сообщения:", error);
        // Откат оптимистического обновления при ошибке
        this.messages = this.messages.filter(
          msg => !(msg.sent_at === messageData.sent_at && msg.isSending)
        );
        if (error.response && error.response.status === 400) {
          this.$toast.error(error.response.data.error);
        } else {
          this.$toast.error('Ошибка при отправке сообщения.');
        }
      } finally {
        this.isSendingMessage = false;
        this.newMessage = ""; // Очищаем поле ввода после успешной или неуспешной отправки
      }
    }, 300),
    clearInput() {
      this.newMessage = "";
      this.$toast.info('Поле ввода очищено.');
    },
    scrollToBottom() {
      this.$nextTick(() => {
        const container = this.$refs.messagesContainer;
        if (container) {
          container.scrollTop = container.scrollHeight;
          this.showScrollButton = false;
        }
      });
    },
    setupScrollListener() {
      const container = this.$refs.messagesContainer;
      if (container) {
        container.addEventListener('scroll', () => {
          const isAtBottom = container.scrollHeight - container.scrollTop <= container.clientHeight + 50;
          this.showScrollButton = !isAtBottom;
        });
      }
    },
    formatTime(timestamp) {
      if (!timestamp) return 'Не указано';
      const date = new Date(timestamp);
      return isNaN(date.getTime())
        ? 'Неверное время'
        : date.toLocaleTimeString('ru-RU', { hour: '2-digit', minute: '2-digit' });
    },
    handleImageError(event) {
      event.target.src = 'https://via.placeholder.com/40?text=Avatar';
    }
  }
};
</script>

<style scoped>
/* Базовые стили */
:root {
  --bg-color: #ffffff;
  --text-color: #1a202c;
  --container-bg: #f7f9fc;
  --border-color: #e0e6ed;
  --accent-color: #007bff;
  --accent-hover: #0056b3;
  --secondary-color: #6c757d;
  --message-you-bg: linear-gradient(135deg, #007bff, #0056b3);
  --message-you-text: #ffffff;
  --message-other-bg: #ffffff;
  --message-other-text: #1a202c;
  --danger-color: #dc3545;
  --danger-hover: #c82333;
  --success-color: #28a745;
  --success-hover: #218838;
  --shadow-sm: 0 2px 6px rgba(0, 0, 0, 0.08);
  --shadow-md: 0 4px 12px rgba(0, 0, 0, 0.12);
}

.dark-theme {
  --bg-color: #1a1f2b;
  --text-color: #e9ecef;
  --container-bg: #2c3241;
  --border-color: #3a4353;
  --accent-color: #4dabf7;
  --accent-hover: #339af0;
  --secondary-color: #adb5bd;
  --message-you-bg: linear-gradient(135deg, #4dabf7, #339af0);
  --message-you-text: #ffffff;
  --message-other-bg: #2d3343;
  --message-other-text: #e9ecef;
  --danger-color: #e74c3c;
  --danger-hover: #d62c1a;
  --success-color: #2ecc71;
  --success-hover: #27ae60;
  --shadow-sm: 0 2px 6px rgba(0, 0, 0, 0.3);
  --shadow-md: 0 4px 12px rgba(0, 0, 0, 0.4);
}

.chat-page {
  max-width: 720px;
  margin: 80px auto 0;
  padding: 24px;
  background: linear-gradient(135deg, var(--bg-color), #f0f2f5);
  border-radius: 16px;
  box-shadow: var(--shadow-md);
  min-height: calc(100vh - 96px);
  display: flex;
  flex-direction: column;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
  color: var(--text-color);
  transition: all 0.3s ease;
}

.chat-header {
  text-align: center;
  margin-bottom: 24px;
  padding-bottom: 16px;
  border-bottom: 2px solid var(--border-color);
  background: rgba(255, 255, 255, 0.1);
  border-radius: 8px;
  box-shadow: var(--shadow-sm);
}

.chat-header h2 {
  font-size: 2rem;
  font-weight: 700;
  color: var(--accent-color);
  margin: 0 0 12px;
  line-height: 1.2;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.trip-info {
  display: flex;
  flex-direction: column;
  gap: 8px;
  font-size: 0.95rem;
  color: var(--secondary-color);
  font-weight: 500;
  background: rgba(255, 255, 255, 0.05);
  padding: 8px 16px;
  border-radius: 8px;
}

.chat-messages {
  flex: 1;
  overflow-y: auto;
  padding: 24px;
  background: var(--container-bg);
  border-radius: 16px;
  margin-bottom: 20px;
  border: 1px solid var(--border-color);
  box-shadow: var(--shadow-sm);
  transition: all 0.3s ease;
  max-height: 70vh;
}

.message {
  display: flex;
  margin-bottom: 20px;
  animation: slideIn 0.5s ease-out;
}

.message-avatar img {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  object-fit: cover;
  border: 2px solid var(--accent-color);
  box-shadow: var(--shadow-sm);
  transition: transform 0.2s ease;
}

.message-avatar img:hover {
  transform: scale(1.1);
}

.message-content {
  margin-left: 16px;
  max-width: 70%;
  display: flex;
  flex-direction: column;
  gap: 4px;
  background: rgba(255, 255, 255, 0.1);
  padding: 8px;
  border-radius: 8px;
}

.message-sender {
  font-weight: 600;
  font-size: 1rem;
  color: var(--accent-color);
  text-transform: capitalize;
}

.message-text {
  background: var(--message-other-bg);
  color: var(--message-other-text);
  padding: 12px 16px;
  border-radius: 12px;
  font-size: 1.1rem;
  line-height: 1.6;
  word-break: break-word;
  box-shadow: var(--shadow-sm);
  transition: all 0.3s ease;
}

.message-you .message-content {
  margin-left: auto;
  margin-right: 16px;
  text-align: right;
}

.message-you .message-text {
  background: var(--message-you-bg);
  color: var(--message-you-text);
  border-radius: 12px 12px 4px 12px;
}

.message-other .message-text {
  border-radius: 12px 12px 12px 4px;
}

.message-time {
  font-size: 0.85rem;
  color: var(--secondary-color);
  margin-top: 4px;
  font-weight: 400;
  opacity: 0.8;
}

.message-status {
  font-size: 0.85rem;
  color: var(--success-color);
  font-style: italic;
  margin-top: 4px;
  animation: pulse 1.5s infinite;
}

.scroll-bottom-btn {
  position: fixed;
  bottom: 100px;
  right: 32px;
  background: var(--accent-color);
  color: var(--message-you-text);
  border: none;
  border-radius: 50%;
  width: 50px;
  height: 50px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  box-shadow: var(--shadow-md);
  transition: all 0.3s ease;
  z-index: 10;
}

.scroll-bottom-btn:hover,
.scroll-bottom-btn:focus-visible {
  background: var(--accent-hover);
  transform: scale(1.15) rotate(360deg);
  box-shadow: 0 6px 15px rgba(0, 0, 0, 0.2);
}

.scroll-bottom-btn:focus-visible {
  outline: 2px solid var(--accent-color);
  outline-offset: 2px;
}

.scroll-icon {
  font-size: 1.8rem;
}

.chat-input {
  display: flex;
  gap: 12px;
  padding: 16px;
  background: linear-gradient(135deg, var(--container-bg), #ffffff);
  border-radius: 16px;
  box-shadow: var(--shadow-md);
  border: 1px solid var(--border-color);
  transition: all 0.3s ease;
}

.chat-input input {
  flex: 1;
  padding: 14px 18px;
  border: 2px solid var(--border-color);
  border-radius: 10px;
  font-size: 1.1rem;
  color: var(--text-color);
  background: var(--bg-color);
  transition: all 0.3s ease;
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.05);
}

.chat-input input:focus {
  outline: none;
  border-color: var(--accent-color);
  box-shadow: 0 0 0 3px rgba(0, 123, 255, 0.2);
  transform: translateY(-2px);
}

.chat-input input:disabled {
  background-color: var(--border-color);
  cursor: not-allowed;
  opacity: 0.6;
}

.action-button {
  padding: 14px 22px;
  border: none;
  border-radius: 10px;
  cursor: pointer;
  font-size: 1rem;
  font-weight: 600;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 8px;
  box-shadow: var(--shadow-sm);
}

.action-button.send {
  background: var(--accent-color);
  color: var(--message-you-text);
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
}

.action-button.send:disabled {
  background: var(--secondary-color);
  color: var(--text-color);
  cursor: not-allowed;
  box-shadow: none;
  opacity: 0.7;
}

.action-button.send:hover:not(:disabled),
.action-button.send:focus-visible:not(:disabled) {
  background: var(--accent-hover);
  transform: translateY(-2px);
  box-shadow: var(--shadow-md);
}

.action-button.send:focus-visible {
  outline: 2px solid var(--accent-color);
  outline-offset: 2px;
}

.action-button.clear-input {
  background: var(--danger-color);
  color: var(--message-you-text);
  border: 2px solid var(--danger-color);
  font-weight: 700;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
}

.action-button.clear-input:hover:not(:disabled),
.action-button.clear-input:focus-visible:not(:disabled) {
  background: var(--danger-hover);
  border-color: var(--danger-hover);
  transform: translateY(-2px);
  box-shadow: var(--shadow-md);
}

.action-button.clear-input:disabled {
  background: var(--secondary-color);
  border-color: var(--secondary-color);
  color: var(--text-color);
  cursor: not-allowed;
  box-shadow: none;
  opacity: 0.7;
}

.send-text {
  display: none;
}

.send-icon {
  font-size: 1.4rem;
}

.loading-state, .error-state {
  padding: 32px;
  border-radius: 12px;
  text-align: center;
  color: var(--secondary-color);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
  border: 2px dashed var(--border-color);
  margin-bottom: 24px;
  background: linear-gradient(135deg, var(--container-bg), #e9ecef);
  box-shadow: var(--shadow-sm);
}

.loading-icon, .error-icon {
  font-size: 3.5rem;
  opacity: 0.9;
}

.loading-state p, .error-state p {
  font-size: 1.2rem;
  font-weight: 500;
}

.retry-button {
  padding: 12px 24px;
  background: var(--accent-color);
  color: var(--message-you-text);
  border: none;
  border-radius: 10px;
  cursor: pointer;
  font-size: 1rem;
  font-weight: 600;
  transition: all 0.3s ease;
  box-shadow: var(--shadow-sm);
}

.retry-button:hover,
.retry-button:focus-visible {
  background: var(--accent-hover);
  transform: translateY(-2px);
  box-shadow: var(--shadow-md);
}

.retry-button:focus-visible {
  outline: 2px solid var(--accent-color);
  outline-offset: 2px;
}

@keyframes slideIn {
  from { opacity: 0; transform: translateY(20px); }
  to { opacity: 1; transform: translateY(0); }
}

@keyframes pulse {
  0% { opacity: 0.6; }
  50% { opacity: 1; }
  100% { opacity: 0.6; }
}

@media (min-width: 480px) {
  .chat-page {
    margin-top: 96px;
    padding: 32px;
    min-height: calc(100vh - 128px);
  }
  
  .chat-header h2 {
    font-size: 2.5rem;
  }
  
  .trip-info {
    flex-direction: row;
    gap: 24px;
    justify-content: center;
    font-size: 1.1rem;
  }
  
  .send-text {
    display: inline;
  }
  
  .message-avatar img {
    width: 52px;
    height: 52px;
  }
  
  .message-content {
    max-width: 65%;
  }
}

@media (max-width: 360px) {
  .chat-page {
    padding: 16px;
    margin-top: 64px;
    max-width: 100%;
  }
  
  .chat-header h2 {
    font-size: 1.8rem;
  }
  
  .trip-info {
    font-size: 0.9rem;
    gap: 8px;
  }
  
  .message-text {
    padding: 10px 14px;
    font-size: 1rem;
  }
  
  .chat-input input {
    padding: 12px 14px;
    font-size: 1rem;
  }
  
  .action-button {
    padding: 12px 18px;
    font-size: 0.95rem;
  }
  
  .scroll-bottom-btn {
    bottom: 90px;
    right: 16px;
    width: 45px;
    height: 45px;
  }
}
</style>