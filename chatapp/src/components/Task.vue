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
    <h1 class="text-h3 font-weight-medium">Vue.js タスク一覧</h1>
    <div class="mt-10">
      <p>ログインユーザ：{{ userName }}さん</p>
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
            タスク入力について
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
.link {
  text-decoration: none;
}
.area {
  width: 500px;
  border: 1px solid #000;
  margin-top: 8px;
}
.item {
  display: block;
}
.util-ml-8px {
  margin-left: 8px;
}
.button-exit {
  color: #000;
  margin-top: 8px;
}
.wrapper1{
}
.wrapper2{
}
.wrapper3{
}
.progress-bar {
  width: 100%;
  height: 24px;
  background-color: #eee;
  border-radius: 12px;
  overflow: hidden;
  margin-top: 8px;
}
.progress-fill {
  height: 100%;
  background-color: #4CAF50;
  transition: width 0.3s ease;
}
</style>