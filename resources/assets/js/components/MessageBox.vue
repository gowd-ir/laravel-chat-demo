<template>
    <div class="row">
        <div class=" offset-3 col-6 mt-5 col-sm-10 offset-sm-1">

            <div class="card">
                <div class="card-header">
                    <strong>聊天室</strong>
                    <span class="badge badge-danger badge-pill">{{ numberOfUsers }}</span>
                    <span class="btn btn-warning btn-sm" @click="deleteHistory">清空聊天记录</span>
                </div>
                <div class="card-block">
                    <div class="badge badge-pill badge-primary" v-text="typing"></div>
                    <div class="list-group chat-room" v-chat-scroll>
                        <message v-for="(message, key) in chat.messages"
                                 color="success"
                                 :username="chat.usernames[key]"
                                 :color="chat.colors[key]"
                                 :time="chat.times[key]"
                                 :message="message"
                                 :key="key">
                        </message>
                    </div>

                </div>
                <div class="card-footer">
                    <input type="text"
                           v-model="message"
                           @keyup.enter="send"
                           class="form-control"
                           placeholder="输入您想说的话 ..">
                </div>
            </div>
        </div>
    </div>

</template>
<script>
    import Message from './Message.vue';
    export default {
        data() {
            return {
                message: '',
                chat: {
                    messages: [],
                    usernames: [],
                    colors: [],
                    times: [],
                },
                typing: '',
                numberOfUsers: 0
            }
        },
        methods: {
            send(){
                if (this.message) {

                    axios.post('/send', {
                        message: this.message,
                        chat: this.chat
                    }).then(response => {
                        this.chat.messages.push(this.message);
                        this.chat.usernames.push('You');
                        this.chat.colors.push('success');
                        this.chat.times.push(this.getTime());
                        // console.log(response);
                        this.message = '';
                    }).catch(error => {
                        console.log(error);
                    });
                }
            },
            getTime(){
                let time = new Date();
                return time.getHours() + ':' + time.getMinutes();
            },
            getOldMessages(){ // 获取之前的聊天记录
                axios.get('/getOldMessage').then(response => {
                    if (response.data != '') {
                        this.chat = response.data;
                    }
                }).catch(error => {

                });
            },
            deleteHistory(){
                axios.delete('/deleteSession').then(response => {
                    this.$toaster.success('chat history is deleted');
                    this.chat = {};
                }).catch(error => {

                });
            }
        },
        mounted(){
            Echo.private('chat')
                .listen('ChatEvent', (e) => {
                    this.chat.messages.push(e.message);
                    this.chat.usernames.push(e.user.name);
                    this.chat.colors.push('warning');
                    this.chat.times.push(this.getTime());
                    console.log(this.chat);
                    axios.post('/saveToSession', { // 保存聊天记录到Session
                        chat: this.chat
                    }).then(response => {
                        console.log(response)
                    }).catch(error => {
                        console.log(error)
                    });
                })
                .listenForWhisper('typing', (e) => {
                    if (e.name != '') {
                        this.typing = 'typing...';
                    } else {
                        this.typing = '';
                    }
                });

            Echo.join('chat')
                .here((users) => {
                    this.getOldMessages();
                    this.numberOfUsers = users.length;
                })
                .joining((user) => {
                    this.numberOfUsers += 1;
                    this.$toaster.success(user.name + ' is joined the chat room.');
                })
                .leaving((user) => {
                    this.numberOfUsers -= 1;
                    this.$toaster.info(user.name + ' is leaved the chat room.');
                })
                .listen('NewMessage', (e) => {
                    //
                });
        },
        watch: {
            message(){
                // console.log(this.message);
                Echo.private('chat')
                    .whisper('typing', {
                        name: this.message
                    });
            }
        },
        components: {
            message: Message
        }
    }
</script>
