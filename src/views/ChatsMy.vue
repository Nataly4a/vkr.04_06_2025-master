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
  --bg-color: #f9fafb;
  --text-color: #111827;
  --container-bg: #ffffff;
  --border-color: #e5e7eb;
  --accent-color: #2563eb;
  --accent-hover: #1d4ed8;
  --secondary-color: #6b7280;
  --menu-item-hover: #f1f5f9;
  --shadow-sm: 0 2px 6px rgba(0, 0, 0, 0.05);
  --shadow-md: 0 8px 24px rgba(0, 0, 0, 0.1);
  --transition: all 0.3s ease;
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
  padding-bottom: 3rem;
  font-family: var(--font-family);
  line-height: 1.6;
  overflow-x: hidden;
  -webkit-overflow-scrolling: touch; /* Smooth scrolling on iOS */
}

.chat-container {
  max-width: 960px;
  margin: 4.5rem auto 0;
  padding: clamp(1rem, 3vw, 1.5rem);
  background: var(--container-bg);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow-md);
  transition: background-color 0.3s ease, box-shadow 0.3s ease;
}

.chat-header {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  margin-bottom: 1.25rem;
  padding-bottom: 0.75rem;
  border-bottom: 1px solid var(--border-color);
}

.section-title {
  font-size: clamp(1.5rem, 5vw, 2rem);
  font-weight: 700;
  color: var(--text-color);
  margin: 0;
  text-align: center;
  letter-spacing: -0.01em;
}

.search-box {
  position: relative;
  max-width: 600px;
  margin: 0 auto;
  width: 100%;
}

.search-input {
  width: 100%;
  padding: clamp(0.625rem, 2vw, 0.75rem) clamp(0.625rem, 2vw, 0.75rem) clamp(0.625rem, 2vw, 0.75rem) clamp(2rem, 5vw, 2.5rem);
  border: 1px solid var(--border-color);
  border-radius: var(--radius-sm);
  font-size: clamp(0.875rem, 2.5vw, 1rem);
  background-color: var(--container-bg);
  color: var(--text-color);
  transition: var(--transition);
  box-shadow: var(--shadow-sm);
  -webkit-appearance: none; /* Remove default iOS styling */
}

.search-input:focus {
  outline: none;
  border-color: var(--accent-color);
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
  background-color: var(--bg-color);
}

.search-icon {
  position: absolute;
  left: clamp(0.5rem, 1.5vw, 0.75rem);
  top: 50%;
  transform: translateY(-50%);
  color: var(--secondary-color);
  font-size: clamp(0.875rem, 2.5vw, 1rem);
  pointer-events: none;
  transition: color 0.3s ease;
}

.search-input:focus + .search-icon {
  color: var(--accent-color);
}

.chat-list {
  display: flex;
  flex-direction: column;
  gap: clamp(0.5rem, 1.5vw, 0.75rem);
}

.chat-item {
  padding: clamp(0.75rem, 2.5vw, 1rem);
  background: var(--container-bg);
  border-radius: var(--radius-sm);
  border: 1px solid var(--border-color);
  cursor: pointer;
  transition: var(--transition);
  display: flex;
  align-items: center;
  gap: clamp(0.5rem, 2vw, 0.75rem);
  animation: fadeIn 0.3s ease;
  min-height: 4rem; /* Ensure touch target size */
  touch-action: manipulation; /* Prevent double-tap zoom */
}

.chat-item:hover {
  background-color: var(--menu-item-hover);
  transform: translateY(-2px);
  box-shadow: var(--shadow-md);
  border-color: var(--accent-color);
}

.chat-item:focus-visible {
  outline: 2px solid var(--accent-color);
  outline-offset: 2px;
}

.active-chat {
  border-left: 3px solid var(--accent-color);
  background-color: rgba(59, 130, 246, 0.06) !important;
  box-shadow: var(--shadow-sm);
}

.chat-avatar {
  flex-shrink: 0;
  width: clamp(2rem, 7vw, 2.5rem);
  height: clamp(2rem, 7vw, 2.5rem);
  border-radius: 50%;
  background-color: var(--accent-color);
  display: flex;
  align-items: center;
  justify-content: center;
  transition: transform 0.3s ease;
}

.chat-item:hover .chat-avatar {
  transform: scale(1.05);
}

