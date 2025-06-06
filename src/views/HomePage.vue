<template>
  <div class="page-container">
    <AppNavbar />

    <div class="search-container">
      <!-- Поле "Откуда" -->
      <div class="input-container">
        <input
          type="text"
          placeholder="Откуда"
          v-model="fromLocation"
          @input="onFromLocationChange"
          @focus="showFromSuggestions = true"
          @blur="hideFromSuggestions"
        />
        <ul v-if="fromLocation && fromAddressSuggestions.length && showFromSuggestions" class="suggestions-list">
          <li
            v-for="(suggestion, index) in fromAddressSuggestions"
            :key="index"
            @mousedown="selectLocation('from', suggestion)"
          >
            {{ suggestion }}
          </li>
        </ul>
        <span v-if="!isFromAddressValid && fromLocation" class="error-message">Выберите адрес из списка</span>
        <div v-if="loadingFromSuggestions" class="suggestions-loading">Загрузка...</div>
      </div>

      <!-- Поле "Куда" -->
      <div class="input-container">
        <input
          type="text"
          placeholder="Куда"
          v-model="toLocation"
          @input="onToLocationChange"
          @focus="showToSuggestions = true"
          @blur="hideToSuggestions"
        />
        <ul v-if="toLocation && toAddressSuggestions.length && showToSuggestions" class="suggestions-list">
          <li
            v-for="(suggestion, index) in toAddressSuggestions"
            :key="index"
            @mousedown="selectLocation('to', suggestion)"
          >
            {{ suggestion }}
          </li>
        </ul>
        <span v-if="!isToAddressValid && toLocation" class="error-message">Выберите адрес из списка</span>
        <div v-if="loadingToSuggestions" class="suggestions-loading">Загрузка...</div>
      </div>

      <!-- Поле "Дата" -->
      <div class="input-container">
        <label for="date-input" class="input-label" :class="{ 'active': drivingDate }">Дата</label>
        <input 
          id="date-input"
          type="date" 
          v-model="drivingDate" 
          :min="minDate"
          class="date-input"
          @change="validateDate"
          @focus="onDateFocus"
          @blur="onDateBlur"
        />
        <span v-if="dateError" class="error-message">{{ dateError }}</span>
      </div>

      <!-- Счётчик пассажиров (+ 1 -) -->
      <div class="passenger-control">
        <button 
          class="passenger-btn" 
          @click="decrementPassenger" 
          :disabled="passengerCount <= 1"
          aria-label="Уменьшить количество пассажиров"
        >−</button>
        <span class="passenger-count">{{ passengerCount }}</span>
        <button 
          class="passenger-btn" 
          @click="incrementPassenger" 
          :disabled="passengerCount >= 20"
          aria-label="Увеличить количество пассажиров"
        >+</button>
      </div>

      <!-- Кнопка "Поиск" -->
      <button 
        class="search-btn" 
        @click="goToSearch"
        :disabled="isSearching"
      >
        <span v-if="!isSearching">Поиск</span>
        <div v-else class="search-spinner">
          <div class="spinner"></div>
          <span>Поиск маршрутов...</span>
        </div>
      </button>
    </div>

    <!-- Уведомление -->
    <transition name="fade">
      <div v-if="notificationVisible" :class="['notification', notificationType]">
        <span>{{ notificationMessage }}</span>
        <button class="close-notification" @click="notificationVisible = false">×</button>
      </div>
    </transition>

    <!-- Информационный раздел -->
    <div class="info-section">
      <div class="info-block">
        <img src="/pngwing.com (7).png" alt="Ваша поездка по низкой цене" />
        <h3>Ваша поездка по низкой цене</h3>
        <p>
          Неважно, куда вы направляетесь, вы сможете найти подходящий маршрут среди множества вариантов и добраться до места по самой выгодной цене.
        </p>
      </div>
      <div class="info-block">
        <img src="/pngwing.com (8).png" alt="Доверяйте своим попутчикам" />
        <h3>Доверяйте своим попутчикам</h3>
        <p>
          С UniGo вы путешествуете с проверенными попутчиками.
          Все участники приложения — это проверенные представители организации,
          что гарантирует вам безопасность и комфорт во время поездки.
        </p>
      </div>
      <div class="info-block">
        <img src="/pngwing.com (9).png" alt="В дорогу за пару кликов!" />
        <h3>В дорогу за пару кликов!</h3>
        <p>
          С UniGo вы сэкономите время и начнёте поездку за считанные минуты!
        </p>
      </div>
    </div>

    <!-- Подвал с контактной информацией и копирайтом -->
    <div class="footer">
      <div class="contact-info">
        <p>Контактная информация: <a href="mailto:info@unigo.com">info@unigo.com</a></p>
        <p><a href="/privacy-policy">Политика конфиденциальности</a></p>
      </div>
      <div class="footer-bottom">
        <p>© 2025 UniGO. Все права защищены.</p>
        <p>Версия 1.0.0</p>
      </div>
    </div>
  </div>
