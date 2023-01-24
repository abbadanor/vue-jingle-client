<script setup lang="ts">
import { reactive, ref } from "vue"
import adapter from 'webrtc-adapter';
import * as XMPP from "stanza"

const form = reactive({
  jid: "adam@chat.a-nord.se",
  useScreen: false,
})

const fullJID = ref('')
const peer = ref('test@chat.a-nord.se/')
const localVideo = ref<HTMLVideoElement | null>(null)
const remoteVideo = ref<HTMLVideoElement | null>(null)

let client: XMPP.Agent;

const localMedia: MediaStream[] = [];

const inlog = console.log.bind(console, '<<in');
const outlog = console.log.bind(console, 'out>>');

function handleConnect() {
  const transports = {
    bosh: false,
    websocket: "wss://chat.a-nord.se/xmpp-websocket"
  };

  client = XMPP.createClient({
    jid: form.jid,
    password: 'password',
    transports: transports,
    allowResumption: false
  });

  client.on('raw:incoming', function (data) {
    inlog(data)
  });
  client.on('raw:outgoing', function (data) {
    outlog(data);
  });

  client.on('session:started', function () {
    client.getRoster();
    client.sendPresence();
    client.discoverICEServers();
    fullJID.value = client.jid;
  });

  client.jingle.on('peerTrackAdded', function (session, track, stream) {
    if (track.kind === 'video' && remoteVideo.value) {
      remoteVideo.value.srcObject = stream;
      remoteVideo.value.muted = true;
      remoteVideo.value.play();
    }
  });

  client.on('jingle:incoming', function (session) {
    // if (session.addTrack) {
    //   for (const stream of localMedia) {
    //     for (const track of stream.getTracks()) {
    //       session.addTrack(track, stream);
    //     }
    //   }
    // }
    session.accept();
  });

  client.jingle.on('log', console.log);

  let media;
  if (form.useScreen) {
    media = navigator.mediaDevices.getDisplayMedia({ video: true });
  } else {
    media = navigator.mediaDevices.getUserMedia({ audio: true, video: {
      facingMode: 'environment'
    } });
  }
  media.then(stream => {
    if (!localVideo.value) {
      console.error("no ee")
      return false;
    }
    localVideo.value.srcObject = stream;
    localVideo.value.muted = true;
    localVideo.value.play();

    localMedia.push(stream);
  });
  client.connect();

  return false;
}

function handleCall() {
  var session = client.jingle.createMediaSession(peer.value);
  for (const stream of localMedia) {
    for (const track of stream.getTracks()) {
      session.addTrack(track, stream);
    }
  }
  session.start();

  return false;
};
</script>

<template>
  <div class="min-h-screen flex-col p-8 bg-gray-50">
    <div class="w-full bg-gray-200 p-8 rounded-md mb-4">
      <h1 class="font-semibold text-2xl mb-4">Connection Settings</h1>
      <form class="flex flex-col" @submit.prevent="handleConnect()">
        <label class="text-sm text-gray-600" for="jid">
          Jid:
        </label>
        <input class="rounded-md" v-model="form.jid" type="text" name="jid" />
        <label class="text-sm text-gray-600 mt-2" for="use-screen">
          Use screen
        </label>
        <input v-model="form.useScreen" type="checkbox" name="use-screen">
        <button  class="bg-gray-300 hover:bg-gray-400 p-2 rounded-md mt-4" type="submit">Connect</button>
      </form>
    </div>
    <div v-if="fullJID" class="w-full bg-gray-200 p-8 rounded-md mb-4">
      <h1 class="font-semibold text-2xl mb-2">Start video session</h1>
      <h2 class="font-semibold text-lg mb-1">My full JID</h2>
      <p>{{ fullJID }}</p>
      <form class="flex flex-col" @submit.prevent="handleCall()">
        <label class="text-sm text-gray-600 mt-2" for="peer">Peer JID:</label>
        <input class="rounded-md" v-model="peer" type="text" name="peer" />
        <button class="bg-gray-300 hover:bg-gray-400 p-2 rounded-md mt-4" type="submit">Call</button>
      </form>
    </div>
    <div v-if="fullJID" class="w-full bg-gray-200 p-8 rounded-md">
      <h1 class="font-semibold text-2xl mb-2">Video streams</h1>
      <h2 class="font-semibold text-lg mb-1">Local video</h2>
      <video ref="localVideo"></video>
      <h2 class="font-semibold text-lg mb-1">Remote video</h2>
      <video ref="remoteVideo"></video>
    </div>
  </div>
</template>
