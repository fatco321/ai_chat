<template>
  <div class="chat-container">
    <div class="messages">
      <div
        v-for="(msg, index) in history"
        :key="index"
        :class="['message', msg.role]"
      >
        <span class="role-label">{{ msg.role === "user" ? "Вы" : "AI" }}:</span>
        {{ msg.text }}
      </div>
      <div v-if="loading" class="message model">
        <em>Печатает...</em>
      </div>
      <div v-if="error" class="error">
        {{ error }}
      </div>
    </div>

    <div class="input-area">
      <input
        v-model="userInput"
        @keyup.enter="sendMessage"
        placeholder="Спросите что-нибудь..."
        :disabled="loading"
      />
      <button @click="sendMessage" :disabled="loading || !userInput.trim()">
        Отправить
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";
import { GoogleGenerativeAI } from "@google/generative-ai";

// Инициализация. Лучше хранить ключ в .env файле
// В Vite переменные среды начинаются с VITE_
const API_KEY = import.meta.env.VITE_GEMINI_API_KEY || "";

const genAI = new GoogleGenerativeAI(API_KEY);
const model = genAI.getGenerativeModel({ model: "gemini-3-flash-preview" });

const userInput = ref("");
const history = ref([]);
const loading = ref(false);
const error = ref(null);

// Создаем сессию чата (хранит контекст разговора)
const chatSession = model.startChat({
  history: [],
  generationConfig: {
    maxOutputTokens: 1000,
  },
});

const sendMessage = async () => {
  const text = userInput.value.trim();
  if (!text) return;

  // 1. Добавляем сообщение пользователя в UI
  history.value.push({ role: "user", text });
  userInput.value = "";
  loading.value = true;
  error.value = null;

  try {
    // 2. Отправляем запрос в Gemini
    const result = await chatSession.sendMessage(text);
    const response = await result.response;
    const answer = response.text();

    // 3. Добавляем ответ модели в UI
    history.value.push({ role: "model", text: answer });
  } catch (err) {
    console.error(err);
    error.value = "Ошибка: Не удалось получить ответ. Проверьте консоль.";
  } finally {
    loading.value = false;
  }
};
</script>

<style scoped>
.chat-container {
  max-width: 600px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}
.message {
  padding: 10px;
  border-radius: 8px;
  background: #f0f0f0;
}
.message.user {
  background: #e3f2fd;
  align-self: flex-end;
}
.message.model {
  background: #f5f5f5;
  align-self: flex-start;
}
.error {
  color: red;
  font-size: 0.9em;
}
.input-area {
  display: flex;
  gap: 10px;
}
input {
  flex-grow: 1;
  padding: 8px;
}
</style>
