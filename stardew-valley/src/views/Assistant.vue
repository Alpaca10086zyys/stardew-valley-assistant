<template>
  <div>
    <router-view />
  </div>
  <nav>
    <ul>
      <img src="../assets/assistant/Main_Logo_ZH.png" alt="标识" class="logo-icon">
    </ul>
  </nav>
  <div>
    <div class="button-container">
      <img src="../assets/assistant/clothes.png" class="clothes-icon "></img>
      <div class="color-button white-button" @click="changeBackground('day')"></div>
      <div class="color-button black-button" @click="changeBackground('night')"></div>
      <div class="color-button blue-button" @click="changeBackground('winter')"></div>
      <div class="user-profile-container" @mouseover="showProfile = true" @mouseleave="hideProfile">
        <img src="../assets/role/The_Player_Icon.png" class="user-image" />
        <div v-if="showProfile" class="profile-details">
          <img src="../assets/role/The_Player_Icon.png" class="profile-image" />
          <p class="user-info">用户姓名: {{ username }}</p> <button @click="logout" class="action-button">退出登录</button>
          <button @click="switchAccount" class="action-button">切换账号</button>
        </div>
      </div>
    </div>

    <component :is="currentComponent"></component>
    <div :class="['background', currentBackground]">
      <div class="chat-container">
        <div class="top-boxes">
          <div class="top-box box-one">
            <img src="../assets/assistant/robot_icon.png" class="icon" />
            <p class="suggestion-text">智能助手</p>
          </div>
          <div class="top-box" @click="goto('/GuessLike')">
            <img src="../assets/guesslike/TV.png" class="icon" />
            <p class="suggestion-text">猜你喜欢</p>
          </div>
          <div class="top-box" @click="goto('/RoleShow')">
            <img src="../assets/role/Speech_bubble.png" class="icon" />
            <p class="suggestion-text">角色对话</p>
          </div>
          <div class="top-box" @click="goto('/RoleMatching')">
            <img src="../assets/guesslike/Love_Icon.png" class="icon" />
            <p class="suggestion-text">伴侣匹配</p>
          </div>
        </div>
        <div class="left-column">
          <button class="new-session" @click="startNewSession">新会话</button>
          <div>
            <h3 class="history-chat-title">历史会话</h3>
            <ul class="session-list">
              <li v-for="session in sessions" :key="session.session_id" class="session-item">
                <button @click="selectSession(session.session_id)" class="session-button" :title="session.summary"
                  :class="{ 'selected': session.session_id === selectedSessionId }" v-if="session.summary != null">
                  {{ session.summary }}
                </button>
                <button @click="deleteSession(session.session_id)" class="delete-button"
                  v-if="session.summary != null"></button> <!-- 添加删除按钮 -->
              </li>
            </ul>
          </div>
        </div>
        <div class="right-column">
          <div class="response-box" ref="responseBox">
            <div v-if="currentSessionId">
              <p v-for="msg in chatMessages" :key="msg.id" style="padding: 1px 2%;"
                :class="{ 'user-message': msg.isUser, 'assistant-message': !msg.isUser }">
                <img v-if="msg.sender === 'User'" class="avatar" src="../assets/role/The_Player_Icon.png"
                  alt="User Avatar">
                <img v-else class="avatar" src="../assets/assistant/White_Chicken.png" alt="Assistant Avatar">
                <span v-if="msg.sender !== 'User' && msg.message === ''&& msg.isStreaming" class="loading-dots">
                  <span class="dot"></span>
                  <span class="dot"></span>
                  <span class="dot"></span>
                </span>
                <MarkdownRenderer :markdown="msg.message" />
                <img v-if="msg.sender !== 'User'" src="../assets/assistant/copy.png" class="copy-icon"
                  @click="copyToClipboard(msg.message)" alt="Copy Icon">
              </p>
            </div>

             <!-- 预设词模块的条件渲染 -->
    <div v-if="this.currentSessionId === null" class="preset-container">
      <img v-if="presets.length > 0" src="../assets/assistant/White_Chicken.png" alt="Assistant Icon" class="preset-image">
      <div class="preset-buttons">
        
        <div v-for="(preset, index) in presets" :key="index" class="preset">
          <button v-if="!preset.loading" @click="selectPreset(preset.text)">
            <img src="../assets/assistant/Magnifying_Glass.png" alt="Magnifying Glass">
            {{ preset.text }}
          </button>
          <span v-else class="loading-dots">
            <span class="dot"></span>
            <span class="dot"></span>
            <span class="dot"></span>
          </span>
        </div>
      </div>
    </div>
  </div>


          
          <div class="input-box">
            <textarea v-model="userInput" placeholder="请输入你的疑问..." @keyup.enter="sendMessage"></textarea>
            <button v-if="!isStreaming" @click="sendMessage">发送</button>
            <button v-else @click="stopStreaming">暂停</button>
          </div>
        </div>

      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

