<template>
    <div>
        <input v-show="isShowEmail" v-model="email" placeholder="Enter your email"
            @keyup.enter="isShowEmail = false;" />
        <div v-if="email">
            <input v-model="roomID" placeholder="Enter room ID" @keyup.enter="enterRoom" />
            <ul>
                <li v-for="(msg, index) in messages" :key="index">{{ msg }}</li>
            </ul>
        </div>
    </div>
</template>

<script setup lang="ts">

import { io } from "socket.io-client";

const SOCKET_EVENTS = {
    JOIN_GAME: 'join_game',
    NEXT_QUESTION: 'next_question',
    SHOW_RESULT: 'show_result',
    UPDATE_SCORE: 'update_score'
}

const socket = io("ws://localhost:3033", {
    reconnectionDelayMax: 10000,
});
console.log(socket)

const email = ref("");
const isShowEmail = ref(true);
const roomID = ref("");
// const messages = ref([]); 

socket.on(SOCKET_EVENTS.JOIN_GAME, (msg) => {
    console.log('new user join game', msg);
});

socket.on(SOCKET_EVENTS.NEXT_QUESTION, (msg) => {
    console.log(msg);
});

socket.on(SOCKET_EVENTS.SHOW_RESULT, (msg) => {
    console.log(msg);
});

socket.on(SOCKET_EVENTS.UPDATE_SCORE, (msg) => {
    console.log(msg);
});

const enterRoom = () => {
    if (roomID.value.trim()) {
        console.log('emit');
        socket.emit(SOCKET_EVENTS.JOIN_GAME, {
            room: roomID.value,
            email: email.value

        });
        roomID.value = '';
    }
}  
</script>