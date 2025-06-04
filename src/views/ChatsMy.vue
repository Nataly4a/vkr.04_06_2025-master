<template>
  <div class="chat-page">
    <AppNavbar />
    <div class="chat-container">
      <div class="chat-header">
        <h1 class="section-title">Мои чаты</h1>
        <div class="search-box">
          <input 
            type="text" 
            placeholder="Поиск чатов..." 
            v-model="searchQuery"
            class="search-input"
            aria-label="Поиск чатов"
          >
          <span class="search-icon">🔍</span>
        </div>
      </div>

      <div class="chat-list">
        <div 
          v-for="chat in filteredChats" 
          :key="chat.id" 
          class="chat-item"
          :class="{ 'active-chat': activeChatId === chat.id }"
          @click="selectChat(chat)"
          role="button"
          tabindex="0"
          @keydown.enter="selectChat(chat)"
        >
          <div class="chat-content">
            <div class="chat-avatar">
              <span class="avatar-icon">🚗</span>
            </div>
            <div class="chat-details">
              <div class="route-info">
                <span class="route">
                  {{ chat.departure_location }} → {{ chat.arrival_location }}
                </span>
                <span class="trip-datetime">
                  {{ formatDate(chat.departure_time) }} в {{ formatTime(chat.departure_time) }}
                </span>
              </div>
              
              <div class="chat-meta">
                <span class="last-message" v-if="chat.last_message">
                  {{ truncateMessage(chat.last_message.text, 40) }}
                </span>
                <span 
                  v-if="chat.unread_count > 0" 
                  class="unread-badge"
                >
                  {{ chat.unread_count }}
                </span>
              </div>
            </div>
          </div>
        </div>

        <div v-if="filteredChats.length === 0" class="empty-state">
          <span class="empty-icon">💬</span>
          <p>У вас пока нет активных чатов</p>
          <button class="refresh-btn" @click="loadChats" aria-label="Обновить список чатов">
            Обновить список
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import AppNavbar from "@/components/AppNavbar.vue";
import Cookies from 'js-cookie';
import { API_CONFIG } from '@/config/api'

export default {
  components: {
    AppNavbar,
  },
  data() {
    return {
      chats: [],
      token: Cookies.get('token') || '',
      searchQuery: '',
      activeChatId: null,
      isLoading: false,
      error: null
    };
  },
  computed: {
    filteredChats() {
      const query = this.searchQuery.toLowerCase();
      return this.chats.filter(chat => 
        chat.departure_location.toLowerCase().includes(query) ||
        chat.arrival_location.toLowerCase().includes(query) ||
        (chat.last_message && chat.last_message.text.toLowerCase().includes(query))
      );
    }
  },
  created() {
    this.loadChats();
  },
  methods: {
    async loadChats() {
      try {
        this.isLoading = true;
        this.error = null;
        
        const response = await axios.get(API_CONFIG.BASE_URL +'/chat/get', {
          headers: { 'Authorization': `Bearer ${this.token}` }
        });

        this.chats = response.data.map(chat => ({
          ...chat,
          departure_time: new Date(chat.departure_time),
          last_message: chat.last_message ? {
            ...chat.last_message,
            created_at: new Date(chat.last_message.created_at)
          } : null,
          unread_count: chat.unread_count || 0
        }));

        const userResponse = await axios.get(
          API_CONFIG.BASE_URL +'/user/get-id',
          { headers: { 'Authorization': `Bearer ${this.token}` } }
        );
        Cookies.set('userId', userResponse.data.user_id);

      } catch (error) {
        console.error("Ошибка при загрузке чатов:", error);
        this.error = "Не удалось загрузить чаты";
      } finally {
        this.isLoading = false;
      }
    },

    selectChat(chat) {
      this.activeChatId = chat.id;
      this.$router.push(`/chat/${chat.id}`);
    },

    formatTime(date) {
      if (!date) return '';
      return new Date(date).toLocaleTimeString('ru-RU', { 
        hour: '2-digit', 
        minute: '2-digit' 
      });
    },

    formatDate(date) {
      const options = { day: 'numeric', month: 'long' };
      return new Date(date).toLocaleDateString('ru-RU', options);
    },

    truncateMessage(text, length) {
      if (!text) return '';
      return text.length > length ? `${text.substring(0, length)}...` : text;
    }
  }
};
</script>

