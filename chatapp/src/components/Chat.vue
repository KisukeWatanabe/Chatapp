<script setup>
import { inject, ref, reactive, onMounted } from "vue"
import { useRouter } from "vue-router"
import socketManager from '../socketManager.js'

const userName = inject("userName")
// 遷移用ルータ
const router = useRouter()
// #endregion

if (userName.value === "") {
  userName.value = "ゲスト"
}

const socket = socketManager.getInstance()

const chatContent = ref("")
const chatList = reactive([])  // [{ name, time, message }]
const logList = reactive([])   // ["ゲストさんが入室", "山田さんが退出"...]

const chatListRef = ref(null)  // チャット欄のDOM参照

onMounted(() => {
  registerSocketEvent()
})

const onPublish = () => {
  const now = new Date()
  const formatter = new Intl.DateTimeFormat('ja-JP', {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit'
  })
  const time = formatter.format(now)

  const payload = {
    name: userName.value,
    time: time,
    message: chatContent.value
  }

  socket.emit("publishEvent", payload)
  chatContent.value = ""
}

const onExit = () => {
 socket.emit("exitEvent", userName.value)
  const log = userName.value + "さんが退室"
  logList.push(log)
}

const onReceivePublish = (data) => {
  chatList.push(data)

  nextTick(() => {
    if (chatListRef.value) {
      chatListRef.value.scrollTop = chatListRef.value.scrollHeight
    }
  })
}
const onChange = () => {
  router.push({ name: "task" })
}

// メモを画面上に表示する
const onMemo = () => {
  // メモの内容を表示
  chatList.unshift(userName.value +"さんのメモ：" + chatContent.value)
  // 入力欄を初期化
  chatContent.value = ""
}
// #endregion

// #region socket event handler
// サーバから受信した入室メッセージ画面上に表示する
const onReceiveEnter = (data) => {
  chatList.push(userName.value + "さんが入室しました")
}

const onReceiveExit = (name) => {
  const log = name + "さんが退室"
  logList.push(log)
}

const registerSocketEvent = () => {
  socket.on("publishEvent", (data) => {
    onReceivePublish(data)
  })

  socket.on("enterEvent", (name) => {
    onReceiveEnter(name)
  })

  socket.on("exitEvent", (name) => {
    onReceiveExit(name)
  })
}
</script>

<template>
  <div class="layout">
    <!-- チャットルームメイン -->
    <div class="chatroom-container">
      <h1 class="title">はあくん チャットルーム</h1>
      <p class="login-user">ログインユーザ：{{ userName }}さん</p>

      <div class="chat-list" ref="chatListRef">
        <ul>
          <li class="item" v-for="(chat, i) in chatList" :key="i">
            <small class="meta">{{ chat.name }}（{{ chat.time }}）</small><br />
            {{ chat.message }}
          </li>
        </ul>
      </div>

      <div class="chat-footer">
        <textarea
          v-model="chatContent"
          placeholder="メッセージを入力してください"
          rows="2"
          class="textarea"
        ></textarea>
        <div class="button-area">
          <div class="left-buttons">
            <button class="button-normal" @click="onPublish">送信</button>
          </div>
          <div class="right-button">
            <router-link to="/task/" class="task">
              <button type="button" class="button-normal" @click="onChange">タスクへ移動</button>
            </router-link>
            <router-link to="/" class="link">
              <button type="button" class="button-normal button-exit" @click="onExit">退室する</button>
            </router-link>
          </div>
        </div>
      </div>
    </div>

    <!-- サイドバー（入退出ログ） -->
    <div class="sidebar">
      <h2>入退出ログ</h2>
      <ul class="log-list">
        <li v-for="(log, i) in logList" :key="i">{{ log }}</li>
      </ul>
    </div>
  </div>
</template>

<style scoped>
/* 全体レイアウト */
.layout {
  display: flex;
  flex-direction: row;
  height: 100vh;
  overflow: hidden;
}

/* メインチャットエリア */
.chatroom-container {
  flex: 1;
  padding: 16px;
  padding-bottom: 120px; 
}

.title {
  font-size: 24px;
  font-weight: bold;
}

.login-user {
  margin-top: 8px;
  font-size: 14px;
  color: #333;
}

/* チャットメッセージ */
.chat-list {
  display: flex;
  flex-direction: column;
  height: 400px;
  overflow-y: auto;
  margin-top: 16px;
  padding-right: 8px;
}

.item {
  margin-bottom: 12px;
  padding: 4px;
  border-bottom: 1px solid #ccc;
  word-break: break-word;
}

.meta {
  font-size: 0.75rem;
  color: gray;
}

/* フッター入力欄 */
.chat-footer {
  position: fixed;
  bottom: 0;
  left: 0;
  width: calc(100% - 280px); /* サイドバーの幅分を差し引く */
  background: #f8f8f8;
  padding: 12px 16px;
  border-top: 1px solid #ccc;
}

.textarea {
  width: 100%;
  max-width: 100%;
  border: 1px solid #000;
  resize: none;
  padding: 6px;
  font-size: 14px;
}

.button-normal {
  padding: 6px 12px;
  font-size: 14px;
  cursor: pointer;
}
.button-area {
  margin-top: 8px;
  display: flex;
  justify-content: space-between; 
  align-items: center;
}

.left-buttons {
  display: flex;
  gap: 8px;
}

.right-button {
  display: flex;
}

.button-exit {
  color: #000;
}

.link {
  text-decoration: none;
}

/* サイドバー */
.sidebar {
  width: 250px;
  background: #f0f0f0;
  border-left: 1px solid #ccc;
  padding: 16px;
  overflow-y: auto;
}

.sidebar h2 {
  font-size: 18px;
  margin-bottom: 12px;
}

.log-list {
  height: 400px;
  overflow-y: auto;
  list-style: none;
  padding: 0;
  font-size: 14px;
  margin: 0;
}

.log-list li {
  margin-bottom: 6px;
  border-bottom: 1px dashed #ccc;
  padding-bottom: 4px;
}
</style>
