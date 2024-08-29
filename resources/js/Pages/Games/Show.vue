<script setup>
import AuthenticatedLayout from '@/Layouts/AuthenticatedLayout.vue';
import { computed, onUnmounted, ref } from 'vue';
import { router, usePage } from '@inertiajs/vue3'

const props = defineProps(['game']);

const boardState = ref(props.game.state ?? [0, 0, 0, 0, 0, 0, 0, 0, 0]);
const players = ref([]);

const page = usePage()
const xTurn = computed(() => boardState.value.reduce((carry, value) => carry + value, 0) === 0)
const yourTurn = computed(() => {
    if (props.game.player_one_id === page.props.auth.user.id) {
        return xTurn.value
    }

    return !xTurn.value
})

const lines = [
    // rows
    [0, 1, 2],
    [3, 4, 5],
    [6, 7, 8],
    // columns
    [0, 3, 6],
    [1, 4, 7],
    [2, 5, 8],
    // diagonals
    [0, 4, 8],
    [2, 4, 6],
]

const channel = Echo.join(`games.${props.game.id}`)
    .here((users) => {
        players.value = users
    })
    .joining((user) => {
        router.reload({
            onSuccess: () => players.value.push(user)
        })
    })
    .leaving((user) => {
        players.value = players.value.filter(({ id }) => id !== user.id)
    })
    .listenForWhisper('PlayerMadeMove', ({ game }) => {
        boardState.value = game.state

        // check for victory
    })

const fillSquare = (index) => {
    if (!yourTurn.value) {
        return
    }

    boardState.value[index] = xTurn.value ? -1 : 1;

    router.put(route('games.update', props.game.id), {
        state: boardState.value,
    })

    channel.whisper('PlayerMadeMove', {
        game: {
            state: boardState.value,
        }
    })

    const winningLine = lines.map((line) => line.reduce((carry, index) => carry += boardState.value[index], 0))
        .find((sum) => Math.abs(sum) === 3)

    if (winningLine === -3) {
        alert('X wins!')
        return
    }

    if (winningLine === 3) {
        alert('O wins!')
        return
    }

    if (!boardState.value.includes(0)) {
        alert('Stalemate!')
    }
}



onUnmounted(() => {
    Echo.leave(`games.${props.game.id}`)
})
</script>

<template>
    <AuthenticatedLayout>
        <menu class="grid grid-cols-3 gap-1.5 max-w-fit mx-auto mt-12">
            <li v-for="(square, index) in boardState" class="bg-gray-300 size-32 grid place-items-center">
                <button @click="fillSquare(index)" v-if="square === 0" class="place-self-stretch bg-gray-200 hover:bg-gray-300 transition-colors"></button>
                <span v-else v-text="square === -1 ? 'X' : 'O'" class="text-4xl font-bold"></span>
            </li>
        </menu>

        <ul class="max-w-sm mx-auto mt-6 space-y-2">
            <li class="flex items-center gap-2">
                <span class="p-1.5 font-bold rounded bg-gray-200" :class="{ 'bg-green-200': xTurn}">X</span>
                <span>{{ game.player_one.name }}</span>
                <span :class="{ '!bg-green-500': players.find(({ id }) => id === game.player_one_id) }" class="bg-red-500 size-2 rounded-full"></span>
            </li>
            <li v-if="game.player_two" class="flex items-center gap-2">
                <span class="p-1.5 font-bold rounded bg-gray-200" :class="{ 'bg-green-200': !xTurn}">O</span>
                <span>{{ game.player_two.name }}</span>
                <span :class="{ '!bg-green-500': players.find(({ id }) => id === game.player_two_id) }" class="bg-red-500 size-2 rounded-full"></span>
            </li>
            <li v-else>Waiting for player two...</li>
        </ul>
    </AuthenticatedLayout>
</template>
