<script setup>
import { inject, ref } from "vue"
import { useRouter } from "vue-router"
import socketManager from '../socketManager.js'

// #region global state
const userName = inject("userName")
// #endregion

// #region local variable
const router = useRouter()
const socket = socketManager.getInstance()
// #endregion

// #region reactive variable
const inputUserName = ref("")
// #endregion

// #region browser event handler
// 入室メッセージをクライアントに送信する
const onEnter = () => {
  // ユーザー名が入力されているかチェック
  if(inputUserName.value ==""){
    alert("ユーザー名を入力ください");
    return;
  }  
  // 入室メッセージを送信
   socket.emit("enterEvent",inputUserName.value);
  // 全体で使用するnameに入力されたユーザー名を格納
   userName.value = inputUserName.value;
  // チャット画面へ遷移
  router.push({ name: "chat" })
}
// #endregion
</script>

<template>
  <div class="login-container">
    <h2 class="subtitle">チャットアプリ</h2>
    <h1 class="title">TASUKU HAA-KUN</h1>

    <div class="form">
      <label class="label">ユーザー名を入力して下さい</label>
      <input type="text" v-model="inputUserName" class="input" placeholder="入力してください" />
      <button @click="onEnter" class="button">入室する</button>
    </div>

    <img src="../images/image.png" class="character" alt="キャラ画像"/>
  </div>
</template>


<style scoped>
@import url('https://fonts.googleapis.com/css2?family=M+PLUS+Rounded+1c:wght@500&display=swap');
.login-container {
  font-family: 'M PLUS Rounded 1c', sans-serif;
  text-align: center;
  padding-top: 60px;
  position: relative;
  height: 100vh;
  background-color: #fffaf5;
}

.title {
  font-size: 48px;
  font-weight: bold;
  margin-bottom: 40px;
  color: #333;
}
.subtitle {
  font-size: 16px;
  color: #888;
  margin-bottom: 4px;
  letter-spacing: 1px;
}

.form {
  display: inline-block;
  background-color: #ffffffdd;
  padding: 24px 32px;
  border-radius: 20px;
  box-shadow: 0 0 8px rgba(0,0,0,0.1);
}


.label {
  display: block;
  font-size: 18px;
  margin-bottom: 12px;
}

.input {
  padding: 10px 20px;
  font-size: 16px;
  border-radius: 30px;
  border: 1px solid #ccc;
  outline: none;
  width: 220px;
  transition: 0.3s ease;
  text-align: center;
}

.input:focus {
  border-color: #7abaff;
  background-color: #f0faff;
}

.button {
  margin-top: 20px;
  padding: 10px 30px;
  font-size: 16px;
  border: none;
  border-radius: 25px;
  background-color: #ffa500;
  color: white;
  cursor: pointer;
  transition: background-color 0.3s;
}

.button:hover {
  background-color: #ff8800;
}

.character {
  position: absolute;
  top: 70px;
  right: 70px;
  width: 350px;
  height: auto;
  z-index: 1;
}
</style>