.avatar-icon {
  font-size: clamp(1rem, 3.5vw, 1.25rem);
  color: #ffffff;
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
  font-size: clamp(0.95rem, 2.8vw, 1.1rem);
  color: var(--text-color);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.trip-datetime {
  font-size: clamp(0.75rem, 2.2vw, 0.875rem);
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
  font-size: clamp(0.8rem, 2.2vw, 0.9rem);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 75%;
}

.unread-badge {
  background-color: #ef4444;
  color: white;
  border-radius: 50%;
  min-width: clamp(1.125rem, 3.5vw, 1.5rem);
  height: clamp(1.125rem, 3.5vw, 1.5rem);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: clamp(0.65rem, 1.8vw, 0.75rem);
  font-weight: 600;
  padding: 0 0.375rem;
  margin-left: 0.5rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.empty-state {
  text-align: center;
  padding: clamp(2rem, 10vw, 3rem) clamp(0.75rem, 2.5vw, 1rem);
  color: var(--secondary-color);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
  background: var(--menu-item-hover);
  border-radius: var(--radius-sm);
  margin: 0.75rem;
}

.empty-icon {
  font-size: clamp(2rem, 8vw, 2.5rem);
  opacity: 0.8;
  color: var(--accent-color);
}

.empty-state p {
  margin: 0;
  font-size: clamp(0.95rem, 2.8vw, 1.1rem);
  font-weight: 500;
}

.refresh-btn {
  background-color: var(--accent-color);
  color: white;
  border: none;
  padding: clamp(0.5rem, 2vw, 0.625rem) clamp(1rem, 3.5vw, 1.5rem);
  border-radius: var(--radius-sm);
  cursor: pointer;
  font-size: clamp(0.875rem, 2.5vw, 0.95rem);
  font-weight: 600;
  transition: var(--transition);
  box-shadow: var(--shadow-sm);
  min-height: 2.5rem; /* Ensure touch target size */
  touch-action: manipulation;
}

.refresh-btn:hover,
.refresh-btn:focus-visible {
  background-color: var(--accent-hover);
  transform: translateY(-1px);
  box-shadow: var(--shadow-md);
}

.refresh-btn:focus-visible {
  outline: 2px solid var(--accent-color);
  outline-offset: 2px;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(8px); }
  to { opacity: 1; transform: translateY(0); }
}

/* Медиа-запросы для мобильной адаптации */
@media (max-width: 768px) {
  .chat-page {
    padding-bottom: 2rem;
  }

  .chat-container {
    max-width: 100%;
    margin: 3rem 0;
    padding: clamp(0.5rem, 1.5vw, 0.75rem);
    border-radius: 0;
    box-shadow: none;
    border-left: none;
    border-right: none;
    border-top: 1px solid var(--border-color);
    border-bottom: 1px solid var(--border-color);
  }

  .chat-header {
    gap: 0.5rem;
    padding: clamp(0.5rem, 1.5vw, 0.75rem);
  }

  .section-title {
    font-size: clamp(1.25rem, 4vw, 1.5rem);
  }

  .search-box {
    max-width: 100%;
    padding: 0 clamp(0.5rem, 1.5vw, 0.75rem);
  }

  .search-input {
    padding: clamp(0.5rem, 1.5vw, 0.625rem) clamp(0.5rem, 1.5vw, 0.625rem) clamp(0.5rem, 1.5vw, 0.625rem) clamp(1.75rem, 4.5vw, 2rem);
    font-size: clamp(0.85rem, 2.2vw, 0.95rem);
    border-radius: 6px;
  }

  .search-icon {
    left: clamp(0.375rem, 1vw, 0.5rem);
    font-size: clamp(0.8rem, 2vw, 0.9rem);
  }

  .chat-list {
    gap: clamp(0.375rem, 1vw, 0.5rem);
  }

  .chat-item {
    padding: clamp(0.5rem, 1.5vw, 0.75rem);
    gap: clamp(0.375rem, 1.5vw, 0.625rem);
    border-radius: 6px;
    min-height: 3.5rem;
  }

  .chat-avatar {
    width: clamp(1.75rem, 5.5vw, 2rem);
    height: clamp(1.75rem, 5.5vw, 2rem);
  }

  .avatar-icon {
    font-size: clamp(0.875rem, 3vw, 1rem);
  }

  .route {
    font-size: clamp(0.85rem, 2.5vw, 1rem);
  }

  .trip-datetime {
    font-size: clamp(0.7rem, 2vw, 0.8rem);
  }

  .last-message {
    font-size: clamp(0.75rem, 2vw, 0.85rem);
    max-width: 70%;
  }

  .unread-badge {
    min-width: clamp(1rem, 3vw, 1.25rem);
    height: clamp(1rem, 3vw, 1.25rem);
    font-size: clamp(0.6rem, 1.5vw, 0.7rem);
    padding: 0 0.25rem;
  }

  .empty-state {
    margin: 0.5rem 0;
    padding: clamp(1.25rem, 6vw, 1.75rem);
    border-radius: 6px;
  }

  .empty-icon {
    font-size: clamp(1.75rem, 6vw, 2rem);
  }

  .empty-state p {
    font-size: clamp(0.9rem, 2.5vw, 1rem);
  }

  .refresh-btn {
    padding: clamp(0.375rem, 1.5vw, 0.5rem) clamp(0.75rem, 2.5vw, 1.25rem);
    font-size: clamp(0.8rem, 2.2vw, 0.9rem);
    border-radius: 6px;
    min-height: 2.25rem;
  }
}

@media (max-width: 480px) {
  .chat-page {
    padding-bottom: 1.5rem;
  }

  .chat-container {
    margin: 2.5rem 0;
    padding: clamp(0.375rem, 1vw, 0.5rem);
  }

  .chat-header {
    gap: 0.375rem;
    padding: clamp(0.375rem, 1vw, 0.5rem);
  }

  .section-title {
    font-size: clamp(1.125rem, 3.5vw, 1.375rem);
  }

  .search-box {
    padding: 0 clamp(0.375rem, 1vw, 0.5rem);
  }

  .search-input {
    padding: clamp(0.375rem, 1vw, 0.5rem) clamp(0.375rem, 1vw, 0.5rem) clamp(0.375rem, 1vw, 0.5rem) clamp(1.5rem, 4vw, 1.75rem);
    font-size: clamp(0.8rem, 2vw, 0.9rem);
  }

  .search-icon {
    left: clamp(0.25rem, 0.8vw, 0.375rem);
    font-size: clamp(0.75rem, 1.8vw, 0.85rem);
  }

  .chat-item {
    padding: clamp(0.375rem, 1vw, 0.625rem);
    gap: clamp(0.25rem, 1vw, 0.5rem);
    min-height: 3rem;
  }

  .chat-avatar {
    width: clamp(1.5rem, 5vw, 1.75rem);
    height: clamp(1.5rem, 5vw, 1.75rem);
  }

  .avatar-icon {
    font-size: clamp(0.75rem, 2.5vw, 0.875rem);
  }

  .route {
    font-size: clamp(0.8rem, 2.2vw, 0.95rem);
  }

  .trip-datetime {
    font-size: clamp(0.65rem, 1.8vw, 0.75rem);
  }

  .last-message {
    font-size: clamp(0.7rem, 1.8vw, 0.8rem);
    max-width: 65%;
  }

  .unread-badge {
    min-width: clamp(0.875rem, 2.5vw, 1rem);
    height: clamp(0.875rem, 2.5vw, 1rem);
    font-size: clamp(0.55rem, 1.2vw, 0.65rem);
    padding: 0 0.2rem;
  }

  .empty-state {
    padding: clamp(1rem, 5vw, 1.5rem);
    margin: 0.375rem 0;
  }

  .empty-icon {
    font-size: clamp(1.5rem, 5vw, 1.75rem);
  }

  .empty-state p {
    font-size: clamp(0.85rem, 2.2vw, 0.95rem);
  }

  .refresh-btn {
    padding: clamp(0.375rem, 1vw, 0.5rem) clamp(0.625rem, 2vw, 1rem);
    font-size: clamp(0.75rem, 2vw, 0.85rem);
    min-height: 2rem;
  }
}

@media (max-width: 360px) {
  .chat-page {
    padding-bottom: 1rem;
  }

  .chat-container {
    margin: 2rem 0;
    padding: clamp(0.25rem, 0.8vw, 0.375rem);
  }

  .chat-header {
    gap: 0.25rem;
    padding: clamp(0.25rem, 0.8vw, 0.375rem);
  }

  .section-title {
    font-size: clamp(1rem, 3vw, 1.25rem);
  }

  .search-box {
    padding: 0 clamp(0.25rem, 0.8vw, 0.375rem);
  }

  .search-input {
    padding: clamp(0.25rem, 0.8vw, 0.375rem) clamp(0.25rem, 0.8vw, 0.375rem) clamp(0.25rem, 0.8vw, 0.375rem) clamp(1.25rem, 3.5vw, 1.5rem);
    font-size: clamp(0.75rem, 1.8vw, 0.85rem);
    border-radius: 4px;
  }

  .search-icon {
    left: clamp(0.2rem, 0.6vw, 0.25rem);
    font-size: clamp(0.7rem, 1.6vw, 0.8rem);
  }

  .chat-item {
    padding: clamp(0.25rem, 0.8vw, 0.5rem);
    gap: clamp(0.2rem, 0.8vw, 0.375rem);
    min-height: 2.75rem;
    border-radius: 4px;
  }

  .chat-avatar {
    width: clamp(1.25rem, 4vw, 1.5rem);
    height: clamp(1.25rem, 4vw, 1.5rem);
  }

  .avatar-icon {
    font-size: clamp(0.65rem, 2vw, 0.75rem);
  }

  .route {
    font-size: clamp(0.75rem, 2vw, 0.9rem);
  }

  .trip-datetime {
    font-size: clamp(0.6rem, 1.6vw, 0.7rem);
  }

  .last-message {
    font-size: clamp(0.65rem, 1.6vw, 0.75rem);
    max-width: 60%;
  }

  .unread-badge {
    min-width: clamp(0.75rem, 2vw, 0.875rem);
    height: clamp(0.75rem, 2vw, 0.875rem);
    font-size: clamp(0.5rem, 1vw, 0.6rem);
    padding: 0 0.15rem;
  }

  .empty-state {
    padding: clamp(0.75rem, 4vw, 1.25rem);
    margin: 0.25rem 0;
    border-radius: 4px;
  }

  .empty-icon {
    font-size: clamp(1.25rem, 4vw, 1.5rem);
  }

  .empty-state p {
    font-size: clamp(0.8rem, 2vw, 0.9rem);
  }

  .refresh-btn {
    padding: clamp(0.25rem, 0.8vw, 0.375rem) clamp(0.5rem, 1.5vw, 0.75rem);
    font-size: clamp(0.7rem, 1.8vw, 0.8rem);
    min-height: 1.75rem;
    border-radius: 4px;
  }
}

@media (max-width: 320px) {
  .chat-page {
    padding-bottom: 0.75rem;
  }

  .chat-container {
    margin: 1.75rem 0;
    padding: clamp(0.2rem, 0.6vw, 0.25rem);
  }

  .chat-header {
    gap: 0.2rem;
    padding: clamp(0.2rem, 0.6vw, 0.25rem);
  }

  .section-title {
    font-size: clamp(0.875rem, 2.8vw, 1.125rem);
  }

  .search-box {
    padding: 0 clamp(0.2rem, 0.6vw, 0.25rem);
  }

  .search-input {
    padding: clamp(0.2rem, 0.6vw, 0.25rem) clamp(0.2rem, 0.6vw, 0.25rem) clamp(0.2rem, 0.6vw, 0.25rem) clamp(1rem, 3vw, 1.25rem);
    font-size: clamp(0.7rem, 1.6vw, 0.8rem);
  }

  .search-icon {
    left: clamp(0.15rem, 0.5vw, 0.2rem);
    font-size: clamp(0.65rem, 1.4vw, 0.75rem);
  }

  .chat-item {
    padding: clamp(0.2rem, 0.6vw, 0.375rem);
    gap: clamp(0.15rem, 0.6vw, 0.25rem);
    min-height: 2.5rem;
  }

  .chat-avatar {
    width: clamp(1rem, 3.5vw, 1.25rem);
    height: clamp(1rem, 3.5vw, 1.25rem);
  }

  .avatar-icon {
    font-size: clamp(0.6rem, 1.8vw, 0.7rem);
  }

  .route {
    font-size: clamp(0.7rem, 1.8vw, 0.85rem);
  }

  .trip-datetime {
    font-size: clamp(0.55rem, 1.4vw, 0.65rem);
  }

  .last-message {
    font-size: clamp(0.6rem, 1.4vw, 0.7rem);
    max-width: 55%;
  }

  .unread-badge {
    min-width: clamp(0.65rem, 1.8vw, 0.75rem);
    height: clamp(0.65rem, 1.8vw, 0.75rem);
    font-size: clamp(0.45rem, 0.8vw, 0.55rem);
    padding: 0 0.1rem;
  }

  .empty-state {
    padding: clamp(0.5rem, 3vw, 1rem);
    margin: 0.2rem 0;
  }

  .empty-icon {
    font-size: clamp(1rem, 3.5vw, 1.25rem);
  }

  .empty-state p {
    font-size: clamp(0.75rem, 1.8vw, 0.85rem);
  }

  .refresh-btn {
    padding: clamp(0.2rem, 0.6vw, 0.25rem) clamp(0.375rem, 1vw, 0.5rem);
    font-size: clamp(0.65rem, 1.6vw, 0.75rem);
    min-height: 1.5rem;
  }
}

</style>