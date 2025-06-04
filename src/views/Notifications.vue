<template>
  <div class="notifications-page" :class="{'dark-theme': isDarkTheme}">
    <div class="notifications-header">
      <h1>Уведомления</h1>
      <button @click="markAllAsRead" class="mark-all-read">
        Пометить все как прочитанные
      </button>
    </div>

    <div v-if="loading" class="loading-spinner">
      Загрузка...
    </div>

    <div v-else-if="notifications.length === 0" class="no-notifications">
      У вас нет уведомлений
    </div>

    <div v-else class="notifications-list">
      <div 
        v-for="notification in notifications" 
        :key="notification.id"
        class="notification-item"
        :class="{'unread': !notification.is_read}"
        @click="handleNotificationClick(notification)"
      >
        <div class="notification-icon">
          <span v-if="notification.type === 'booking'">📅</span>
          <span v-else-if="notification.type === 'cancel'">❌</span>
          <span v-else-if="notification.type === 'confirm'">✅</span>
          <span v-else>🔔</span>
        </div>
        <div class="notification-content">
          <p class="notification-message">{{ notification.message }}</p>
          <p class="notification-time">{{ formatTime(notification.created_at) }}</p>
        </div>
        <div v-if="!notification.is_read" class="unread-dot"></div>
      </div>

      <button 
        v-if="hasMore" 
        @click="loadMore" 
        class="load-more"
        :disabled="loadingMore"
      >
        {{ loadingMore ? 'Загрузка...' : 'Загрузить еще' }}
      </button>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import { API_CONFIG } from '@/config/api';
import Cookies from 'js-cookie';

export default {
  name: 'NotificationsPage',
  data() {
    return {
      notifications: [],
      loading: true,
      loadingMore: false,
      offset: 0,
      limit: 20,
      hasMore: true,
      isDarkTheme: false,
      ws: null
    };
  },
  created() {
    this.checkSavedTheme();
    this.fetchNotifications();
    this.setupWebSocket();
  },
  beforeUnmount() {
    if (this.ws) {
      this.ws.close();
    }
  },
  methods: {
    checkSavedTheme() {
      const savedTheme = localStorage.getItem('darkTheme');
      if (savedTheme !== null) {
        this.isDarkTheme = savedTheme === 'true';
      } else if (window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches) {
        this.isDarkTheme = true;
      }
    },
    
    async fetchNotifications() {
      try {
        const token = Cookies.get('token');
        if (!token) {
          this.$router.push('/login');
          return;
        }

        const userResponse = await axios.get(
          API_CONFIG.BASE_URL +'/user/get-id',
          { headers: { 'Authorization': `Bearer ${token}` } }
        );

        const response = await axios.get(`${API_CONFIG.BASE_URL}/notification/${userResponse.data.user_id}`, {
          params: {
            limit: this.limit,
            offset: this.offset
          },
          headers: {
            Authorization: `Bearer ${token}`
          }
        });

        this.notifications = [...this.notifications, ...response.data];
        this.hasMore = response.data.length === this.limit;
        this.offset += this.limit;
      } catch (error) {
        console.error('Ошибка при загрузке уведомлений:', error);
      } finally {
        this.loading = false;
        this.loadingMore = false;
      }
    },
    
    async loadMore() {
      this.loadingMore = true;
      await this.fetchNotifications();
    },
    
    async markAsRead(notificationId) {
      try {
        const token = Cookies.get('token');
        await axios.put(
          `${API_CONFIG.BASE_URL}/notification/${notificationId}/read`,
          {},
          {
            headers: {
              Authorization: `Bearer ${token}`
            }
          }
        );
        
        // Обновляем локальное состояние
        const index = this.notifications.findIndex(n => n.id === notificationId);
        if (index !== -1) {
          this.notifications[index].is_read = true;
        }
      } catch (error) {
        console.error('Ошибка при пометке уведомления как прочитанного:', error);
      }
    },
    
    async markAllAsRead() {
      try {
        const token = Cookies.get('token');
        await axios.put(
          `${API_CONFIG.BASE_URL}/notification/read-all`,
          {},
          {
            headers: {
              Authorization: `Bearer ${token}`
            }
          }
        );
        
        // Обновляем локальное состояние
        this.notifications = this.notifications.map(n => ({ ...n, is_read: true }));
      } catch (error) {
        console.error('Ошибка при пометке всех уведомлений как прочитанных:', error);
      }
    },
    
    formatTime(timestamp) {
      const date = new Date(timestamp);
      return date.toLocaleString();
    },
    
    handleNotificationClick(notification) {
      if (!notification.is_read) {
        this.markAsRead(notification.id);
      }
      
      // Навигация в зависимости от типа уведомления
      if (notification.trip_id) {
        this.$router.push(`/trip/${notification.trip_id}`);
      } else if (notification.booking_id) {
        this.$router.push(`/booking/${notification.booking_id}`);
      }
      // Добавьте другие условия для разных типов уведомлений
    },
    
    async setupWebSocket() {
      const token = Cookies.get('token');
      if (!token) return;

      const userResponse = await axios.get(
        API_CONFIG.BASE_URL +'/user/get-id',
        { headers: { 'Authorization': `Bearer ${token}` } }
      );
      
      //const wsProtocol = window.location.protocol === 'https:' ? 'wss://' : 'ws://';
      //const wsUrl = `${wsProtocol}${window.location.host}/ws`;
      const wsUrl = `wss://unigo-1rot.onrender.com/ws`;
      
      this.ws = new WebSocket(wsUrl);
      
      this.ws.onopen = () => {
        // Аутентификация через WebSocket
        this.ws.send(JSON.stringify({
          type: 'auth',
          user_id: userResponse.data.user_id
        }));
      };
      
      this.ws.onmessage = (event) => {
        const data = JSON.parse(event.data);
        
        if (data.type === 'new_notification') {
          // Добавляем новое уведомление в начало списка
          this.notifications.unshift(data.notification);
        }
      };
      
      this.ws.onerror = (error) => {
        console.error('WebSocket error:', error);
      };
      
      this.ws.onclose = () => {
        console.log('WebSocket connection closed');
      };
    }
  }
};
</script>

