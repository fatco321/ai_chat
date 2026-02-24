<template>
  <div class="chat-window">
    <section ref="messagesEl" class="chat-messages">
      <div v-if="!history.length && !loading" class="empty-state">
        Начните диалог.
      </div>

      <div
        v-for="(msg, index) in history"
        :key="index"
        class="message-row"
        :class="msg.role"
      >
        <div class="avatar">
          <span v-if="msg.role === 'user'">Вы</span>
          <img v-else :src="faviconUrl" alt="AI" class="avatar-img" />
        </div>
        <div class="bubble">
          <p v-if="msg.role === 'user'" class="bubble-text">{{ msg.text }}</p>

          <p v-else class="bubble-text" v-html="msg.text"></p>
        </div>
      </div>

      <div v-if="loading" class="message-row model">
        <div class="avatar">
          <img :src="faviconUrl" alt="AI" class="avatar-img" />
        </div>
        <div class="bubble typing">
          <span class="dot"></span>
          <span class="dot"></span>
          <span class="dot"></span>
        </div>
      </div>
    </section>

    <footer class="composer">
      <div v-if="error" class="error-banner">
        {{ error }}
      </div>
      <div class="composer-inner">
        <textarea
          ref="inputEl"
          v-model="userInput"
          placeholder="Напиши сообщение..."
          :disabled="loading"
          rows="1"
          @input="autoGrow"
          @keydown.enter.exact.prevent="sendMessage"
        ></textarea>
        <button
          class="send-btn"
          @click="sendMessage"
          :disabled="loading || !userInput.trim()"
        >
          Отправить
        </button>
      </div>
      <div class="composer-hint">
        Enter — отправить • Shift+Enter — новая строка
      </div>
    </footer>
  </div>
</template>

<script setup>
import { nextTick, onMounted, ref, watch } from "vue";
import faviconUrl from "../assets/favicon.png";

const WORKER_URL = import.meta.env.VITE_WORKER_URL || "";

const userInput = ref("");
const history = ref([]);
const loading = ref(false);
const error = ref(null);
const messagesEl = ref(null);
const inputEl = ref(null);

const scrollToBottom = async () => {
  await nextTick();
  if (!messagesEl.value) return;
  messagesEl.value.scrollTop = messagesEl.value.scrollHeight;
};

const autoGrow = () => {
  if (!inputEl.value) return;
  inputEl.value.style.height = "auto";
  const next = Math.min(inputEl.value.scrollHeight, 160);
  inputEl.value.style.height = `${next}px`;
};

const sendMessage = async () => {
  const text = userInput.value.trim();
  if (!text) return;
  if (!WORKER_URL) {
    error.value = "Не задан VITE_WORKER_URL в .env";
    return;
  }

  history.value.push({ role: "user", text });
  userInput.value = "";
  autoGrow();
  loading.value = true;
  error.value = null;

  try {
    const response = await fetch(WORKER_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ prompt: text }),
    });

    if (!response.ok) {
      throw new Error(`Ошибка сервера: ${response.status}`);
    }

    const data = await response.json();
    const aiText = data.candidates?.[0]?.content?.parts?.[0]?.text;

    if (aiText) {
      history.value.push({ role: "model", text: aiText });
    } else {
      history.value.push({
        role: "model",
        text: "Ошибка: пустой ответ от нейросети.",
      });
      console.error("Полный ответ:", data);
    }
  } catch (err) {
    console.error(err);
    error.value = "Ошибка соединения. Проверьте консоль.";
    history.value.push({ role: "model", text: "⚠️ Ошибка связи с сервером." });
  } finally {
    loading.value = false;
  }
};

watch(history, scrollToBottom, { deep: true });
watch(loading, scrollToBottom);

onMounted(() => {
  autoGrow();
});
</script>

<style scoped>
.chat-window {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  background: #f7f7f8;
  color: #0f172a;
}

.chat-messages {
  flex: 1;
  padding: 24px 28px 100px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 18px;
}

.empty-state {
  margin: 120px auto 0;
  color: #6b7280;
  font-size: 14px;
}

.message-row {
  display: grid;
  grid-template-columns: 40px 1fr;
  gap: 14px;
  align-items: flex-start;
}

.message-row.user .avatar {
  background: #111827;
}

.message-row.model .avatar {
  background: #0f766e;
}

.avatar {
  width: 36px;
  height: 36px;
  border-radius: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  font-weight: 600;
  color: #fff;
  overflow: hidden;
}

.avatar-img {
  width: 22px;
  height: 22px;
  object-fit: contain;
  display: block;
}

.bubble {
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 14px;
  padding: 12px 14px;
}

.message-row.user .bubble {
  background: #f3f4f6;
  border-color: #e5e7eb;
}

.bubble-text {
  font-size: 14px;
  line-height: 1.5;
  white-space: pre-wrap;
}

.typing {
  display: flex;
  gap: 6px;
  width: fit-content;
}

.dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #9ca3af;
  animation: pulse 1s infinite ease-in-out;
}

.dot:nth-child(2) {
  animation-delay: 0.2s;
}

.dot:nth-child(3) {
  animation-delay: 0.4s;
}

@keyframes pulse {
  0%,
  100% {
    transform: translateY(0);
    opacity: 0.4;
  }
  50% {
    transform: translateY(-4px);
    opacity: 1;
  }
}

.composer {
  position: sticky;
  bottom: 0;
  padding: 16px 28px 24px;
  background: linear-gradient(180deg, rgba(247, 247, 248, 0.6), #f7f7f8 60%);
  border-top: 1px solid #e5e7eb;
}

.composer-inner {
  display: flex;
  gap: 12px;
  align-items: flex-end;
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 16px;
  padding: 10px 12px;
}

textarea {
  flex: 1;
  resize: none;
  border: none;
  outline: none;
  background: transparent;
  color: #0f172a;
  font-size: 14px;
  font-family: inherit;
  min-height: 24px;
  max-height: 160px;
}

.send-btn {
  background: #10a37f;
  border: none;
  color: #ffffff;
  font-weight: 600;
  border-radius: 12px;
  padding: 10px 16px;
  cursor: pointer;
}

.send-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.composer-hint {
  margin-top: 8px;
  color: #6b7280;
  font-size: 11px;
  text-align: right;
}

.error-banner {
  margin-bottom: 10px;
  padding: 10px 12px;
  border-radius: 12px;
  background: #fee2e2;
  border: 1px solid #fecaca;
  color: #b91c1c;
  font-size: 12px;
}
</style>
