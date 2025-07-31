<script setup>
import { inject, ref, reactive, onMounted, computed } from "vue"
import { useRouter } from "vue-router"
import socketManager from '../socketManager.js'
import initSqlJs from 'sql.js'

// #region global state
const userName = inject("userName")
// 遷移用ルータ
const router = useRouter()
// オブジェクトを仮定する
// #endregion

// #region local variable
const socket = socketManager.getInstance()
// #endregion
// #region reactive variable
const taskContent = ref("")
// 初期タスク（投稿で追加）
const tasks = ref([])
//db用
const taskDB = ref([])  // [{ id, taskName, members, progress, status }]

//DB変数
let db = null
// localStorage キー
const STORAGE_KEY = 'test-db-1'

//非同期処理追加
onMounted(async () => {
  const SQL = await initSqlJs({ locateFile: file => `https://sql.js.org/dist/${file}` })
  db = loadDatabase(SQL)
  initSchema()
  loadTasks()
})
//テーブル作成
function initSchema() {
  db.run(`CREATE TABLE IF NOT EXISTS taskDB (id INTEGER PRIMARY KEY AUTOINCREMENT, taskName TEXT, members TEXT, progress INTEGER, status TEXT);`)
  console.log("Database schema initialized.")
}
//タスクをロードする関数
function loadTasks() {
  const result = db.exec('SELECT id, taskName, members, progress, status FROM taskDB')
  taskDB.value = result[0]?.values ?? []
    tasks.value = taskDB.value.map(row => ({
    id: row[0],
    name: row[1],
    members: row[2],
    progress: row[3],
    status: row[4]
  }))
  console.log("Tasks loaded:", tasks.value)
  console.log("TaskDB loaded:", taskDB.value)
}

function addTask(taskName) {
  db.run('INSERT INTO taskDB (taskName, members, progress, status) VALUES (?,?,?,?)', [taskName.value, null,0, "未着手"])
  //taskContent.value = taskName.value
  console.log("AT Task added:", taskName.value)
  loadTasks()
  saveDatabase()
}

function saveDatabase() {
  const data = db.export()
  localStorage.setItem(STORAGE_KEY, JSON.stringify(Array.from(data)))
}

function loadDatabase(SQL) {
  const saved = localStorage.getItem(STORAGE_KEY)
  if (saved) {
    const bytes = new Uint8Array(JSON.parse(saved))
    return new SQL.Database(bytes)
  }
  return new SQL.Database()
}


// #endregion
// #region lifecycle
onMounted(() => {
  registerSocketEvent()
})
// #endregion
// #region browser event handler
// 全体の進捗度合いについて計算する関数
const overallProgress = computed(() => {
  const taskList = tasks.value;
  if (taskList.length === 0) return 0;
  const totalProgress = taskList.reduce((sum, task) => {
    return sum + Math.min(Math.max(task.progress, 0), 100); // 0〜100に制限
  }, 0);
  return Math.round(totalProgress / taskList.length);
});
// ログイン画面まで戻るためのボタン
const onChange = () => {
  router.push({ name: "chat" })
}
// ボタンが押されたらタスクをレンジスライダーで表示する
const onPublish = () => {
  if (taskContent.value.trim() !== "") {
    tasks.value.push({
      id: row[0],
      name: taskContent.value,
      members: null,
      progress: 0,
      status: "未着"
    })
    addTask(taskContent)
    console.log("OP Task added:", taskContent.value)
    taskContent.value = ""
  }
  console.log()
}
// サーバから受信した投稿メッセージを画面上に表示する
const onReceivePublish = (data) => {
  taskContent.unshift(data)
}
// イベント登録をまとめる
const registerSocketEvent = () => {
  // 入室イベントを受け取ったら実行
  socket.on("enterEvent", (data) => {
    onReceiveEnter(data)
  })
  // 退室イベントを受け取ったら実行
  socket.on("exitEvent", (data) => {
    onReceiveExit(data)
  })
  // 投稿イベントを受け取ったら実行
  socket.on("publishEvent", (data) => {
   onReceivePublish(data)
  })
}
// #endregion
</script>
<template>
  <div class="mx-auto my-5 px-4 text-center">
    <h1 class="text-h3 font-weight-medium">タスク一覧</h1>
    <div class="mt-10">
      <p>ログインユーザー：{{ userName }}さん</p>
      <div class="wrapper1">
        <div class="container">
          <p>全体の進捗: {{ overallProgress }}%</p>
          <progress :value="overallProgress" max="100"></progress>
        </div>
      </div>
      <div class="wrapper2">
        <div class="container">
          <textarea v-model="taskContent" variant="outlined" placeholder="タスクを入力してください" rows="4" class="area"></textarea>
            <div class="mt-5">
              <button class="button-normal" @click="onPublish">投稿</button>
            </div>
        </div>
      </div>
      <div class="wrapper3">
        <div class="mt-5" v-if="tasks.length !== 0">
          <ul>
            <li class="item mt-4" v-for="(task, i) in tasks" :key="i">
            <div>{{ task.name }}</div>
              <input
                type="range"
                min="0"
                max="100"
                step="1"
                v-model="task.progress"
              />
              <p>進捗：{{ task.progress }}%</p>
            </li>
          </ul>
        </div>

         <div class="mt-5" v-if="taskDB.length !== 0">
          <ul>
            <li class="item mt-4" v-for="(task, i) in taskDB" :key="i">
            <div>{{ task[1] }}</div>
              <input
                type="range"
                min="0"
                max="100"
                step="1"
                v-model="task.progress"
              />
              <p>進捗：{{ task.progress }}%</p>
            </li>
          </ul>
        </div>

        <div v-else>タスクがまだありません</div>
      タスク担当について
      </div>
    </div>
    <router-link to="/" class="link">
      <button type="button" class="button-normal button-exit" @click="onExit">退室する</button>
    </router-link>
    <router-link to="/chat/" class="chat">
      <button type="button" class="button-normal" @click="onChange">チャットへ移動</button>
    </router-link>
  </div>