<style scoped>
.notifications-page {
  max-width: 800px;
  margin: 80px auto 20px;
  padding: 20px;
  background-color: var(--bg-color);
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.notifications-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  padding-bottom: 10px;
  border-bottom: 1px solid var(--border-color);
}

.mark-all-read {
  background-color: var(--primary-color);
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 14px;
  transition: background-color 0.3s;
}

.mark-all-read:hover {
  background-color: var(--primary-hover);
}

.loading-spinner,
.no-notifications {
  text-align: center;
  padding: 40px;
  color: var(--text-secondary);
}

.notifications-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.notification-item {
  display: flex;
  align-items: center;
  padding: 15px;
  background-color: var(--card-bg);
  border-radius: 8px;
  cursor: pointer;
  transition: transform 0.2s, box-shadow 0.2s;
  position: relative;
}

.notification-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.notification-item.unread {
  background-color: var(--unread-bg);
}

.notification-icon {
  font-size: 24px;
  margin-right: 15px;
  width: 40px;
  text-align: center;
}

.notification-content {
  flex: 1;
}

.notification-message {
  margin: 0;
  font-size: 16px;
  color: var(--text-color);
}

.notification-time {
  margin: 5px 0 0;
  font-size: 12px;
  color: var(--text-secondary);
}

.unread-dot {
  width: 10px;
  height: 10px;
  background-color: var(--primary-color);
  border-radius: 50%;
  margin-left: 10px;
}

.load-more {
  margin-top: 20px;
  padding: 10px;
  background-color: var(--secondary-bg);
  color: var(--text-color);
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.load-more:hover {
  background-color: var(--menu-item-hover);
}

.load-more:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}

/* Темная тема */
.dark-theme .notifications-page {
  background-color: var(--dropdown-bg);
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.3);
}

.dark-theme .notification-item.unread {
  background-color: var(--dark-unread-bg);
}
</style>

<style>
:root {
  --unread-bg: rgba(0, 66, 129, 0.1);
  --card-bg: #ffffff;
  --text-secondary: #777777;
}

.dark-theme {
  --unread-bg: rgba(74, 156, 255, 0.1);
  --dark-unread-bg: rgba(74, 156, 255, 0.2);
  --card-bg: #2d2d2d;
  --text-secondary: #aaaaaa;
}
</style>