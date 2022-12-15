<template>
  <v-row>
    <!-- Chat list -->
    <v-col sm="3">
      <v-card elevation="7" class="chat-list">
        <v-card class="pa-2" color="#424242" dark>Chats</v-card>
        <v-list class="py-0" two-line>
          <v-list-item-group v-model="selected" active-class="green--text">
            <template v-for="(item, index) in items">
              <v-list-item :key="item.wsm_recipient" @click="loadCoversation(item)">
                <template>
                  <v-list-item-content>
                    <v-list-item-title v-text="item.wsm_recipient"></v-list-item-title>

                    <v-list-item-subtitle
                      v-text="item.wsm_body"
                    ></v-list-item-subtitle>
                  </v-list-item-content>
                </template>
              </v-list-item>

              <v-divider
                v-if="index < items.length - 1"
                :key="index"
              ></v-divider>
            </template>
          </v-list-item-group>
        </v-list>
      </v-card>
    </v-col>

    <!-- Conversations -->
    <v-col md="9">
      <v-card elevation="7">
        <v-card class="chat-box wpp-bg" color="#efeae2">
          <v-card class="pa-2" color="#424242" dark>Ali Connors</v-card>
          <div
            v-for="(message, index) in messages"
            :key="index"
            :class="[
              'pa-1',
              'd-flex',
              { 'justify-start': !message.wsm_outbound },
              { 'justify-end': message.wsm_outbound },
            ]"
          >
            <v-card
              class="pa-2 ma-2 message-card"
              :color="message.wsm_outbound ? 'green accent-1' : 'white'"
            >
              <div>
                <div>{{ message.wsm_body }}</div>
                <p class="text-right text-subtitle-2 font-italic">
                  {{ message.created_at }}
                  <v-icon
                    v-bind="attrs"
                    :color="formatReadStatus(message.wsm_status).color"
                    small
                    v-on="on"
                    >{{ formatReadStatus(message.wsm_status).icon }}</v-icon
                  >
                </p>
              </div>
            </v-card>
          </div>
        </v-card>
        <v-divider></v-divider>
        <div class="pa-3">
          <v-text-field
            v-model="message"
            dense
            outlined
            background-color="white"
            :append-outer-icon="'mdi-send'"
            filled
            clear-icon="mdi-close-circle"
            clearable
            label="Type a message..."
            type="text"
            class="message-box"
            @click:append-outer="sendMessage"
            @keydown.enter.exact.prevent="sendMessage"
          ></v-text-field>
        </div>
      </v-card>
    </v-col>
  </v-row>
</template>