<style scoped>
/* Базовые стили и переменные */
/* Базовые стили и переменные */
:root {
  --bg-color: #f9fafb; /* Softer background for better contrast */
  --text-color: #111827;
  --container-bg: #ffffff;
  --border-color: #e5e7eb;
  --accent-color: #2563eb; /* Vibrant blue for modern feel */
  --accent-hover: #1d4ed8;
  --secondary-color: #6b7280;
  --menu-item-hover: #f1f5f9;
  --shadow-sm: 0 2px 6px rgba(0, 0, 0, 0.05);
  --shadow-md: 0 8px 24px rgba(0, 0, 0, 0.1);
  --transition: all 0.3s ease; /* Smoother transitions */
  --font-family: 'Inter', system-ui, -apple-system, sans-serif;
  --radius-sm: 8px;
  --radius-md: 12px;
}

.dark-theme {
  --bg-color: #111827;
  --text-color: #f3f4f6;
  --container-bg: #1f2937;
  --border-color: #374151;
  --accent-color: #3b82f6;
  --accent-hover: #2563eb;
  --secondary-color: #9ca3af;
  --menu-item-hover: #374151;
}

.chat-page {
  background-color: var(--bg-color);
  min-height: 100vh;
  padding-bottom: 4rem;
  font-family: var(--font-family);
  line-height: 1.6;
  overflow-x: hidden;
}

.chat-container {
  max-width: 960px;
  margin: 5rem auto 0;
  padding: clamp(1rem, 4vw, 1.5rem);
  background: var(--container-bg);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow-md);
  transition: background-color 0.3s ease, box-shadow 0.3s ease;
}

.chat-header {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  margin-bottom: 1.5rem;
  padding-bottom: 1rem;
  border-bottom: 1px solid var(--border-color);
}

.section-title {
  font-size: clamp(1.5rem, 5vw, 2.25rem);
  font-weight: 700;
  color: var(--text-color);
  margin: 0;
  text-align: center;
  letter-spacing: -0.02em;
}

.search-box {
  position: relative;
  max-width: 600px;
  margin: 0 auto;
  width: 100%;
}

.search-input {
  width: 100%;
  padding: clamp(0.75rem, 2.5vw, 0.875rem) clamp(0.75rem, 2.5vw, 0.875rem) clamp(0.75rem, 2.5vw, 0.875rem) clamp(2.5rem, 6vw, 3rem);
  border: 1px solid var(--border-color);
  border-radius: var(--radius-sm);
  font-size: clamp(0.9rem, 2.5vw, 1rem);
  background-color: var(--container-bg);
  color: var(--text-color);
  transition: var(--transition);
  box-shadow: var(--shadow-sm);
}

.search-input:focus {
  outline: none;
  border-color: var(--accent-color);
  box-shadow: 0 0 0 4px rgba(59, 130, 246, 0.15);
  background-color: var(--bg-color);
}

.search-icon {
  position: absolute;
  left: clamp(0.75rem, 2.5vw, 1rem);
  top: 50%;
  transform: translateY(-50%);
  color: var(--secondary-color);
  font-size: clamp(1rem, 3vw, 1.125rem);
  pointer-events: none;
  transition: color 0.3s ease;
}

.search-input:focus + .search-icon {
  color: var(--accent-color);
}

.chat-list {
  display: flex;
  flex-direction: column;
  gap: clamp(0.5rem, 2vw, 0.75rem);
}

.chat-item {
  padding: clamp(0.875rem, 3vw, 1.25rem);
  background: var(--container-bg);
  border-radius: var(--radius-sm);
  border: 1px solid var(--border-color);
  cursor: pointer;
  transition: var(--transition);
  display: flex;
  align-items: center;
  gap: clamp(0.75rem, 2.5vw, 1rem);
  animation: fadeIn 0.3s ease;
}

.chat-item:hover {
  background-color: var(--menu-item-hover);
  transform: translateY(-3px);
  box-shadow: var(--shadow-md);
  border-color: var(--accent-color);
}

.chat-item:focus-visible {
  outline: 2px solid var(--accent-color);
  outline-offset: 2px;
}

.active-chat {
  border-left: 4px solid var(--accent-color);
  background-color: rgba(59, 130, 246, 0.08) !important;
  box-shadow: var(--shadow-sm);
}

.chat-avatar {
  flex-shrink: 0;
  width: clamp(2.25rem, 8vw, 3rem);
  height: clamp(2.25rem, 8vw, 3rem);
  border-radius: 50%;
  background-color: var(--accent-color);
  display: flex;
  align-items: center;
  justify-content: center;
  transition: transform 0.3s ease;
}

.chat-item:hover .chat-avatar {
  transform: scale(1.1);
}

.avatar-icon {
  font-size: clamp(1.25rem, 4vw, 1.5rem);
  color: #ffffff;
}

