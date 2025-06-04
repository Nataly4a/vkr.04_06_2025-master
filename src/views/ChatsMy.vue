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
:root {
  --bg-color: #ffffff;
  --text-color: #1a1a1a;
  --container-bg: #ffffff;
  --border-color: #e5e7eb;
  --accent-color: #004281;
  --accent-hover: #003366;
  --secondary-color: #6b7280;
  --menu-item-hover: #f3f4f6;
  --shadow-sm: 0 2px 4px rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 16px rgba(0, 0, 0, 0.1);
  --transition: all 0.2s ease;
  --font-family: 'Inter', system-ui, -apple-system, sans-serif;
}

.dark-theme {
  --bg-color: #0f172a;
  --text-color: #e2e8f0;
  --container-bg: #1e293b;
  --border-color: #475569;
  --accent-color: #60a5fa;
  --accent-hover: #3b82f6;
  --secondary-color: #94a3b8;
  --menu-item-hover: #334155;
}

.chat-page {
  background-color: var(--bg-color);
  min-height: 100vh;
  padding-bottom: 3rem;
  font-family: var(--font-family);
  line-height: 1.5;
}

.chat-container {
  max-width: 960px;
  margin: 5rem auto 0;
  padding: clamp(1rem, 3vw, 1.5rem);
  background: var(--container-bg);
  border-radius: 12px;
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
  font-size: clamp(1.5rem, 5vw, 2rem);
  font-weight: 600;
  color: var(--text-color);
  margin: 0;
  text-align: center;
}

.search-box {
  position: relative;
  max-width: 600px;
  margin: 0 auto;
  width: 100%;
}

.search-input {
  width: 100%;
  padding: clamp(0.6rem, 2vw, 0.75rem) clamp(0.6rem, 2vw, 0.75rem) clamp(0.6rem, 2vw, 0.75rem) clamp(2rem, 5vw, 2.5rem);
  border: 1px solid var(--border-color);
  border-radius: 8px;
  font-size: clamp(0.875rem, 2.5vw, 1rem);
  background-color: var(--container-bg);
  color: var(--text-color);
  transition: var(--transition);
}

.search-input:focus {
  outline: none;
  border-color: var(--accent-color);
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
  background-color: var(--bg-color);
}

.search-icon {
  position: absolute;
  left: clamp(0.5rem, 2vw, 0.75rem);
  top: 50%;
  transform: translateY(-50%);
  color: var(--secondary-color);
  font-size: clamp(0.875rem, 2.5vw, 1rem);
  pointer-events: none;
}

.chat-list {
  display: flex;
  flex-direction: column;
  gap: clamp(0.5rem, 2vw, 0.75rem);
}

.chat-item {
  padding: clamp(0.75rem, 2.5vw, 1rem);
  background: var(--container-bg);
  border-radius: 10px;
  border: 1px solid var(--border-color);
  cursor: pointer;
  transition: var(--transition);
  display: flex;
  align-items: center;
  gap: clamp(0.5rem, 2vw, 0.75rem);
}

.chat-item:hover {
  background-color: var(--menu-item-hover);
  transform: translateY(-2px);
  box-shadow: var(--shadow-sm);
  border-color: var(--accent-color);
}

.chat-item:focus-visible {
  outline: 2px solid var(--accent-color);
  outline-offset: 2px;
}

.active-chat {
  border-left: 4px solid var(--accent-color);
  background-color: rgba(59, 130, 246, 0.05) !important;
}

.chat-avatar {
  flex-shrink: 0;
  width: clamp(2rem, 8vw, 2.5rem);
  height: clamp(2rem, 8vw, 2.5rem);
  border-radius: 50%;
  background-color: var(--border-color);
  display: flex;
  align-items: center;
  justify-content: center;
}

.avatar-icon {
  font-size: clamp(1rem, 4vw, 1.25rem);
}

