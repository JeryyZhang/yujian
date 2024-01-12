<script setup>
import { reactive, ref, computed, onBeforeMount,watch} from "vue";
import axios from 'axios'

let sendMsg = ref("");
let msgList = ref([]);
let isLoading = ref(false);
//配置面板控制
let isShowConfig = ref(true);
function isShowConfigContr(params) {
  isShowConfig.value = !isShowConfig.value;
}
//配置
let permission = ref(null);
let key = ref(null);
let rolePre = ref('')
let temperature = ref(10);
let presence_penalty = ref(10);
let frequency_penalty = ref(10);
let chatContrl = ref(false)
let temperatureFillter = computed(() => temperature.value / 10);
let presence_penaltyFillter = computed(() => presence_penalty.value / 10);
let frequency_penaltyFillter = computed(() => frequency_penalty.value / 10);

function clearn() {
  msgList.value = []
  isShowConfigContr()
}

function confirm() {
  checkPW()
  isShowConfigContr()
}

//聊天
function ClipboardTxt(e) {
  navigator.clipboard.writeText(e.target.innerText).then(function() {
  /* clipboard successfully set */
  console.log(e.target.innerText);
}, function() {
  /* clipboard write failed */
  console.log('no');
});
} 

const vScroll = {
  updated(el, binding, vnode, prevVnode) {
    el.scrollTop = el.scrollHeight;
  },
};

let send = async () => {
  msgList.value.push({
    type: "user",
    value: sendMsg.value,
  });
  let msg = sendMsg.value;
  sendMsg.value = "";

  msgList.value.push({
    type: "ai",
    value: "",
  });
  isLoading.value = true;

  if (chatContrl.value) {
    
    const res = await fetch(
    "https://service-4b2mja88-1259193519.sg.apigw.tencentcs.com/v1/chat/completions",
    {
      headers: {
        "Content-Type": "application/json",
        Authorization:
          `Bearer ${key.value}`,
      },
      method: "POST",
      body: JSON.stringify({
        model: "gpt-3.5-turbo",
        messages: [
          {
            role: "system",
            content: rolePre.value,
          },
          {
            role: "user",
            content: msg,
          },
        ],
        temperature:temperatureFillter.value,
        presence_penalty:presence_penaltyFillter.value,
        frequency_penalty:frequency_penaltyFillter.value,
        stream: true,
      }),
    }
  );

  if (res.status !== 200) {
    isLoading.value = false;
    msgList.value[msgList.value.length - 1].value += "脑子进水了，等会再试";
    throw new Error("OpenAI API returned an error");
  }

  const reader = res.body.getReader();
  const decoder = new TextDecoder();
  let packed;
  while (!(packed = await reader.read()).done) {
    let result = decoder.decode(packed.value); // uint8array转字符串
    const lines = result.trim().split("\n\n"); // 拆分返回的行
    for (let i in lines) {
      let line = lines[i].substring(6); // 去掉开头的 data:
      if (line === "[DONE]") {
        // 结束
        break;
      }
      let data = JSON.parse(line);
      let delta = data["choices"][0]["delta"];
      isLoading.value = false;
      if(delta.content){
    msgList.value[msgList.value.length - 1].value += delta.content;
      }

    }
  }
  }else{
    isLoading.value = false;
    msgList.value[msgList.value.length - 1].value += "你没有权限使用";
  }

};

onBeforeMount(()=>{
  checkPW()
})
watch(permission,(newV,oldV)=>{
  localStorage.removeItem('permission')
  localStorage.setItem('permission',newV)
})
async function checkPW() {
  
  let cache = localStorage.getItem('permission')
  if(cache){
    permission.value = cache
    let pw = await axios({
    url:"https://fc-mp-729ef06d-bfa2-40a0-be90-3a64436c3d5a.next.bspapp.com/pw",
    method:"POST",
    data:{
      permission:cache
    }
  })
  chatContrl.value = pw.data
  }else{
    isShowConfig.value = false
  }
}
//历史
let history = reactive([]);

