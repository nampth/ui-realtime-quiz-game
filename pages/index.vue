<template>
    <div class="p-8">
        <input class="px-2 py-1 border rounded" v-show="isShowEmail" v-model="email" placeholder="Enter your email"
            @keyup.enter="fetchGame()" />
        <div v-if="!isShowEmail">
            <div v-if="!isJoinedGame">
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
            </div>

            <div v-if="!isHost && !isJoinedGame">
                <h3 class="text-3xl ">or Join a game</h3>
                <input class="px-2 py-1 border rounded" v-model="roomID" placeholder="Enter room ID"
                    @keyup.enter="enterRoom" />

            </div>
            <div v-if="isJoinedGame" class="space-y-8">
                <div v-if="isHost">
                    <h3 class="text-3xl">Join room <span class="text-blue-500">{{ roomID }}</span> to play game</h3>
                    <button @click="nextQuestion" class="bg-red-500 text-white px-2 py-1">{{ indCurrentQuestion == 0 ?
                        'Start'
                        : 'Next' }}</button>
                </div>
                <div class="bg-slate-100 p-4">
                    <h3 class="text-3xl">{{ players ? players.length : 0 }} players joined</h3>
                    <ol>
                        <li v-for="(player, index) in players" :key="index">Top {{ index + 1 }}: {{ player.email }} - {{
                            player.score }}</li>
                    </ol>
                </div>
                <div v-if="currentQuestion" class="bg-slate-100 p-4">
                    <h3 class="text-xl">Question number {{ indCurrentQuestion + 1 }}</h3>
                    <h3 class="text-blue-500 text-xl">{{ timeCounter > 0 ? timeCounter : 'Time up' }}</h3>
                    <p>
                        {{ currentQuestion?.question }}
                    </p>
                    <div class="flex space-x-4" v-if="timeCounter > 0">
                        <div v-for="option, ind in currentQuestion?.options">
                            <input type="radio" :value="ind" v-model="selectedAnswer" />
                            <label for="one" class="text-sm ml-1">{{ option }}</label>
                        </div>
                    </div>
                    <button class="bg-green-500 px-2 py-1 text-white" @click="submitAnswer"
                        v-if="timeCounter > 0 && !isHost && !isSubmitAnswer">Submit</button>
                </div>
                <div v-else>
                    <p class="text-red-500">Game ended</p>
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
    DISCONNECT: 'disconnect',
    ANSWER_QUESTION: 'answer_question',
    UPDATE_PLAYERS: 'update_players',
    END_GAME: 'end_game'
}

const email = ref("");
const games = ref([])
const isShowEmail = ref(true);
const roomID = ref("");
const connected = ref(false)
const isHost = ref(false)
const isSubmitAnswer = ref(false)
const players = ref([])
const currentQuestion = reactive({
    question: "",
    options: []
})
const indCurrentQuestion = ref(0)
const timeCounter = ref(0)
const isJoinedGame = ref(false);
const intervalTimeCounter = ref(null)
const selectedAnswer = ref(null)

const socket = io("ws://localhost:3033");
const API = 'http://localhost:3033';

const resetTimeCounter = () => {
    if (intervalTimeCounter.value) {
        clearInterval(intervalTimeCounter.value);
        intervalTimeCounter.value = null;
    }
    timeCounter.value = 0;
}
const setTimeCounter = (duration: number) => {
    timeCounter.value = duration;
    return setInterval(() => {
        timeCounter.value--;
    }, 1000);
}
socket.on(SOCKET_EVENTS.JOIN_GAME, (newPlayers) => {
    players.value = newPlayers.players
    indCurrentQuestion.value = newPlayers.current_index;

});

socket.on(SOCKET_EVENTS.UPDATE_SCORE, (leaderBoard) => {
    console.log('update leader board', leaderBoard);
    resetTimeCounter();
    players.value = leaderBoard;
})


socket.on(SOCKET_EVENTS.END_GAME, () => {
    resetTimeCounter();
    currentQuestion.value = null;
})

watchEffect(() => {
    if (timeCounter.value < 0) {
        resetTimeCounter()
    }
})

socket.on(SOCKET_EVENTS.NEXT_QUESTION, (question) => {
    // set question     
    isSubmitAnswer.value = false;
    currentQuestion.question = question.question;
    currentQuestion.options = question.options;
    indCurrentQuestion.value = question.index;
    // reset answer
    selectedAnswer.value = null;
    // set interval
    console.log('set interval');
    resetTimeCounter()
    intervalTimeCounter.value = setTimeCounter(question.duration)
});


socket.on(SOCKET_EVENTS.CONNECT, () => {
    connected.value = true;
});

socket.on(SOCKET_EVENTS.DISCONNECT, () => {
    connected.value = false;
});

const submitAnswer = () => {
    isSubmitAnswer.value = true;
    socket.emit(SOCKET_EVENTS.ANSWER_QUESTION, {
        selected_option: selectedAnswer.value,
        email: email.value
    });
}

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
        isJoinedGame.value = true;
    }
}

const enterRoom = () => {
    if (roomID.value.trim()) {
        socket.emit(SOCKET_EVENTS.JOIN_GAME, {
            room_id: roomID.value,
            email: email.value

        });
        roomID.value = '';
        isJoinedGame.value = true;

    }
}  
</script>