</template>

<script>
import AppNavbar from "@/components/AppNavbar.vue";
import Cookies from 'js-cookie';

const API_TOKEN = "72a0f8ef0a9e1bd454cf61b1d040c7b875965ed6";
const SUGGESTIONS_URL =
  "https://suggestions.dadata.ru/suggestions/api/4_1/rs/suggest/address";

export default {
  components: {
    AppNavbar,
  },
  data() {
    const today = new Date();
    const minDate = today.toISOString().split('T')[0];
    
    return {
      fromLocation: "",
      toLocation: "",
      drivingDate: "",
      passengerCount: 1,
      fromAddressSuggestions: [],
      toAddressSuggestions: [],
      notificationVisible: false,
      notificationMessage: "",
      notificationType: "",
      isFromAddressValid: true,
      isToAddressValid: true,
      dateError: "",
      minDate: minDate,
      showFromSuggestions: false,
      showToSuggestions: false,
      loadingFromSuggestions: false,
      loadingToSuggestions: false,
      isSearching: false,
      debounceTimer: null,
      isDatePlaceholder: true
    };
  },
  computed: {
    isFormValid() {
      return this.fromLocation && 
             this.toLocation && 
             this.drivingDate && 
             this.isFromAddressValid && 
             this.isToAddressValid && 
             !this.dateError;
    },
  },
  methods: {
    incrementPassenger() {
      if (this.passengerCount < 20) {
        this.passengerCount++;
        this.showNotification(`Количество пассажиров: ${this.passengerCount}`, "info", 1500);
      } else {
        this.showNotification(
          "Максимальное количество пассажиров - 20",
          "error"
        );
      }
    },
    decrementPassenger() {
      if (this.passengerCount > 1) {
        this.passengerCount--;
        this.showNotification(`Количество пассажиров: ${this.passengerCount}`, "info", 1500);
      }
    },

    async goToSearch() {
      if (!this.isFormValid) {
        let message = "Пожалуйста, укажите все данные для поиска:";
        if (!this.fromLocation) message += "\n- Откуда";
        if (!this.toLocation) message += "\n- Куда";
        if (!this.drivingDate) message += "\n- Дата";
        if (!this.isFromAddressValid) message += "\n- Выберите корректный адрес отправления";
        if (!this.isToAddressValid) message += "\n- Выберите корректный адрес назначения";
        if (this.dateError) message += `\n- ${this.dateError}`;
        
        this.showNotification(message, "error");
        return;
      }

      this.isSearching = true;
      this.showNotification("Ищем лучшие маршруты для вас...", "info");

      try {
        await new Promise(resolve => setTimeout(resolve, 1500));
        
        const searchParams = {
          from: this.fromLocation,
          to: this.toLocation,
          date: this.drivingDate,
          passengers: this.passengerCount,
        };
        Cookies.set("searchParams", JSON.stringify(searchParams), { expires: 1 });

        this.showNotification("Маршруты найдены! Перенаправляем...", "success", 1000);
        await new Promise(resolve => setTimeout(resolve, 1000));
        
        this.$router.push({ name: "search-results" });
      } catch (error) {
        this.showNotification("Ошибка при поиске маршрутов. Пожалуйста, попробуйте позже.", "error");
        console.error("Search error:", error);
      } finally {
        this.isSearching = false;
      }
    },

    validateDate() {
      const selectedDate = new Date(this.drivingDate);
      const today = new Date();
      today.setHours(0, 0, 0, 0);
      
      if (selectedDate < today) {
        this.dateError = "Нельзя выбрать прошедшую дату";
        this.isDatePlaceholder = false;
        return false;
      }
      
      this.dateError = "";
      this.isDatePlaceholder = false;
      return true;
    },

    showNotification(message, type, duration = 3000) {
      if (!message || !type) return;
      
      this.notificationVisible = false;
      
      setTimeout(() => {
        this.notificationMessage = message;
        this.notificationType = type;
        this.notificationVisible = true;
        
        if (duration > 0) {
          setTimeout(() => {
            this.notificationVisible = false;
          }, duration);
        }
      }, 100);
    },

    async getSuggestions(query, type) {
      if (!query || query.length < 2) {
        if (type === 'from') {
          this.fromAddressSuggestions = [];
          this.loadingFromSuggestions = false;
        } else {
          this.toAddressSuggestions = [];
          this.loadingToSuggestions = false;
        }
        return [];
      }
      
      try {
        if (type === 'from') {
          this.loadingFromSuggestions = true;
        } else {
          this.loadingToSuggestions = true;
        }
        
        await new Promise(resolve => setTimeout(resolve, 300));
        
        const response = await fetch(SUGGESTIONS_URL, {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            Authorization: `Token ${API_TOKEN}`,
          },
          body: JSON.stringify({ query }),
        });
        
        if (!response.ok) {
          throw new Error("Ошибка при получении подсказок");
        }
        
        const data = await response.json();
        return data.suggestions.map((suggestion) => suggestion.value);
      } catch (error) {
        console.error("Ошибка при запросе подсказок:", error);
        this.showNotification("Не удалось загрузить подсказки. Проверьте соединение.", "error");
        return [];
      } finally {
        if (type === 'from') {
          this.loadingFromSuggestions = false;
        } else {
          this.loadingToSuggestions = false;
        }
      }
    },

    async onFromLocationChange() {
      clearTimeout(this.debounceTimer);
      this.debounceTimer = setTimeout(async () => {
        this.fromAddressSuggestions = await this.getSuggestions(this.fromLocation, 'from');
        this.isFromAddressValid = !this.fromLocation || this.fromAddressSuggestions.includes(this.fromLocation);
      }, 500);
    },

    async onToLocationChange() {
      clearTimeout(this.debounceTimer);
      this.debounceTimer = setTimeout(async () => {
        this.toAddressSuggestions = await this.getSuggestions(this.toLocation, 'to');
        this.isToAddressValid = !this.toLocation || this.toAddressSuggestions.includes(this.toLocation);
      }, 500);
    },

    selectLocation(field, suggestion) {
      if (field === "from") {
        this.fromLocation = suggestion;
        this.isFromAddressValid = true;
        this.fromAddressSuggestions = [];
      } else if (field === "to") {
        this.toLocation = suggestion;
        this.isToAddressValid = true;
        this.toAddressSuggestions = [];
      }
      
      this.showNotification(`Выбрано: ${suggestion}`, "success", 1500);
    },
    
    hideFromSuggestions() {
      setTimeout(() => {
        this.showFromSuggestions = false;
      }, 200);
    },
    
    hideToSuggestions() {
      setTimeout(() => {
        this.showToSuggestions = false;
      }, 200);
    },

    onDateFocus() {
      this.isDatePlaceholder = false;
    },

    onDateBlur() {
      if (!this.drivingDate) {
        this.isDatePlaceholder = true;
      }
    }
  },
};
</script>