<script>
import Echo from 'laravel-echo';
import Vue from 'vue';
window.Pusher = require('pusher-js');
export default {
  data: () => ({
    selected: [2],
    items: [
      {
        wsm_body: `I'll be in your neighborhood doing errands this weekend. Do you want to hang out?`,
        wsm_recipient: 'Ali Connors',
      },
      {
        wsm_body: `Wish I could come, but I'm out of town this weekend.`,
        wsm_recipient: 'Jennifer',
      },
      {
        wsm_body:
          'We should eat this: Grate, Squash, Corn, and tomatillo Tacos.',
        wsm_recipient: 'Britta Holt',
      },
    ],
    messages: [
      {
        wsm_outbound: false,
        wsm_body: 'Nunc tellus magna, volutpat vel orci eu, venenatis vestibulum magna. Morbi fermentum, purus laoreet egestas maximus, metus ex bibendum nulla, in posuere nunc neque id nulla.',
        created_at: '05/02/2022',
        wsm_status: 'sent',
      },
      {
        wsm_outbound: true,
        wsm_body: 'estibulum pellentesque maximus lacus, quis viverra justo pharetra sed. Curabitur tempus consequat dolor, ut gravida dui pulvinar eget.',
        created_at: '05/02/2022',
        wsm_status: 'read',
      },
      {
        wsm_outbound: false,
        wsm_body: 'Nulla id eros consequat purus interdum iaculis quis ut orci. Mauris dapibus turpis sit amet egestas consectetur. ',
        created_at: '05/02/2022',
        wsm_status: 'sent',
      },
      {
        wsm_outbound: true,
        wsm_body: 'estibulum pellentesque maximus lacus, quis viverra justo pharetra sed. Curabitur tempus consequat dolor, ut gravida dui pulvinar eget.',
        created_at: '05/02/2022',
        wsm_status: 'delivered',
      },
    ],
    message: '',
    selectedConversation: {},
  }),
  created(){
    this.loadMessages();
  },
  mounted() {
    window.Echo = new Echo({
      broadcaster: 'pusher',
      key: process.env.pusherAppKey,
      cluster: process.env.pusherCluster,
      forceTLS: true,
    });
    window.Echo.channel('webhooks').listen('Webhook', (res) => {
      const message = res?.message;
      const change = res?.change;
      // console.log(this.selectedConversation?.wsm_recipient);
      console.log(message,change);
      if (this.selectedConversation?.wsm_recipient === message.wsm_recipient){
        if (change === false ){
          this.addMessage(message);
          this.scrollToBottom();
        }else{
          const msgIndex = this.messages.findIndex((m) => m?.wsm_id === message.wsm_id);
          if (msgIndex !== -1){
            Vue.set(this.messages,msgIndex, message);
          }
        }
        this.addMessage(message);
        this.scrollToBottom();
      }
    });
  },
  methods: {
    addMessage(message){
      this.messages.push(message);
      // this.messages = this.messages.concat(message);
      
    },
    async loadMessages(){
      await this.$axios.get('/messages').then(({data}) => {
        this.items = data.data;

        console.log(data);
      });
    },
    async loadCoversation(chat){
      this.selectedConversation = chat;
      await this.$axios.get('/messages/'+chat.wsm_recipient).then(({data}) => {
        this.messages = data.data;
        this.scrollToBottom();
        console.log(data);
      });
    },
    async sendMessage() {
      const payload ={
        wsm_body: this.message,
        wsm_recipient:  this.selectedConversation.wsm_recipient,
        number_id: '107160942162205',
      }
      await this.$axios.post('/messages', payload).then(({data}) => {
        this.messages.push({
          wsm_outbound: data.data.wsm_outbound,
          wsm_body: data.data.wsm_body,
          created_at: this.$moment(data.data.created_at).format('YYYY/MM/DD hh:mm'),
          status: data.data.wsm_status,
          wsm_id: data.data.wsm_id,
      });
        console.log(data);
        this.message = '';
        this.scrollToBottom();

      }).catch(error=>{
        console.log(error);
      });

    },
    scrollToBottom() {
      setTimeout(() => {
        const container = this.$el.querySelector('.chat-box');
        if (container?.scrollHeight) {
          container.scrollTop = container.scrollHeight;
        };

      }, 100);
    },
    formatReadStatus(status) {
      const statuses = {
        read: {
          color: 'blue',
          icon: 'mdi-check-all',
          // tooltip: $t('chat.msgDelivered'),
          tooltip: 'Read',
        },
        delivered: {
          color: 'grey',
          icon: 'mdi-check-all',
          // tooltip: $t('chat.msgRead'),
          tooltip: 'Delivered',
        },
        sent: {
          color: 'grey',
          icon: 'mdi-check',
          // tooltip: $t('chat.msgSent'),
          tooltip: 'Sent',
        },
        failed: {
          color: 'red',
          icon: 'mdi-alert-circle',
          // tooltip: $t('chat.msgFailed'),
          tooltip: 'Failed',
        }
      }

      if (statuses?.[status]) {
        return statuses[status]
      } else {
        return {
          color: 'grey',
          icon: 'mdi-cards-diamond-outline',
          tooltip: '',
        }
      }
    },
  }
}
</script>

<style>
.message-box .v-text-field__details {
  display: none;
}
.chat-box {
  height: 80vh;
  overflow-y: scroll;
}
.message-card {
  max-width: 60%;
}
.chat-list {
  height: 85vh;
}
.wpp-bg {
  background-image: url('/bg-whatsapp.png');
  background-repeat: repeat;
  border-color: #efeae2;
}
</style>