</template>
<style scoped>
@import url('https://fonts.googleapis.com/css2?family=M+PLUS+Rounded+1c&display=swap');

* {
  font-family: 'M PLUS Rounded 1c', 'Arial', 'Meiryo', sans-serif;
  box-sizing: border-box;
}

body {
  background-color: #fffaf5;
}

/* 全体 */
.mx-auto {
  max-width: 800px;
  margin: 0 auto;
  padding: 40px 16px;
  background-color: #fffaf5;
}

/* タイトル・ログインユーザー */
h1 {
  font-size: 32px;
  font-weight: bold;
  margin-bottom: 16px;
  color: #333;
}

.login-user {
  font-size: 14px;
  color: #666;
  margin-bottom: 24px;
}

/* タスク進捗バー */
.container {
  background: #ffffffcc;
  border-radius: 12px;
  padding: 20px;
  margin-bottom: 20px;
  box-shadow: 0 0 6px rgba(0,0,0,0.1);
}

progress {
  width: 100%;
  height: 16px;
  appearance: none;
}

progress::-webkit-progress-bar {
  background-color: #eee;
  border-radius: 8px;
}

progress::-webkit-progress-value {
  background-color: #4CAF50;
  border-radius: 8px;
}

/* タスク入力欄 */
.area {
  width: 100%;
  border: 1px solid #ccc;
  border-radius: 12px;
  padding: 10px;
  font-size: 14px;
  margin-top: 12px;
  resize: none;
  background-color: #fff;
}

/* タスク一覧 */
.item {
  background: #ffffffdd;
  border-radius: 12px;
  box-shadow: 0 0 4px rgba(0,0,0,0.05);
  padding: 16px;
  margin-bottom: 16px;
  text-align: left;
}

.assignee-input {
  margin: 6px 0;
  padding: 6px;
  border: 1px solid #ccc;
  border-radius: 10px;
  font-size: 14px;
  width: 100%;
}

/* スライダー */
input[type="range"] {
  width: 100%;
  margin-top: 10px;
}

/* ボタンエリア */
.button-normal {
  padding: 10px 20px;
  border: none;
  border-radius: 25px;
  font-size: 14px;
  cursor: pointer;
  background-color: #ffa500;
  color: white;
  transition: background-color 0.3s ease;
}

.button-normal:hover {
  background-color: #ff8800;
}

.button-exit {
  background-color: #aaa;
  margin-top: 16px;
}

.button-exit:hover {
  background-color: #888;
}

/* リンクボタン */
.link, .chat {
  text-decoration: none;
  margin-left: 8px;
}
</style>