<style scoped>
/* Основной контейнер для всей страницы */
.page-container {
  background-color: #f5f7fa;
  min-height: 100vh;
  padding-top: 80px;
}
.my-component {
  background-color: var(--card-bg);
  color: var(--text-color);
  border: 1px solid var(--border-color);
}

.my-component:hover {
  background-color: var(--hover-color);
}
/* Уведомления */
.notification {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  padding: 15px 40px 15px 20px;
  border-radius: 8px;
  color: white;
  font-weight: 500;
  display: flex;
  justify-content: center;
  align-items: center;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  max-width: 90%;
  z-index: 10000;
}

.notification.success {
  background-color: #4CAF50;
}

.notification.error {
  background-color: #F44336;
}

.notification.info {
  background-color: #2196F3;
}

.close-notification {
  position: absolute;
  right: 10px;
  top: 50%;
  transform: translateY(-50%);
  background: none;
  border: none;
  color: white;
  font-size: 20px;
  cursor: pointer;
  padding: 0 5px;
  opacity: 0.8;
}

.close-notification:hover {
  opacity: 1;
}

.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s, transform 0.3s;
}
.fade-enter, .fade-leave-to {
  opacity: 0;
  transform: translate(-50%, -20px);
}

/* Основной контейнер для поиска */
.search-container {
  display: flex;
  align-items: center;
  gap: 15px;
  padding: 25px;
  background-color: rgba(0, 66, 129, 0.1);
  border-radius: 12px;
  max-width: 1200px;
  margin: 20px auto 0 auto;
  box-shadow: 0 4px 20px rgba(0, 66, 129, 0.1);
  flex-wrap: wrap;
}

