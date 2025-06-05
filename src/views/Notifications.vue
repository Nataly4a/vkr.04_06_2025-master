<template>
  <AppNavbar />
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
import AppNavbar from "@/components/AppNavbar.vue";

export default {
  name: 'NotificationsPage',
  components: {
    AppNavbar,
  },
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
      
      const wsUrl = `wss://unigo-1rot.onrender.com/ws`;
      
      this.ws = new WebSocket(wsUrl);
      
      this.ws.onopen = () => {
        this.ws.send(JSON.stringify({
          type: 'auth',
          user_id: userResponse.data.user_id
        }));
      };
      
      this.ws.onmessage = (event) => {
        const data = JSON.parse(event.data);
        
        if (data.type === 'new_notification') {
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
  max-width: 900px;
  margin: 90px auto 30px;
  padding: 25px;
  background-color: var(--bg-color);
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
}

.notifications-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 25px;
  padding-bottom: 15px;
  border-bottom: 2px solid var(--border-color);
  background: linear-gradient(to right, transparent, rgba(255, 255, 255, 0.1));
  border-radius: 8px 8px 0 0;
}

.notifications-header h1 {
  font-size: 2.2rem;
  font-weight: 700;
  color: var(--text-primary);
  margin: 0;
  text-transform: uppercase;
  letter-spacing: 1.5px;
  animation: fadeIn 0.5s ease-in;
}

.mark-all-read {
  background-color: var(--primary-color);
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 15px;
  font-weight: 500;
  transition: all 0.3s ease;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
}

.mark-all-read:hover {
  background-color: var(--primary-hover);
  transform: translateY(-2px);
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
}

.mark-all-read:active {
  transform: translateY(0);
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.loading-spinner,
.no-notifications {
  text-align: center;
  padding: 50px 20px;
  color: var(--text-secondary);
  font-size: 1.1rem;
  font-weight: 500;
  background: var(--card-bg);
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
}

.notifications-list {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.notification-item {
  display: flex;
  align-items: center;
  padding: 20px;
  background-color: var(--card-bg);
  border-radius: 10px;
  cursor: pointer;
  transition: all 0.3s ease;
  position: relative;
  border: 1px solid var(--border-color-light);
}

.notification-item:hover {
  transform: translateY(-3px);
  box-shadow: 0 6px 15px rgba(0, 0, 0, 0.1);
  background-color: var(--card-hover);
}

.notification-item.unread {
  background-color: var(--unread-bg);
  border-color: var(--primary-color);
}

.notification-icon {
  font-size: 28px;
  margin-right: 20px;
  width: 50px;
  text-align: center;
  color: var(--text-primary);
  transition: transform 0.3s ease;
}

.notification-item:hover .notification-icon {
  transform: rotate(15deg);
}

.notification-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.notification-message {
  margin: 0;
  font-size: 1.1rem;
  color: var(--text-color);
  font-weight: 500;
  line-height: 1.4;
}

.notification-time {
  margin: 0;
  font-size: 0.9rem;
  color: var(--text-secondary);
  font-style: italic;
}

.unread-dot {
  width: 12px;
  height: 12px;
  background-color: var(--primary-color);
  border-radius: 50%;
  margin-left: 15px;
  box-shadow: 0 0 5px var(--primary-color);
  animation: pulse 1.5s infinite;
}

.load-more {
  margin-top: 25px;
  padding: 12px 25px;
  background-color: var(--secondary-bg);
  color: var(--text-color);
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1rem;
  font-weight: 500;
  transition: all 0.3s ease;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.load-more:hover {
  background-color: var(--menu-item-hover);
  transform: translateY(-2px);
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);
}

.load-more:active {
  transform: translateY(0);
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.load-more:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

/* Темная тема */
.dark-theme .notifications-page {
  background-color: var(--dropdown-bg);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.4);
}

.dark-theme .notification-item.unread {
  background-color: var(--dark-unread-bg);
  border-color: var(--primary-color-dark);
}

.dark-theme .notification-icon {
  color: var(--text-primary-dark);
}

.dark-theme .notification-message {
  color: var(--text-color-dark);
}

.dark-theme .notification-time {
  color: var(--text-secondary-dark);
}
</style>

<style>
:root {
  --unread-bg: rgba(0, 66, 129, 0.1);
  --card-bg: #ffffff;
  --card-hover: #f9f9f9;
  --text-secondary: #666666;
  --border-color-light: #e0e0e0;
  --text-primary: #333333;
  --primary-color: #007bff;
  --primary-hover: #0056b3;
  --secondary-bg: #e9ecef;
  --menu-item-hover: #dfe4ea;
}

.dark-theme {
  --unread-bg: rgba(74, 156, 255, 0.1);
  --dark-unread-bg: rgba(74, 156, 255, 0.2);
  --card-bg: #2d2d2d;
  --card-hover: #3a3a3a;
  --text-secondary: #bbbbbb;
  --border-color-light: #444444;
  --text-primary: #e0e0e0;
  --text-primary-dark: #ffffff;
  --text-color-dark: #ffffff;
  --text-secondary-dark: #cccccc;
  --primary-color: #4dabf7;
  --primary-color-dark: #339af0;
  --primary-hover: #339af0;
  --secondary-bg: #3c3f4a;
  --menu-item-hover: #4a4e5a;
  --dropdown-bg: #252830;
}
</style>