</script>

<template>
  <main>
    <!-- <header>
      <div class="breath"></div>
    </header> -->
    <section class="chat" v-scroll>
      <div class="msgs" v-scroll v-show="isShowConfig" @dblclick="ClipboardTxt">
        <template v-for="(item, index) of msgList" :key="index">
          <div class="ai" v-if="item.type == 'ai'">
            <img
              src="http://zheyu.site/chat/%E9%9A%83%E8%A7%81.svg"
              width="40"
              height="40"
              alt=""
            />
            <div
              :class="{
                aiMsg: true,
                msgLoading: index == msgList.length - 1 && isLoading,
              }"
            >
              {{ item.value }}
            </div>
          </div>
          <div class="user" v-if="item.type == 'user'">
            {{ item.value }}
          </div>
        </template>
      </div>
      <div class="setting" v-show="!isShowConfig">
        <!-- <section class="history">
          <h3>历史</h3>
          <ul>
            <li>123</li>
            <li>123</li>
            <li>123</li>
            <li>123</li>
          </ul>
          <div class="add">新启对话</div>
        </section> -->
        <section class="config">
          <h3>配置</h3>
          <div class="form">
            <span>权限密码</span>
            <input
              type="password"
              v-model="permission"
              placeholder="无权限密码不可使用"
            />
          </div>
          <div class="form">
            <span>调用密钥</span>
            <input
              type="password"
              v-model="key"
              placeholder="默认key值失效时填写自己的"
            />
          </div>
          <div class="form">
            <span>身份角色</span>
            <input
              type="text"
              v-model="rolePre"
              placeholder="具体描述AI的设定"
            />
          </div>
          <div class="more">
            <span>发散思维</span>
            <input
              type="range"
              min="0"
              max="20"
              steps="1"
              v-model="temperature"
            />
            <i>{{ temperatureFillter }}</i>
          </div>
          <div class="more">
            <span>存在惩罚</span>
            <input
              type="range"
              min="0"
              max="20"
              steps="1"
              v-model="presence_penalty"
            />
            <i>{{ presence_penaltyFillter }}</i>
          </div>
          <div class="more">
            <span>频率惩罚</span>
            <input
              type="range"
              min="0"
              max="20"
              steps="1"
              v-model="frequency_penalty"
            />
            <i>{{ frequency_penaltyFillter }}</i>
          </div>
          <div class="btn_contianer">
            <div class="btn clear" @click="clearn">清空</div>
            <!-- <div class="btn export">导出</div> -->
            <div class="btn confirm" @click="confirm">确认</div>
          </div>
        </section>
      </div>
    </section>
    <footer>
      <div class="search">
        <div class="breath" @click="isShowConfigContr"></div>
        <input
          type="text"
          v-model="sendMsg"
          @keyup.enter.native="send"
          placeholder="逻辑清晰有助于回答更准确" />
        
      </div>
      <div class="send" @click="send">问</div>
    </footer>
  </main>
</template>

<style scoped>
main {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
}
.breath {
  position: absolute;
  left: 5px;
  top: 50%;
  width: 20px;
  height: 20px;
  transform: translateY(-50%);
  background: transparent;
  border: 5px solid white;
  border-radius: 50%;
  cursor: pointer;
}
.chat {
  flex: 1;
  box-sizing: border-box;
  padding: 50px 10% 0 10%;
  overflow-y: auto;
  overflow-x: hidden;

}
@media screen and (max-width: 600px){
    .chat{
      padding: 50px 10px 0 10px;
    }
}
.chat::-webkit-scrollbar {
  display: none;
}
.setting {
  width: 100%;
  height: 100%;
  display: flex;
  padding-bottom: 20px;
  box-sizing: border-box;
}
.history {
  width: 30%;
  min-height: 100%;
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  box-sizing: border-box;
  padding: 0 30px;
}
.history ul::-webkit-scrollbar {
  width: 15px;
}
.history ul::-webkit-scrollbar-track {
  background: rgba(179, 177, 177, 0);
  border-radius: 10px;
}
.history ul::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.534);
  border-radius: 10px;
}
.history ul::-webkit-scrollbar-thumb:hover {
  background: rgb(100, 100, 100);
  border-radius: 10px;
}