/* Контейнеры для полей ввода */
.input-container {
  position: relative;
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 200px;
}

.input-container input {
  width: 100%;
  height: 50px;
  padding: 10px 15px;
  border: 1px solid #ddd;
  border-radius: 8px;
  font-size: 16px;
  box-sizing: border-box;
  transition: border-color 0.3s;
  background-color: white;
}

.input-container input:focus {
  border-color: rgba(0, 66, 129, 0.8);
  outline: none;
  box-shadow: 0 0 0 2px rgba(0, 66, 129, 0.1);
}

.error-message {
  color: #F44336;
  font-size: 13px;
  margin-top: 5px;
  display: block;
}

/* Стили для метки даты */
.input-label {
  position: absolute;
  top: 50%;
  left: 15px;
  transform: translateY(-50%);
  color: #999;
  font-size: 16px;
  pointer-events: none;
  transition: all 0.2s ease;
  opacity: 1;
}

.input-label.active {
  top: -10px;
  left: 10px;
  font-size: 12px;
  color: rgba(0, 66, 129, 0.8);
  background-color: white;
  padding: 0 5px;
  opacity: 1;
}

/* Стили для поля даты */
.date-input {
  width: 100%;
  height: 50px;
  padding: 10px 15px;
  border: 1px solid #ddd;
  border-radius: 8px;
  font-size: 16px;
  box-sizing: border-box;
  background-color: white;
  color: #333;
  transition: border-color 0.3s;
  -webkit-appearance: none;
  -moz-appearance: none;
  appearance: none;
}

.date-input:focus {
  border-color: rgba(0, 66, 129, 0.8);
  outline: none;
  box-shadow: 0 0 0 2px rgba(0, 66, 129, 0.1);
}

.date-input:invalid:not(:focus) + .input-label {
  opacity: 1;
}

.date-input:focus + .input-label,
.date-input:not(:invalid) + .input-label {
  opacity: 0;
}

/* Стили для счётчика пассажиров */
.passenger-control {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background-color: white;
  border-radius: 8px;
  padding: 0 15px;
  height: 50px;
  border: 1px solid #ddd;
  min-width: 180px;
  position: relative;
}

