<script setup lang="ts">
import { ref } from 'vue'


type Message = {
  id: string,
  message: string,
  name: string,
  createdAt: number,
  type: 'text' | 'alert'
}
const connected = ref(false)
const messages = ref<Message[]>([])
const alert = ref()
const name = ref()
const text = ref()
function setAlert(message: string) {
  alert.value = "Alerte: " + message
  setTimeout(() => alert.value = undefined, 5000)
}
function addMessage(message:Message){
  messages.value.push(message)
}
function connect() {
  try {
    const websocket = new WebSocket("ws://localhost:7512")
    websocket.onmessage = (event) => {
      const data = JSON.parse(event.data)
      switch (data.action) {
        case 'getMessages':
          if (data.status == 200) {
            const messageList = data.result.map((result: { _id: any; _source: { name: any; message: any; _kuzzle_info: { createdAt: any; }; }; }) => { return { id: result._id, name: result._source.name, message: result._source.message, createdAt: result._source._kuzzle_info.createdAt } })
            messages.value = messageList
            console.log('Messages updated')
          }
          break;
        case 'create':
          const newMessage: Message = { id: data.result._id, name: data.result._source.name, message: data.result._source.message, createdAt: data.result._source._kuzzle_info.createdAt, type: 'text' }
          addMessage(newMessage)
          break;
        case 'publish':
          if (data.result._source.alert) {
            const alertMessage: Message = { id: data.requestId, message: data.result._source.alert, name: 'System', createdAt: data.timestamp, type: 'alert' }
            addMessage(alertMessage)
          }
          break;
        case 'postMessage':
          if (data.status != 200 && data.result) {
            setAlert(data.result)
          }
          break
        default:
          console.log(data)
      }
    }

    websocket.onopen = () => {
      console.log('connected')
      connected.value = true
      websocket.send(JSON.stringify({
        index: "chat",
        collection: "chat-messages",
        controller: "realtime",
        action: "subscribe",
        body: {
        },
        scope: "in",
        users: "none"
      }))

      websocket.send(JSON.stringify({
        controller: "chat-message",
        action: "getMessages"
      }))
    }
    websocket.addEventListener('close', () => {
      console.log('disconnected')
      connected.value = false
    })
    return websocket
  } catch (error) {
    console.log(error)
  }}
const websocket = connect()

function sendText(e: Event) {
  e.preventDefault()
  try {
    if (name.value && text.value) {
      const request = {
        controller: "chat-message",
        action: "postMessage",
        name: name.value,
        message: text.value
      }
      websocket?.send(JSON.stringify(request))
    }
    text.value = undefined
  } catch (error) {
    console.log(error)
  }

}
</script>

<template>
  <h1>Chat-app</h1>
  <ul class="chat-list" >
    <li :class="message.type === 'alert' ? 'chat-message alert' : 'chat-message'" v-for="message in messages"
      :key="message.id">
      <p>De <span class="chat-message_sender">{{ message.name }} </span> le {{ new
    Date(message.createdAt).toLocaleString('fr-FR') }}</p>
      <p>{{ message.message }}</p>
    </li>
  </ul>
  <button v-if="!connected" @click="connect()">Reconnect</button>
  <div class="alert" v-if="alert">{{ alert }}</div>
  <form @submit="sendText" class="chat-box">
    <div class="chat-box-wrapper">
      <div class="chat-box-wrapper_name"><label for="name"> Entrez votre nom:</label>
        <input type="text" name="name" id="name" v-model="name" required>
      </div>
      <div class="chat-box-wrapper_text">
        <label for="text"> Entrez votre message:</label>
        <textarea v-model="text" name="text" id="text"></textarea required>
      </div>
    </div>
    <button type="submit" :disabled="!name || !text">Envoyer</button>
  </form>
  
</template>

<style scoped>
.chat-list {
  list-style: none;
  padding: 10px;
  background: rgb(247, 247, 247);
  border-radius: 10px;
  height: 50vh;
  width: 70vw;
  overflow-y: scroll;
}

.chat-message_sender {
  font-weight: bold;
}

.chat-message {
  background-color: white;
  padding: 2px;
  border-radius: 5px;
  margin-bottom: 10px;
}

.alert {
  background-color: rgb(238, 91, 91);
}

.chat-box {
  display: flex;
  flex-direction: column;
  place-items: center;
  justify-content: space-between;
  border: 1px solid lightgray;
  padding: 5px;
}

.chat-box-wrapper {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  width: 100%;
  margin-bottom: 5px;
}

.chat-box-wrapper_name {
  display: flex;
  flex-direction: column;
  width: 20%;
}

.chat-box-wrapper_text {
  display: flex;
  flex-direction: column;
  width: 70%;
}
</style>