import snow from '../components/snow.vue'
import flower from '../components/flower.vue'
import star from '../components/star.vue'
import { MarkdownIt } from 'vue3-markdown-it'
import MarkdownRenderer from '../components/MarkdownRenderer.vue';
axios.defaults.baseURL = 'http://localhost:5000';

axios.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
}, error => {
  return Promise.reject(error);
});
export default {
  components: {
    snow,
    flower,
    star,
    MarkdownIt,
    MarkdownRenderer,
  },
  data() {
    return {
      showProfile: false,
      hideProfileTimer: null,
      username: localStorage.getItem('username'),
      sessions: [],//只包含session_id和summary信息
      currentSessionId: null,
      userInput: '',
      chatMessages: [],  // 保存所有的聊天消息
      currentBackground: 'background-winter',
      currentComponent: 'snow',
      controller: new AbortController(),
      response: '',
      isStreaming: false,
      selectedSessionId: null, // 新增的属性
      presets: [
        { text: '', loading: true }
      ],
    };
  },
  created() {
    this.loadSessions();
    this.checkAuth();
  },
  mounted() {
    if (this.currentSessionId === null) {
      this.fetchPresets();
    }
  },
  methods: {
    goto(route) {
      this.$router.push(route);
    },
    toggleDropdown() {
      this.showDropdown = !this.showDropdown;
    },
    checkAuth() {
      const token = localStorage.getItem('token');
      if (!token) {
        this.$router.push('/login');
      }
    },
    switchAccount() {
      localStorage.removeItem('username');
      localStorage.removeItem('token'); // 移除token
      this.$router.push('/login');
    },
    hideProfile() {
      this.hideProfileTimer = setTimeout(() => {
        this.showProfile = false;
      }, 1000);
    },
    cancelHideProfile() {
      clearTimeout(this.hideProfileTimer);
    },
    logout() {
      localStorage.removeItem('username');
      localStorage.removeItem('token'); // 移除token
      this.$router.push('/'); // 跳转到登录页面
    },
    loadSessions() {//从后端获取当前用户的所有会话
      axios
        .get('http://localhost:5000/get_sessions')
        .then(response => {
          this.sessions = response.data.sessions;
        })
        .catch(error => {
          console.error('Error loading sessions:', error);
        });
    },
    fetchPresets() {
      axios
        .get(`http://localhost:5000/generate_presets`)
        .then(response => {
          const presets = response.data.presets;
          this.presets = presets.map(preset => ({ text: preset, loading: false }));
        })
        .catch(error => {
          console.error('Failed to fetch presets:', error);
        });
    },
    async selectPreset(presetText) {
      this.userInput = presetText;
      console.log("userInput from presetText", presetText);
      try {
        const response = await axios.post('http://localhost:5000/preset_start_new_session', {
          question: presetText,
        });
        const session = response.data;
        this.sessions.push(session);
        await this.selectSession(session.session_id);
        console.log("add dialog:", session.session_id);

        // 等待 session 被正确选择后，再发送消息
        // await this.sendMessage();
        this.sendMessage();
      } catch (error) {
        console.error('Fail to select presets:', error);
      }

    },
    startNewSession() {
      axios
        .post('http://localhost:5000/new_session')
        .then(response => {
          const session = response.data;
          console.log(response.data);
          this.sessions.push(session);
          this.selectSession(session.session_id);
          console.log("add dialog:", session.session_id);
        })
        .catch(error => {
          console.error('Error creating new session:', error);
        });
    },
    selectSession(sessionId) {
  this.currentSessionId = sessionId;
  this.selectedSessionId = sessionId; // 设置选中的sessionId
  return new Promise((resolve, reject) => {
    this.loadHistory(sessionId)
      .then(() => {
        console.log("select dialog:", sessionId);
        resolve();
      })
      .catch(error => {
        console.error('Error loading history:', error);
        reject(error);
      });
  });
},

loadHistory(sessionId) {
  return axios
    .get(`http://localhost:5000/get_history/${sessionId}`)
    .then(response => {
      this.chatMessages = response.data.history;
      console.log("load dialog:", sessionId);
    })
    .catch(error => {
      console.error('Error loading history:', error);
      throw error;
    });
},
    deleteSession(sessionId) {  // 添加删除会话的方法
      axios
        .delete(`http://localhost:5000/delete_session/${sessionId}`)
        .then(() => {
          // this.sessions = this.sessions.filter(session => session !== sessionId);
          this.loadSessions();
          if (this.currentSessionId === sessionId) {
            this.currentSessionId = null;
            this.chatMessages = [];
          }
          console.log("delete dialog:", sessionId);
        })
        .catch(error => {
          console.error('Error deleting session:', error);
        });
    },
    scrollToBottom() {
      const responseBox = this.$refs.responseBox;
      responseBox.scrollTop = responseBox.scrollHeight;
    },
    stopStreaming() {
      this.isStopped = true;
      this.isStreaming = false;
      this.controller.abort();
      this.saveAnswer(this.response);
      // 找到正在流式传输的消息并停止其流式状态
      const aiMessageIndex = this.chatMessages.findIndex(msg => msg.isStreaming);
      if (aiMessageIndex !== -1) {
        this.chatMessages[aiMessageIndex].isStreaming = false;
      }
    },
    // async sendMessage() {
    //   if (this.userInput.trim() === '') return;
    //   const messageToSend = this.userInput;
    //   console.log("messageToSend", messageToSend);
    //   this.controller = new AbortController();
    //   this.response = '';
    //   this.isStopped = false;
    //   this.isStreaming = true;

    //   this.userInput = '';

    //   this.chatMessages.push({
    //     id: Date.now(),
    //     sender: 'User',
    //     message: messageToSend
    //   });

    //   this.$nextTick(() => {
    //     this.scrollToBottom();
    //   });

    //   // 初始化 AI 消息
    //   const aiMessageId = Date.now() + 1;
    //   this.chatMessages.push({
    //     id: aiMessageId,
    //     sender: 'AI',
    //     message: ''
    //   });

    //   this.$nextTick(() => {
    //     this.scrollToBottom();
    //   });

    //   const response = await fetch('http://localhost:5000/send_message', {
    //     method: 'POST',
    //     headers: {
    //       'Content-Type': 'application/json',
    //       'Authorization': `Bearer ${localStorage.getItem('token')}`  // 添加token
    //     },
    //     body: JSON.stringify({ session_id: this.currentSessionId, message: messageToSend }),//发送到服务器的数据
    //     signal: this.controller.signal
    //   });

    //   const reader = response.body.getReader();
    //   const aiMessageIndex = this.chatMessages.findIndex(msg => msg.id === aiMessageId);
    //   while (true) {
    //     if (this.isStopped) break;
    //     const { done, value } = await reader.read();
    //     if (done) break;
    //     this.response += new TextDecoder().decode(value);

    //     // 更新 AI 消息内容
    //     if (aiMessageIndex !== -1) {
    //       this.chatMessages[aiMessageIndex].message = this.response;
    //     }
    //     this.$nextTick(() => {
    //       this.scrollToBottom();
    //     });
    //   }

    //   this.isStreaming = false;
    //   // 在消息完全接收后或流式输出停止后保存答案
    //   if (!this.isStopped) {
    //     this.saveAnswer(this.response);
    //   }

    //   const sessionIndex = this.sessions.findIndex(session => session.session_id === this.currentSessionId);
    //   //更新session summary
    //   if (sessionIndex != -1 && this.sessions[sessionIndex].summary === '新会话') {
    //     axios
    //       .get(`http://localhost:5000/update_summary/${this.currentSessionId}`)
    //       .then(response => {
    //         const updatedSummary = response.data.summary;
    //         this.sessions[sessionIndex].summary = updatedSummary;
    //         console.log("update summary:", this.currentSessionId, updatedSummary);
    //       })
    //       .catch(error => {
    //         console.error('Error loading history:', error);
    //         this.isStreaming = false;
    //       });
    //     this.loadSessions();
    //   }
    // }
    // ,
    async sendMessage() {
      if (this.userInput.trim() === '') return;
      const messageToSend = this.userInput;
      console.log("messageToSend", messageToSend);
      this.controller = new AbortController();
      this.response = '';
      this.isStopped = false;
      this.isStreaming = true;

      this.userInput = '';

      this.chatMessages.push({
        id: Date.now(),
        sender: 'User',
        message: messageToSend
      });

      this.$nextTick(() => {
        this.scrollToBottom();
      });

      // 初始化 AI 消息
      const aiMessageId = Date.now() + 1;
      this.chatMessages.push({
        id: aiMessageId,
        sender: 'AI',
        message: '', isStreaming: true
      });

      this.$nextTick(() => {
        this.scrollToBottom();
      });

      try {

        const response = await fetch('http://localhost:5000/send_message', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${localStorage.getItem('token')}`  // 添加token
          },
          body: JSON.stringify({ session_id: this.currentSessionId, message: messageToSend }),
          signal: this.controller.signal
        });
        const reader = response.body.getReader();
        const aiMessageIndex = this.chatMessages.findIndex(msg => msg.id === aiMessageId);
        const decoder = new TextDecoder();

        while (true) {
          if (this.isStopped) break;
          const { done, value } = await reader.read();
          if (done) break;
          this.response += decoder.decode(value, { stream: true });
          // 更新 AI 消息内容
          console.log("id", aiMessageIndex)
          if (aiMessageIndex !== -1) {
            this.chatMessages[aiMessageIndex].message = this.response;
          }
          this.$nextTick(() => {
            this.scrollToBottom();
          });
        }

        this.isStreaming = false;
        // 在消息完全接收后或流式输出停止后保存答案
        if (!this.isStopped) {
          this.saveAnswer(this.response);
        }

        const sessionIndex = this.sessions.findIndex(session => session.session_id === this.currentSessionId);
        //更新session summary
        if (sessionIndex !== -1 && this.sessions[sessionIndex].summary === '新会话') {
          axios
            .get(`http://localhost:5000/update_summary/${this.currentSessionId}`)
            .then(response => {
              const updatedSummary = response.data.summary;
              this.sessions[sessionIndex].summary = updatedSummary;
              console.log("update summary:", this.currentSessionId, updatedSummary);
            })
            .catch(error => {
              console.error('Error loading history:', error);
              this.isStreaming = false;
            });
          this.loadSessions();
        }
      } catch (error) {
        console.error('Error:', error);
        this.isStreaming = false;
      }
    },

    async saveAnswer(answer) {
      try {
        await axios.post('http://localhost:5000/save_answer', {
          session_id: this.currentSessionId,
          answer: answer
        });
      } catch (error) {
        console.error('Error saving answer:', error);
      }
    },
    changeBackground(type) {
      if (type === 'day') {
        this.currentBackground = 'background-day';
        this.currentComponent = 'flower';
      } else if (type === 'night') {
        this.currentBackground = 'background-night';
        this.currentComponent = 'star';
      } else if (type === 'winter') {
        this.currentBackground = 'background-winter';
        this.currentComponent = 'snow';
      }
    },
    copyToClipboard(text) {
      navigator.clipboard.writeText(text).then(() => {
        alert('复制成功');
      }).catch(err => {
        console.error('复制失败', err);
      })
    }
  }
};
</script>