.passenger-btn {
  width: 30px;
  height: 30px;
  border-radius: 50%;
  border: 1px solid rgba(0, 66, 129, 0.3);
  background-color: transparent;
  color: rgba(0, 66, 129, 0.8);
  font-size: 16px;
  font-weight: bold;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s ease;
}

.passenger-btn:hover:not(:disabled) {
  background-color: rgba(0, 66, 129, 0.1);
  border-color: rgba(0, 66, 129, 0.5);
}

.passenger-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.passenger-count {
  min-width: 30px;
  text-align: center;
  font-size: 16px;
  color: rgba(0, 66, 129, 0.8);
  font-weight: 500;
}

/* Стили для кнопки поиска */
.search-btn {
  height: 50px;
  padding: 0 25px;
  background-color: rgba(0, 66, 129, 1);
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  white-space: nowrap;
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 120px;
}

.search-btn:hover:not(:disabled) {
  background-color: rgba(0, 66, 129, 0.9);
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(0, 66, 129, 0.2);
}

.search-btn:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}

.search-spinner {
  display: flex;
  align-items: center;
  gap: 10px;
}

.spinner {
  width: 18px;
  height: 18px;
  border: 3px solid rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  border-top-color: white;
  animation: spin 1s ease-in-out infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

/* Выпадающие подсказки */
.suggestions-list {
  position: absolute;
  top: 100%;
  left: 0;
  width: 100%;
  background-color: white;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  list-style: none;
  margin: 5px 0 0;
  padding: 0;
  z-index: 1000;
  border-radius: 8px;
  max-height: 300px;
  overflow-y: auto;
}

.suggestions-list li {
  padding: 12px 15px;
  cursor: pointer;
  border-bottom: 1px solid rgba(0, 0, 0, 0.05);
  transition: background-color 0.2s;
}

.suggestions-list li:last-child {
  border-bottom: none;
}

.suggestions-list li:hover {
  background-color: rgba(0, 66, 129, 0.05);
}

.suggestions-loading {
  padding: 10px 15px;
  color: #666;
  font-size: 14px;
  font-style: italic;
}

/* Информационный раздел */
.info-section {
  display: flex;
  justify-content: space-between;
  gap: 30px;
  padding: 40px;
  background-color: rgba(0, 66, 129, 0.05);
  border-radius: 12px;
  margin: 40px auto;
  max-width: 1200px;
  flex-wrap: wrap;
}

.info-block {
  flex: 1 1 calc(33% - 20px);
  text-align: center;
  box-sizing: border-box;
  padding: 30px 25px;
  background-color: white;
  border-radius: 12px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  min-height: 280px;
  transition: transform 0.3s, box-shadow 0.3s;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
}

.info-block:hover {
  transform: translateY(-5px);
  box-shadow: 0 8px 20px rgba(0, 66, 129, 0.1);
}

.info-block img {
  width: 80px;
  height: auto;
  margin-bottom: 20px;
  opacity: 0.9;
}

.info-block h3 {
  font-size: 18px;
  margin-bottom: 15px;
  color: rgba(0, 66, 129, 1);
}

.info-block p {
  font-size: 14px;
  color: #555;
  line-height: 1.6;
}

/* Подвал */
.footer {
  background-color: rgba(0, 66, 129, 0.05);
  padding: 30px 0 20px;
  text-align: center;
  font-size: 14px;
  margin-top: 60px;
}

.contact-info {
  margin-bottom: 15px;
}

.contact-info p {
  margin: 8px 0;
}

.contact-info a {
  color: rgba(0, 66, 129, 0.8);
  text-decoration: none;
  transition: color 0.2s;
}

.contact-info a:hover {
  color: rgba(0, 66, 129, 1);
  text-decoration: underline;
}

.footer-bottom {
  margin-top: 20px;
  font-size: 12px;
  color: #666;
  display: flex;
  justify-content: center;
  gap: 20px;
}

/* Плавные переходы */
input, button, .info-block, .notification, .input-label {
  transition: all 0.3s ease;
}

/* Мобильная адаптация */
@media (max-width: 1024px) {
  .page-container {
    padding-top: 80px;
  }

  .search-container {
    flex-wrap: wrap;
    margin-top: 20px;
    gap: 15px;
    padding: 20px;
  }
  
  .input-container,
  .passenger-control {
    flex: 1 1 calc(50% - 10px);
    min-width: 200px;
  }
  
  .search-btn {
    flex: 1 1 100%;
  }
  
  .info-block {
    flex: 1 1 calc(50% - 15px);
  }

  .input-label {
    font-size: 15px;
  }
  .input-label.active {
    font-size: 11px;
    top: -12px;
  }
}

@media (max-width: 768px) {
  .page-container {
    padding-top: 60px;
  }

  .search-container {
    margin-top: 15px;
    padding: 15px;
    border-radius: 8px;
    width: calc(100% - 30px);
    margin-left: auto;
    margin-right: auto;
  }

  .input-container,
  .passenger-control,
  .search-btn {
    flex: 1 1 100%;
  }
  
  .passenger-control {
    justify-content: center;
    gap: 20px;
  }
  
  .info-section {
    padding: 15px;
    margin: 20px auto;
    gap: 15px;
    width: calc(100% - 30px);
  }
  
  .info-block {
    flex: 1 1 100%;
    min-height: auto;
    padding: 20px 15px;
  }
  
  .footer {
    padding: 20px 15px;
    margin-top: 30px;
  }
  
  .footer-bottom {
    flex-direction: column;
    gap: 5px;
  }

  .date-input {
    font-size: 15px;
    height: 45px;
    padding: 10px 15px;
  }
  .input-label {
    font-size: 15px;
  }
  .input-label.active {
    font-size: 11px;
    top: -12px;
  }
}

@media (max-width: 480px) {
  .page-container {
    padding-top: 60px;
  }

  .search-container {
    margin-top: 10px;
    padding: 12px;
    width: calc(100% - 24px);
  }
  
  .input-container input,
  .passenger-control,
  .search-btn {
    height: 44px;
    font-size: 14px;
  }
  
  .passenger-btn {
    width: 26px;
    height: 26px;
    font-size: 14px;
  }
  
  .info-block {
    padding: 18px 12px;
  }
  
  .info-block h3 {
    font-size: 16px;
    margin-bottom: 12px;
  }
  
  .info-block p {
    font-size: 13px;
    line-height: 1.5;
  }
  
  .info-block img {
    width: 70px;
    margin-bottom: 15px;
  }
  
  .notification {
    padding: 10px 30px 10px 15px;
    font-size: 13px;
    max-width: 96%;
    top: 15px;
  }
  
  .footer {
    padding: 15px 10px;
    margin-top: 25px;
    font-size: 13px;
  }
  
  .footer-bottom {
    font-size: 11px;
  }

  .date-input {
    font-size: 14px;
    height: 44px;
    padding: 10px 12px;
  }
  .input-label {
    font-size: 14px;
  }
  .input-label.active {
    font-size: 10px;
    top: -10px;
  }
}

@media (max-width: 360px) {
  .search-container {
    margin-top: 5px;
    padding: 10px;
    width: calc(100% - 20px);
  }
  
  .input-container input {
    padding: 10px 10px;
    font-size: 13px;
  }
  
  .passenger-control {
    height: 42px;
  }
  
  .search-btn {
    font-size: 14px;
    padding: 0 15px;
  }
  
  .info-block {
    padding: 15px 10px;
  }

  .date-input {
    font-size: 13px;
    height: 42px;
    padding: 10px 10px;
  }
  .input-label {
    font-size: 13px;
  }
  .input-label.active {
    font-size: 10px;
    top: -10px;
  }
}

/* Анимации */
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(-10px); }
  to { opacity: 1; transform: translateY(0); }
}
</style>