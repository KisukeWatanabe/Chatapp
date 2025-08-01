chat.vue
<script setup>
import { inject, ref, reactive, onMounted, onBeforeUnmount } from "vue"
import { useRouter } from "vue-router"
import socketManager from '../socketManager.js'

const userName = inject("userName")
// 遷移用ルータ
const router = useRouter()
// #endregion


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


// #endregion

// #region socket event handler
// サーバから受信した入室メッセージ画面上に表示する
const onReceiveEnter = (name) => {
  const log = name + "さんが入室しました"
  logList.push(log)
}
const onReceiveExit = (name) => {
  const log = name + "さんが退室しました"
  logList.push(log)
}

const registerSocketEvent = () => {
  // ソケットを一回切っておく
  socket.off("publishEvent")
  socket.off("enterEvent")
  socket.off("exitEvent")

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
      <h1 class="title">TASUKU HAA-KUNのチャットルーム</h1>
      <p class="login-user">ログインユーザー：{{ userName }}さん</p>

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
@import url('https://fonts.googleapis.com/css2?family=M+PLUS+Rounded+1c&display=swap');

* {
 font-family: 'M PLUS Rounded 1c', 'Arial', 'Meiryo', sans-serif;
  box-sizing: border-box;
}

/* 全体レイアウト */
.layout {
  display: flex;
  flex-direction: row;
  height: 100vh;
  overflow: hidden;
  background-color: #fffaf5;
}

/* メインチャットエリア */
.chatroom-container {
  flex: 1;
  padding: 20px;
  padding-bottom: 140px;
}

.title {
  font-size: 28px;
  font-weight: bold;
  margin-bottom: 8px;
  color: #333;
  background-color: transparent;
}

.login-user {
  font-size: 14px;
  color: #444;
}

/* チャットメッセージ */
.chat-list {
  display: flex;
  flex-direction: column;
  height: 400px;
  overflow-y: auto;
  margin-top: 20px;
  padding: 12px;
  background-color: #ffffffcc;
  border-radius: 12px;
  box-shadow: 0 0 4px rgba(0,0,0,0.1);
}

.item {
  margin-bottom: 10px;
  padding-bottom: 6px;
  border-bottom: 1px solid #ccc;
  word-break: break-word;
}

.meta {
  font-size: 0.75rem;
  color: gray;
  margin-bottom: 4px;
}

/* フッター入力欄 */
.chat-footer {
  position: fixed;
  bottom: 0;
  left: 0;
  width: calc(100% - 310px); /* サイドバーの幅分を差し引く */
  background: #fefefe;
  padding: 12px 20px;
  border-top: 1px solid #ccc;
  box-shadow: 0 -2px 5px rgba(0, 0, 0, 0.05);
}

.textarea {
  width: 100%;
  border: 1px solid #ccc;
  border-radius: 20px;
  padding: 10px 16px;
  font-size: 14px;
  resize: none;
  outline: none;
  background-color: #fff;
}

.button-area {
  margin-top: 10px;
  display: flex;
  justify-content: space-between; 
  align-items: center;
}

.left-buttons {
  display: flex;
  gap: 10px;
}

.right-button {
  display: flex;
}

/* ボタン共通 */
.button-normal {
  padding: 8px 18px;
  border: none;
  border-radius: 20px;
  font-size: 14px;
  cursor: pointer;
  background-color: #ffa500;
  color: white;
  transition: background-color 0.3s ease;
}

.button-normal:hover {
  background-color: #ff8800;
}

/* 退室ボタン */
.button-exit {
  background-color: #aaa;
  color: #fff;
}

.button-exit:hover {
  background-color: #888;
}

.link {
  text-decoration: none;
}

/* サイドバー */
.sidebar {
  width: 280px;
  background: #f0f0f0;
  border-left: 1px solid #ccc;
  padding: 20px 16px;
  overflow-y: auto;
}

.sidebar h2 {
  font-size: 18px;
  margin-bottom: 16px;
  color: #333;
}

/* ログリスト */
.log-list {
  height: calc(100vh - 100px);
  overflow-y: auto;
  list-style: none;
  padding: 0;
  font-size: 13px;
  margin: 0;
}

.log-list li {
  margin-bottom: 10px;
  border-bottom: 1px dashed #ccc;
  padding-bottom: 6px;
  color: #ffa500;
}

</style>