<style scoped lang="scss">
//呼吸灯效果
.loading-dots {
  display: inline-flex;
  justify-content: center;
  align-items: center;
  padding-left: 5px;
  /* 调整与头像的距离 */
}

.dot {
  width: 8px;
  height: 8px;
  margin: 0 2px;
  background-color: #333;
  border-radius: 50%;
  animation: blink 1.4s infinite both;
}

.dot:nth-child(1) {
  animation-delay: 0s;
}

.dot:nth-child(2) {
  animation-delay: 0.2s;
}

.dot:nth-child(3) {
  animation-delay: 0.4s;
}

@keyframes blink {

  0%,
  80%,
  100% {
    opacity: 0;
  }

  40% {
    opacity: 1;
  }
}

.user-profile-container {
  //border-radius: 50%;
  width: 25%;

  cursor: pointer;
}

.user-image {
  width: 100%;
  height: 100%;
  //border-radius: 50%;
}

.profile-details {
  position: absolute;
  top: 60px;
  right: 0;
  width: 200px;
  background-color: #facb95;
  border: 1px solid #8b4513;
  border-radius: 10px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  padding: 10px;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.profile-image {
  width: 80px;
  height: 80px;
  //border-radius: 50%;
  margin-bottom: 10px;
}

.user-info {
  margin: 5px 0;
  color: #8b4513;
}

.action-button {
  margin: 5px 0;
  padding: 5px 10px;
  background-color: #8b4513;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  width: 100%;
  text-align: center;
}

nav img {
  position: absolute;
  height: 25px;
  margin-top: 15px;
  margin-right: 15px;
  vertical-align: middle;
}

.logo-icon {
  position: absolute;
  top: 0px;
  left: 15px;
  width: 180px;
  height: auto;
  transform-origin: top left;
}

.loading-container {
  display: flex;
  align-items: center;
  justify-content: center;
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(255, 255, 255, 0.8);
  z-index: 1000;
}

.loading-rabbit {
  width: 100px;
  height: auto;
}

.loading-text {
  color: #8a390a;
  font-weight: bold;
}

loading-container {
  display: flex;
  align-items: center;
  justify-content: center;
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(255, 255, 255, 0.8);
  z-index: 1000;
}

.loading-rabbit {
  width: 100px;
  height: auto;
}

.loading-text {
  color: #8a390a;
  font-weight: bold;
}

.snowflake {
  --size: 0.4vw;
  width: var(--size);
  height: var(--size);
  background-size: 100% 100%;
  position: fixed;
  top: -5vh;
  z-index: 1;
}

@keyframes snowfall {
  100% {
    transform: translate3d(var(--end), 100vh, 0);
  }
}

@for $i from 1 through 100 {
  .snowflake:nth-child(#{$i}) {
    --size: #{random(10) * 0.1}vw;
    --end: #{random(20) - 70}vw;
    left: #{random(150)}vw;
    animation: snowfall #{10 + random(20)}s linear infinite;
    animation-delay: -#{random(10)}s;
  }
}

.chat-container {
  justify-content: center;
  margin-top: 10%;
  display: flex;
  height: 80%;
  width: 65%;
  margin: 5% auto;
  flex-direction: row; //修改为水平排列
  //flex-direction: column;
  align-items: center;
  position: relative;
  padding: 20px;
  border-radius: 20px;
  background-color: #fcc587f0;
  box-shadow:
    inset #8a390a 0 0 0 2px,
    inset #ba4d0d 0 0 0 5px,
    inset #ffa845 0 0 0 9px,
    inset #cd710f 0 0 0 12px,
    inset #ba4d0d 0 0 0 14px,
    inset #8a390a 0 0 0 16px;
}

.top-boxes {
  display: flex;
  position: absolute;
  justify-content: space-between;
  bottom: 100%;
  /* 调整这个控制与 chat-container 顶部的距离 */
  left: 0;
  right: 0;
  padding: 0 20%;
  gap: 10px;
  /* 添加方框之间的间距 */

}

.top-box {
  width: 30%;
  height: 35px;
  background-color: #fcc587fe;
  box-shadow:
    -1px 0px 0 0 #8a390a,
    -2px 0px 0 0 #cd710f,
    /* 左边阴影 */
    -3px 0px 0 0 #ffa845,
    -4px 0px 0 0 #cd710f,
    -5px 0px 0 0 #8a390a,

    1px 0px 0 0 #8a390a,
    /* 右边阴影 */
    2px 0px 0 0 #cd710f,
    3px 0px 0 0 #ffa845,
    4px 0px 0 0 #cd710f,
    5px 0px 0 0 #8a390a,


    inset 0px 1px 0 0 #8a390a,
    /* 上边阴影 */
    inset 0px 2px 0 0 #cd710f,
    inset 0px 3px 0 0 #ffa845,
    inset 0px 4px 0 0 #cd710f,
    inset 0px 5px 0 0 #8a390a;
  cursor: pointer;
  transform: translateY(16px);
  display: flex;
  align-items: center;
  justify-content: center;
}

.box-one {
  cursor: auto;
  height: 51px;

}


.left-column {
  position: relative;
  width: 30%;
  padding: 20px;
  height: 81%;
  margin-left: 35px;
  margin-right: 20px;
  border: 2px solid #cccccc00;
  border-radius: 10px;
  background-color: rgba(255, 255, 255, 0.833);
  box-shadow: 4px 4px 3px rgba(0, 0, 0, 0.2);
}

.right-column {
  width: 90%;
  height: 90%;
  //border:  5px solid #e20e0e;
  display: flex;
  flex-direction: column;
  row-gap: 10px;
  justify-content: space-between;
}

.new-session {
  width: 100%;
  font-size: 1.02em;
  box-shadow: 4px 4px 3px rgba(0, 0, 0, 0.2);
}

.history-chat-title {
  padding-top: 6%;
  padding-left: 1%;
  color: #8a390a;
  font-size: 1.2em;
  text-shadow: 2px 2px 2px #5c190b3d;
}

.session-list {
  position: absolute;
  list-style-type: none;
  padding: 0;
  overflow-y: auto;
  overflow-x: hidden;
  scrollbar-color: #8a390a rgba(255, 255, 255, 0.9);
  height: 70%;
  width: 88%;
  border-radius: 10px;
}

.session-item {
  display: flex;
  align-items: center;
  margin-bottom: 15px;
  margin-right: 15px;
}

.session-button {
  flex-grow: 1;
  padding: 10px;
  background-color: rgba(255, 255, 255, 0.833);
  color: #ba4d0d;
  border-bottom-style: 2px solid #ba4d0d;
  border-radius: 4px;
  cursor: pointer;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  max-width: 80%;
  /* 固定按钮最大宽度 */
  transition: background-color 0.5s ease; //过渡效果
}

.session-button:hover {
  background-color: rgba(254, 236, 216, 0.833);
}

@keyframes background-fade {
  from {
    background-color: rgba(254, 236, 216, 0.833);
  }

  to {
    background-color: rgba(230, 178, 120, 0.833);
  }
}

.session-button.selected {
  animation: background-fade 0.5s forwards;
  /* 应用动画 */
}

.session-button::after {
  content: attr(title);
  position: absolute;
  white-space: nowrap;
  overflow: visible;
  max-width: none;
  text-overflow: none;
  visibility: hidden;
}

.delete-button {
  padding: 15px;
  margin-left: 15px;
  background: url('../assets/assistant/delete.png') no-repeat center center;

  border: 2px solid#f8965eab;
  border-radius: 4px;
  cursor: pointer;
}

.delete-button:hover {
  transform: scale(1.05);
}

.background {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;

  z-index: -1;
}

.background-winter {
  background: url('../assets/assistant/bkgd-winter.png') no-repeat center center;
  background-size: cover;
}

.background-day {
  background: url('../assets/assistant/bkgd-day.jpg') no-repeat center center;
  background-size: cover;
}

.background-night {
  background: url('../assets/assistant/bkgd-night.jpg') no-repeat center center;
  background-size: cover;
}

.response-box {
  justify-content: center;
  width: 90%;
  //border: 2px solid #ccc;
  border-radius: 10px;
  padding: 10px;
  margin-top: 2%;
  background-color: rgba(255, 255, 255, 0.9);
  overflow-y: auto;
  height: 100%;
  //font-family:"";
  font-size: 1.2em;
  color: #9c3f0a;
  font-weight: bold;
  text-shadow: 2px 2px 2px #5c190b3d;
  box-shadow: 4px 4px 3px rgba(0, 0, 0, 0.2);
  scrollbar-color: #8a390a rgba(255, 255, 255, 0.9);
}


.input-box {

  display: flex;
  align-items: center;
  width: 94%;
  height: 12%;
}

textarea {
  height: 30%;
  flex-grow: 1;
  padding: 10px;
  margin-right: 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
  resize: none;
  min-width: 200px;
  box-shadow: 4px 4px 3px rgba(0, 0, 0, 0.2);
}

button {
  padding: 10px 20px;
  background-color: #ba4d0d;
  border-color: #e9f5ff00;
  color: white;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  text-shadow: 2px 2px 2px #5c190b3d;
  box-shadow: 4px 4px 3px rgba(0, 0, 0, 0.2);
}

button:hover {
  background-color: #99410f;
}

.user-message {
  text-align: right;
  border: 1px solid #99410f;
  border-radius: 4px;
  padding: 5px;
  margin: 10px 10px;
  background-color: #fb955ae0;
  display: flex;
  align-items: center;

}

.assistant-message {
  position: relative;
  text-align: left;
  border: 1px solid #99410f;
  border-radius: 4px;
  padding: 5px;
  margin: 5px 0;
  background-color: #f1f1f1;
  display: flex;
  align-items: center;
}

.avatar {
  width: 30px;
  height: 30px;
  border-radius: 50%;
  margin-right: 10px;
}

.button-container {
  //border: 5px solid #9c3f0a;
  position: absolute;
  top: 20px;
  right: 20px;
  display: flex;
  gap: 10px;
}

.clothes-icon {
  width: 25px;
  height: 25px;
}

.user-icon {
  width: 50px;
  height: 50px;
}

.color-button {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  cursor: pointer;
  border: 2px solid #ccc;
}

.white-button {
  background-color: rgb(254, 250, 227);
}

.black-button {
  background-color: rgb(25, 56, 147);
}

.blue-button {
  background-color: rgb(145, 198, 244);
}

.markdown-it {
  font-family: Arial, sans-serif;
}

.markdown-it ul {
  list-style-type: disc;
  padding-left: 20px;
}

.copy-icon {
  position: absolute;
  top: 5px;
  right: 5px;
  width: 15px;
  height: 15px;
  cursor: pointer;
}

.icon {
  width: 25px;
  height: 25px;
}

.suggestion-text {
  font-size: 16px;
  color: #8a390a;
  font-weight: bold;
  text-shadow: 2px 2px 2px #5c190b3d;
}

.preset-container {
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  height: 100%;
  margin: 0 20px;
}

.preset-image {
  display: block;
  margin-bottom: 10px;
}

.preset-buttons {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
}

.preset {
  flex: 1 1 20%; 
  display: flex;
  justify-content: center;
  padding: 10px; /* 添加一些内边距以保持间距 */
}

.preset button {
  display: flex;
  align-items: center;
  border: 1.5px solid brown;
  border-radius: 5px;
  box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.3);
  background-color: white;
  color: brown;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.3);
  margin: 0 0px;
  padding: 10px 12px;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.3s, color 0.3s;
  text-align: center;
}

.preset button:hover {
  background-color: #f5f5f5;
}

.preset button:active {
  background-color: #e0e0e0;
}

.preset button img {
  width: 20px;
  height: 20px;
  margin-right: 8px;
}

</style>