.chat-details {
  flex-grow: 1;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.route-info {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.route {
  font-weight: 600;
  font-size: clamp(1rem, 3vw, 1.2rem);
  color: var(--text-color);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.trip-datetime {
  font-size: clamp(0.8rem, 2.5vw, 0.95rem);
  color: var(--secondary-color);
  font-weight: 500;
}

.chat-meta {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 0.375rem;
}

.last-message {
  color: var(--secondary-color);
  font-size: clamp(0.85rem, 2.5vw, 0.95rem);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 80%;
}

.unread-badge {
  background-color: #ef4444;
  color: white;
  border-radius: 50%;
  min-width: clamp(1.25rem, 4vw, 1.75rem);
  height: clamp(1.25rem, 4vw, 1.75rem);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: clamp(0.7rem, 2vw, 0.85rem);
  font-weight: 600;
  padding: 0 0.5rem;
  margin-left: 0.75rem;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.empty-state {
  text-align: center;
  padding: clamp(2.5rem, 12vw, 4rem) clamp(1rem, 3vw, 1.5rem);
  color: var(--secondary-color);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1.25rem;
  background: var(--menu-item-hover);
  border-radius: var(--radius-md);
  margin-top: 1rem;
}

.empty-icon {
  font-size: clamp(2.5rem, 10vw, 3rem);
  opacity: 0.8;
  color: var(--accent-color);
}

.empty-state p {
  margin: 0;
  font-size: clamp(1rem, 3vw, 1.2rem);
  font-weight: 500;
}

.refresh-btn {
  background-color: var(--accent-color);
  color: white;
  border: none;
  padding: clamp(0.625rem, 2.5vw, 0.75rem) clamp(1.25rem, 4vw, 2rem);
  border-radius: var(--radius-sm);
  cursor: pointer;
  font-size: clamp(0.9rem, 2.5vw, 1rem);
  font-weight: 600;
  transition: var(--transition);
  box-shadow: var(--shadow-sm);
}

.refresh-btn:hover,
.refresh-btn:focus-visible {
  background-color: var(--accent-hover);
  transform: translateY(-2px);
  box-shadow: var(--shadow-md);
}

.refresh-btn:focus-visible {
  outline: 2px solid var(--accent-color);
  outline-offset: 2px;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

/* Медиа-запросы для адаптации */
@media (max-width: 1024px) {
  .chat-container {
    max-width: 95%;
    margin-top: 4rem;
  }

  .section-title {
    font-size: clamp(1.375rem, 4vw, 1.875rem);
  }

  .search-box {
    max-width: 85%;
  }

  .chat-item {
    padding: clamp(0.75rem, 2.5vw, 1rem);
  }
}

@media (max-width: 768px) {
  .chat-container {
    max-width: 100%;
    margin: 3.5rem 0.5rem 0;
    padding: clamp(0.75rem, 2vw, 1rem);
    border-radius: 0;
    box-shadow: none;
    border: 1px solid var(--border-color);
  }

  .chat-header {
    gap: 0.75rem;
    padding: 0.75rem;
  }

  .section-title {
    font-size: clamp(1.25rem, 3.5vw, 1.625rem);
  }

  .search-input {
    padding: clamp(0.625rem, 2vw, 0.75rem) clamp(0.625rem, 2vw, 0.75rem) clamp(0.625rem, 2vw, 0.75rem) clamp(2rem, 5vw, 2.5rem);
  }

  .chat-item {
    padding: clamp(0.625rem, 2vw, 0.875rem);
    gap: clamp(0.5rem, 2vw, 0.75rem);
  }

  .chat-avatar {
    width: clamp(2rem, 6vw, 2.5rem);
    height: clamp(2rem, 6vw, 2.5rem);
  }

  .route {
    font-size: clamp(0.9rem, 2.5vw, 1.05rem);
  }

  .trip-datetime {
    font-size: clamp(0.75rem, 2vw, 0.85rem);
  }

  .last-message {
    font-size: clamp(0.8rem, 2vw, 0.9rem);
    max-width: 75%;
  }

  .unread-badge {
    min-width: clamp(1.125rem, 3.5vw, 1.5rem);
    height: clamp(1.625rem, 1.125rem);
    font-size: clamp(0.65rem, 1.8vw, 0.75rem);
  }

  .empty-state {
    margin: 1rem 0.5rem;
    padding: clamp(1.5rem, 3vw, 2rem);
  }
}

@media (max-width: 480px) {
  .chat-page {
    padding-bottom: 2.5rem;
  }

  .chat-container {
    margin: 3rem 0;
    padding: clamp(0.5rem, 1.5vw, 0.75rem);
    border-left: none;
    border-right: none;
  }

  .chat-header {
    gap: 0.5rem;
    padding: 0.5rem;
  }

  .section-title {
    font-size: clamp(1.125rem, 5vw, 1.375rem);
  }

  .search-box {
    max-width: 100%;
    padding: 0 0.5rem;
  }

  .search-input {
    padding: clamp(0.5rem, 1.5vw, 0.625rem);
    font-size: clamp(0.85rem, 2vw, 0.95rem);
    border-radius: 6px;
  }

  .search-icon {
    left: clamp(0.375rem, 1vw, 0.5rem);
    font-size: clamp(0.75rem, 2vw, 0.875rem);
  }

  .chat-item {
    padding: clamp(0.5rem, 2vw, 0.75rem);
    gap: clamp(0.375rem, 1.5vw, 0.625rem);
    border-radius: 8px;
  }

  .chat-avatar {
    width: clamp(1.75rem, 6vw, 5vw, 2.25rem);
    height: clamp(1.75rem, 6vw, 5vw,);
  }

  .avatar-icon {
    font-size: clamp(0.875rem, 3vw, 1rem);
  }

  .route {
    font-size: clamp(0.85rem, 2vw, 1rem);
  }

  .trip-datetime {
    font-size: clamp(0.7rem, 1.8vw, 0.8rem);
  }

  .last-message {
    font-size: clamp(0.75rem, 1.8vw, 0.85rem);
    max-width: 70%;
  }

  .unread-badge {
    min-width: clamp(1rem, 3vw, 1.25rem);
    height: clamp(1rem, 3vw, 1.25rem);
    font-size: clamp(0.6rem, 1.5vw, 0.7rem);
    padding: 0 0.375rem;
  }

  .empty-state {
    padding: clamp(1.25rem, 6vw, 2rem);
    border-radius: 8px;
  }

  .empty-icon {
    font-size: clamp(2rem, 6vw, 2.5rem);
  }

  .empty-state p {
    font-size: clamp(0.9rem, 2.5vw, 1rem);
  }

  .refresh-btn {
    padding: clamp(0.5rem, 1.5vw, 0.625rem) clamp(1rem, 2.5vw, 1.5rem);
    font-size: clamp(0.85rem, 2vw, 0.9rem);
    border-radius: 6px;
  }
}

@media (max-width: 360px) {
  .chat-container {
    margin-top: 2.5rem;
    padding: clamp(0.375rem, 1vw, 0.5rem);
  }

  .section-title {
    font-size: clamp(1rem, 3vw, 1.25rem);
  }

  .search-input {
    padding: clamp(0.375rem, 1vw, 0.5rem) clamp(0.375rem, 1vw, 0.5rem) clamp(0.375rem, 1vw, 0.5rem) clamp(1.5rem, 4vw, 2rem);
    font-size: clamp(0.8rem, 1.8vw, 0.875rem);
  }

  .search-icon {
    left: clamp(0.25rem, 0.8vw, 0.375rem);
    font-size: clamp(0.7rem, 1.8vw, 0.8rem);
  }

  .chat-item {
    padding: clamp(0.375rem, 1vw, 0.625rem);
    gap: clamp(0.25rem, 1vw, 0.5rem);
  }

  .chat-avatar {
    width: clamp(1.5rem, 4vw, 1.75rem);
    height: clamp(1.5rem, 4vw, 1.75rem);
  }

  .avatar-icon {
    font-size: clamp(0.75rem, 2.5vw, 0.875rem);
  }

  .route {
    font-size: clamp(0.8rem, 1.8vw, 0.9rem);
  }

  .trip-datetime {
    font-size: clamp(0.65rem, 1.5vw, 0.75rem);
  }

  .last-message {
    font-size: clamp(0.7rem, 1.5vw, 0.8rem);
    max-width: 65%;
  }

  .unread-badge {
    min-width: clamp(0.875rem, 2.5vw, 1rem);
    height: clamp(0.875rem, 2.5vw, 1rem);
    font-size: clamp(0.55rem, 1.2vw, 0.65rem);
  }

  .empty-state {
    padding: clamp(1rem, 5vw, 1.5rem);
  }

  .empty-icon {
    font-size: clamp(1.75rem, 5vw, 2rem);
  }

  .refresh-btn {
    padding: clamp(0.375rem, 1vw, 0.5rem) clamp(0.75rem, 2vw, 1rem);
    font-size: clamp(0.8rem, 1.8vw, 0.85rem);
  }
}

</style>