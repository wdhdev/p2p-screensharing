<script setup lang="ts">
import { MediaConnection } from "peerjs";
import { onMounted, onUnmounted, ref } from "vue";
import { connectionToPeer, peer, peerName } from "../lib/peer";
import Button from "./Button.vue";
import { username } from "../lib/user";

const props = defineProps<{
    isHost: boolean;
}>();

const videoEl = ref<HTMLVideoElement | null>(null);
const stream = ref<MediaStream>();
const call = ref<MediaConnection>();

// Some browsers don't support the close event so we use
// the data channel to signal a close
function onData(data: unknown) {
    if(typeof data !== "string" || props.isHost) return;

    if(data === "stream_end") {
        stream.value?.getTracks().forEach((track) => track.stop());
        stream.value = undefined;

        if(videoEl.value) videoEl.value.srcObject = null;
    }
}

onMounted(() => {
    connectionToPeer.value?.on("data", onData);

    peer.on("call", (mediaConnection) => {
        if(props.isHost) return mediaConnection.close();

        if(mediaConnection.peer !== connectionToPeer.value?.peer) return mediaConnection.close();

        mediaConnection.on("stream", (mediaStream) => {
            // Replace the previous stream if any (eg. laggy Change Source)
            if(stream.value) {
                stream.value.getTracks().forEach((track) => track.stop());
                stream.value = undefined;
            }

            stream.value = mediaStream;

            if(videoEl.value) videoEl.value.srcObject = mediaStream;
        });

        mediaConnection.answer();
    });
});

onUnmounted(() => {
    stream.value?.getTracks().forEach((track) => track.stop());

    peer.off("call");
    connectionToPeer.value?.off("data", onData);
});

function startStream() {
    if(!navigator.mediaDevices.getDisplayMedia) {
        return alert("Your browser does not support screen capture.");
    }

    navigator.mediaDevices
        .getDisplayMedia({
            audio: true,
            video: {
                frameRate: (window as any).FRAMERATE || 60, // Experimental, do not set the frame rate higher than 60 as it can cause issues
            },
        })
        .then((mediaStream) => {
            // Close previous stream if any
            if(stream.value) {
                stream.value.getTracks().forEach((track) => track.stop());
                stream.value = undefined;
            }

            if(call.value) call.value.close();

            stream.value = mediaStream;

            if(videoEl.value) videoEl.value.srcObject = mediaStream;

            call.value = peer.call(connectionToPeer.value?.peer as string, mediaStream);
        })
        .catch(console.error);
}

function stopStream() {
    stream.value?.getTracks().forEach((track) => track.stop());
    stream.value = undefined;
    call.value?.close();
    call.value = undefined;
    connectionToPeer.value?.send("stream_end");
}
</script>

<template>
    <div id="stream-section">
        <section>
            <div class="bg-slate-900 aspect-video flex items-center justify-center">
                <p v-if="!stream" class="motion-safe:animate-pulse">Waiting for a stream...</p>
                <video ref="videoEl" v-show="stream" class="w-full h-full" controls autoplay :muted="isHost"></video>
            </div>
        </section>

        <section v-if="isHost" class="h-14 bg-slate-700 flex items-center justify-center">
            <Button v-if="!stream" type="button" @click="startStream">Start Stream</Button>
            <div v-else class="px-4 w-full lg:w-96 lg:px-0 flex items-center justify-between">
                <Button type="button" @click="startStream">Change Source</Button>
                <Button type="button" class="bg-red-400 hover:bg-red-500 transition-colors" @click="stopStream">Stop Stream</Button>
            </div>
        </section>
    </div>
</template>
