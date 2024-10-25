<template>
    <div class="p-8">
        <input v-show="isShowEmail" v-model="email" placeholder="Enter your email" @keyup.enter="fetchGame()" />
        <div v-if="!isShowEmail">
            <h3 class="text-3xl ">Created games</h3>
            <div class="space-y-4">
                <div v-if="games.length > 0" v-for="game, ind in games" class="grid grid-cols-3">
                    <div>{{ game.name }}</div>
                    <div>{{ game.description }}</div>
                    <div>
                        <button class="bg-blue-500 text-white px-2 py-1" @click="playGame(game.id)">Play</button>
                    </div>
                </div>
                <p v-else class="text-red-500">No available games</p>
            </div>
            <template v-if="isHost">
                <h3 class="text-3xl">Join room <span class="text-blue-500">{{ roomID }}</span> to play game</h3>
                <button @click="nextQuestion" class="bg-red-500 text-white px-2 py-1">{{ indCurrentQuestion == 0 ? 'Start'
                    : 'Next' }}</button>
            </template>

            <template v-if="!isHost && !isJoinedGame">
                <h3 class="text-3xl ">Join a game</h3>
                <input v-model="roomID" placeholder="Enter room ID" @keyup.enter="enterRoom" />

            </template>
            <div>
                <h3>{{ players.length }} players joined</h3>
                <ul>
                    <li v-for="(player, index) in players" :key="index">{{ player.email }}</li>
                </ul>
            </div>
            <div v-if="question">
                <h3 class="text-xl">Question number {{ indCurrentQuestion }}</h3>
                <p>
                    {{ question?.question }}
                </p>
                <div class="flex">
                    <div v-for="option, ind in question?.options">
                        <input type="radio" :value="ind" v-model="selectedAnswer" />
                        <label for="one">{{ option }}</label>
                    </div>

                </div>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">

import { io } from "socket.io-client";

const SOCKET_EVENTS = {
    JOIN_GAME: 'join_game',
    NEXT_QUESTION: 'next_question',
    SHOW_RESULT: 'show_result',
    UPDATE_SCORE: 'update_score',
    CONNECT: 'connect',
    DISCONNECT: 'disconnect'
}

const email = ref("");
const games = ref([])
const isShowEmail = ref(true);
const roomID = ref("");
const connected = ref(false)
const isHost = ref(false)
const players = ref([])
const currentQuestion = reactive({
    question: "",
    options: []
})
const indCurrentQuestion = ref(0)
const isJoinedGame = ref(false); 

const socket = io("ws://localhost:3033");
const API = 'http://localhost:3033';

socket.on(SOCKET_EVENTS.JOIN_GAME, (newPlayers) => {
    console.log('=========>', newPlayers, typeof newPlayers);
    players.value = newPlayers
});

socket.on(SOCKET_EVENTS.NEXT_QUESTION, (question) => {
    console.log(question, currentQuestion, currentQuestion.value);
    currentQuestion.question = question.question;
    currentQuestion.options = question.options;
    indCurrentQuestion.value = question.index;
});

socket.on(SOCKET_EVENTS.SHOW_RESULT, (msg) => {
    console.log(msg);
});

socket.on(SOCKET_EVENTS.UPDATE_SCORE, (msg) => {
    console.log(msg);
});

socket.on(SOCKET_EVENTS.CONNECT, () => {
    connected.value = true;
});

socket.on(SOCKET_EVENTS.DISCONNECT, () => {
    connected.value = false;
});

const nextQuestion = () => {
    socket.emit(SOCKET_EVENTS.NEXT_QUESTION, roomID.value);
}

const fetchGame = async () => {
    isShowEmail.value = false;
    const { data, status, error, refresh } = await useFetch('/game', {
        query: { email: email.value },
        method: 'GET',
        baseURL: API
    })
    if (data?.value) {
        games.value = data.value[0];
    }
}

const playGame = async (game_id: string) => {

    const { data, status, error, refresh } = await useFetch(`/game/play/${game_id}`, {
        query: { email: email.value },
        method: 'PUT',
        baseURL: API
    })
    if (data?.value) {
        roomID.value = data.value?.room;
        isHost.value = true;
    }
}

const enterRoom = () => {
    if (roomID.value.trim()) {
        console.log('emit');
        socket.emit(SOCKET_EVENTS.JOIN_GAME, {
            room_id: roomID.value,
            email: email.value

        });
        roomID.value = '';
    }
}  
</script>