.history ul {
  width: 100%;
  overflow: auto;
  box-sizing: border-box;
  padding: 0 10px 0 0;
}
.history li {
  width: 100%;
  height: 30px;
  line-height: 30px;
  margin-bottom: 25px;
  border: 1px solid;
  border-radius: 3px;
  cursor: pointer;
  padding: 0 10px;
  box-sizing: border-box;
}
.config {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}
.config input {
  flex: 1;
  height: 30px;
  padding: 0 20px;
}
.config span {
  margin-right: 20px;
}
.form {
  display: flex;
  align-items: center;
}
.form input {
  flex: 1;
}
.btn_contianer {
  width: 100%;
  display: flex;
  justify-content: space-around;
}
.btn {
  width: 20%;
  height: 30px;
  line-height: 30px;
  text-align: center;
  border: 1px solid;
  border-radius: 10px;
  cursor: pointer;
}
.more {
  display: flex;
  align-items: center;
}
.more span {
  display: inline-block;
  height: 30px;
  line-height: 30px;
}
.history .add {
  width: 100%;
  height: 30px;
  line-height: 30px;
  text-align: center;
  margin-top: 30px;
  border: 1px solid;
  border-radius: 10px;
  cursor: pointer;
}
.msgs {
  display: flex;
  flex-direction: column;
  width: 100%;
  overflow-y: scroll;
}
.msgs::-webkit-scrollbar {
  display: none;
}
.ai {
  width: 70%;
  display: flex;
  align-items: flex-start;
  padding: 20px 0;
}
.ai img {
  box-sizing: border-box;
  padding: 5px;
  border: 1px solid;

  border-radius: 50%;
  margin-right: 20px;
}
.aiMsg {
  min-width: 10%;
  min-height: 40px;
  padding: 20px;
  box-sizing: border-box;
  word-wrap: break-word;
  border-radius: 10px;
  background: #202020;
  box-shadow: 4px 4px 10px black;
  transition: 0.5s all;
  transform: translateY(10px);
  opacity: 1;
  white-space: pre-wrap;
  white-space: pre-line;
  animation: up 0.5s ease-in forwards;
}
.msgLoading {
  display: flex;
  justify-content: center;
  align-items: center;
}

.msgLoading::before,
.msgLoading::after,
.msgLoading::before {
  content: "";
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background-color: rgb(255, 255, 255);
  margin: 0 5px;
  animation: pulse 1.5s ease-in-out infinite;
}

.msgLoading::before {
  animation-delay: 0s;
}

.msgLoading::after {
  animation-delay: 500ms;
}

.msgLoading::before::before {
  animation-delay: 1000ms;
}

@keyframes pulse {
  0% {
    transform: scale(0.5);
  }

  50% {
    transform: scale(1);
  }

  100% {
    transform: scale(0.5);
  }
}
@keyframes up {
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
.user {
  margin-top: 20px;
  align-self: flex-end;
}
footer {
  width: 100%;
  min-height: 56px;
  box-sizing: border-box;
  padding: 0 20px;
  display: flex;
  justify-content: center;
  align-items: center;
}
@media screen and (max-width: 600px){
  footer{
      padding: 0 10px;
    }
}
.search {
  position: relative;
  width: 80%;
  height: 40px;
}
.search input {
  width: 100%;
  height: 100%;
  border-radius: 30px;
  padding: 10px 0 10px 40px;
  margin: 0;
  border: none;
  outline: none;
  resize: none;
  font-size: 16px;
  overflow: auto;
  box-sizing: border-box;
}
input:focus {
  outline: none;
}
.send {
  background: buttonface;
  width: 40px;
  height: 40px;
  border-radius: 30px;
  line-height: 40px;
  text-align: center;
  margin-left: 10px;
  cursor: pointer;
}
</style>