.chat-details {
  flex-grow: 1;
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.route-info {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.route {
  font-weight: 600;
  font-size: clamp(0.95rem, 3vw, 1.1rem);
  color: var(--text-color);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.trip-datetime {
  font-size: clamp(0.75rem, 2.5vw, 0.875rem);
  color: var(--secondary-color);
  font-weight: 500;
}

.chat-meta {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 0.25rem;
}

.last-message {
  color: var(--secondary-color);
  font-size: clamp(0.8rem, 2.5vw, 0.9rem);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 75%;
}

.unread-badge {
  background-color: #ef4444;
  color: white;
  border-radius: 50%;
  min-width: clamp(1.125rem, 4vw, 1.5rem);
  height: clamp(1.125rem, 4vw, 1.5rem);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: clamp(0.625rem, 2vw, 0.75rem);
  font-weight: 600;
  padding: 0 0.375rem;
  margin-left: 0.5rem;
}

.empty-state {
  text-align: center;
  padding: clamp(2rem, 10vw, 3.75rem) clamp(0.75rem, 3vw, 1.25rem);
  color: var(--secondary-color);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
}

.empty-icon {
  font-size: clamp(2rem, 8vw, 2.5rem);
  opacity: 0.7;
}

.empty-state p {
  margin: 0;
  font-size: clamp(1rem, 3vw, 1.1rem);
  font-weight: 500;
}

.refresh-btn {
  background-color: var(--accent-color);
  color: white;
  border: none;
  padding: clamp(0.5rem, 2vw, 0.625rem) clamp(1rem, 3vw, 1.5rem);
  border-radius: 8px;
  cursor: pointer;
  font-size: clamp(0.875rem, 2.5vw, 0.95rem);
  font-weight: 500;
  transition: var(--transition);
}

.refresh-btn:hover,
.refresh-btn:focus-visible {
  background-color: var(--accent-hover);
  transform: translateY(-1px);
  box-shadow: var(--shadow-sm);
}

.refresh-btn:focus-visible {
  outline: 2px solid var(--accent-color);
  outline-offset: 2px;
}

/* Медиа-запросы для адаптации */
@media (max-width: 1024px) {
  .chat-container {
    max-width: 90%;
    margin-top: 4.5rem;
  }

  .section-title {
    font-size: clamp(1.375rem, 4vw, 1.75rem);
  }

  .search-box {
    max-width: 80%;
  }
}

@media (max-width: 768px) {
  .chat-container {
    padding: clamp(0.75rem, 2vw, 1rem);
    margin-top: 4rem;
    border-radius: 0;
    box-shadow: none;
    border-top: 1px solid var(--border-color);
  }

  .chat-header {
    gap: 0.75rem;
  }

  .chat-item {
    padding: clamp(0.625rem, 2vw, 0.875rem);
  }

  .chat-avatar {
    width: clamp(1.75rem, 6vw, 2.25rem);
    height: clamp(1.75rem, 6vw, 2.25rem);
  }

  .route {
    font-size: clamp(0.875rem, 2.5vw, 1rem);
  }

  .trip-datetime {
    font-size: clamp(0.7rem, 2vw, 0.8rem);
  }

  .last-message {
    font-size: clamp(0.75rem, 2vw, 0.85rem);
    max-width: 70%;
  }

  .unread-badge {
    min-width: clamp(1rem, 3.5vw, 1.25rem);
    height: clamp(1rem, 3.5vw, 1.25rem);
    font-size: clamp(0.6rem, 1.8vw, 0.7rem);
  }
}

@media (max-width: 480px) {
  .chat-container {
    margin-top: 3.5rem;
    padding: clamp(0.5rem, 1.5vw, 0.75rem);
  }

  .chat-header {
    gap: 0.5rem;
  }

  .section-title {
    font-size: clamp(1.25rem, 3.5vw, 1.5rem);
  }

  .search-input {
    padding: clamp(0.5rem, 1.5vw, 0.625rem) clamp(0.5rem, 1.5vw, 0.625rem) clamp(0.5rem, 1.5vw, 0.625rem) clamp(1.75rem, 4vw, 2.25rem);
    font-size: clamp(0.8rem, 2vw, 0.9rem);
  }

  .search-icon {
    left: clamp(0.375rem, 1.5vw, 0.625rem);
    font-size: clamp(0.8rem, 2vw, 0.9rem);
  }

  .chat-item {
    padding: clamp(0.5rem, 1.5vw, 0.75rem);
    gap: clamp(0.375rem, 1.5vw, 0.625rem);
  }

  .chat-avatar {
    width: clamp(1.5rem, 5vw, 2rem);
    height: clamp(1.5rem, 5vw, 2rem);
  }

  .avatar-icon {
    font-size: clamp(0.875rem, 3vw, 1rem);
  }

  .route {
    font-size: clamp(0.8rem, 2vw, 0.95rem);
  }

  .trip-datetime {
    font-size: clamp(0.65rem, 1.8vw, 0.75rem);
  }

  .last-message {
    font-size: clamp(0.7rem, 1.8vw, 0.8rem);
    max-width: 65%;
  }

  .unread-badge {
    min-width: clamp(0.875rem, 3vw, 1.125rem);
    height: clamp(0.875rem, 3vw, 1.125rem);
    font-size: clamp(0.55rem, 1.5vw, 0.65rem);
  }

  .empty-state {
    padding: clamp(1.5rem, 8vw, 2.5rem) clamp(0.5rem, 2vw, 1rem);
  }

  .empty-icon {
    font-size: clamp(1.75rem, 6vw, 2rem);
  }

  .empty-state p {
    font-size: clamp(0.875rem, 2.5vw, 1rem);
  }

  .refresh-btn {
    padding: clamp(0.375rem, 1.5vw, 0.5rem) clamp(0.75rem, 2.5vw, 1.25rem);
    font-size: clamp(0.8rem, 2vw, 0.875rem);
  }
}

</style>