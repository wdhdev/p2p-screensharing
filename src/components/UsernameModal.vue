<script setup lang="ts">
import { ref, watch } from "vue";
import { username } from "../lib/user";
import Button from "./Button.vue";
import TextInput from "./TextInput.vue";

const text = ref("");
const error = ref("");

watch(text, (value) => {
    if(value.length > 16) {
        error.value = "Username cannot exceed 16 characters.";
        return;
    }

    error.value = "";
});

function submit() {
    const trimmed = text.value.trim();

    if(!trimmed.length) {
        error.value = "Username cannot be empty.";
        return;
    }

    if(trimmed.length > 16) {
        error.value = "Username cannot exceed 16 characters.";
        return;
    }

    error.value = "";
    username.value = trimmed;
}
</script>

<template>
    <h2 className="text-center text-lg font-semibold mb-4">Choose a Username</h2>

    <form @submit.prevent="submit">
        <div className="flex flex-col space-y-1 mb-3">
            <TextInput id="username" v-model="text" :error="error" placeholder="John Doe" />
        </div>

        <div className="text-center">
            <Button type="submit" class="w-full sm:w-min">Confirm</Button>
        </div>
    </form>
</template>
