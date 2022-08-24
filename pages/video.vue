<template>
  <div>
    <b-form id="room-name-form" :style="formRoom ? 'display: none' : 'display: block'">
      Enter a Room Name to join: <input id="room-name-input" v-model="nameRoom" name="room_name"/>
      <button @click.prevent="startRoom()">Join Room</button>
    </b-form>
    <div id="video-container">
      <!-- <div v-if="type === 'local'" :id="identityLocal" :class="type === 'local' ? 'local' : ''"></div>
      <div v-if="type === 'remote'" :id="identityRemote" :class="type !== 'local' ? 'remote' : ''"></div> -->
    </div>
  </div>
</template>


<script>
import Twilio from 'twilio-video'
import axios from 'axios'
export default {
  // eslint-disable-next-line vue/component-definition-name-casing
  name: 'VideoPage',
  data() {
    // const container = document.getElementById("video-container");
    return {
      // container,
      type: '',
      identityLocal: '',
      identityRemote: '',
      formRoom: false,
      nameRoom: '',
    }
  },
  // created() {
  //   this.joinVideoRoom()
  // },
  methods: {
    async startRoom() {
      this.formRoom = true;
      // retrieve the room name
      console.log("startRoom")
      console.log(this.nameRoom)
      const response = await axios.post(`http://localhost:5000/join-room`, {
        roomName: this.nameRoom
      });
      console.log(response)
      const { token } = await response.data;
      console.log(token)

      const room = await this.joinVideoRoom(this.nameRoom, token).then(res => res)
      console.log(room);

      // render the local and remote participants' video and audio tracks
      this.handleConnectedParticipant(room.localParticipant, "local");
      const getType = room.localParticipant ? "local" : "remote";
      console.log(getType)
      room.participants.forEach(this.handleConnectedParticipant);
      room.on("participantConnected", this.handleConnectedParticipant);

      room.on("participantDisconnected", this.handleDisconnectedParticipant);
      window.addEventListener("pagehide", () => room.disconnect());
      window.addEventListener("beforeunload", () => room.disconnect());

    },
    handleConnectedParticipant(participant, types = "remote") {
      console.log(types)
      console.log(participant)
      // create a div for this participant's tracks
      const participantDiv = document.createElement("div");
      participantDiv.setAttribute("id", participant.identity);
      if(types === "local") {
        participantDiv.setAttribute("class", "local");
      } else {
        participantDiv.setAttribute("class", "remote");
      }
      const container = document.getElementById("video-container");
      container.appendChild(participantDiv);

      // iterate through the participant's published tracks and
      // call `handleTrackPublication` on them
      // eslint-disable-next-line no-console
      console.log("identity ", participant.identity);
      // eslint-disable-next-line no-console
      console.log("participant: ", participant);
      participant.tracks.forEach((trackPublication) => {
        // console.log("trackPublication: ", trackPublication);
        this.handleTrackPublication(trackPublication, participant);
      });

      // listen for any new track publications
      participant.on("trackPublished", this.handleTrackPublication);
    },
    handleTrackPublication(trackPublication, participant) {
      // function displayTrack(track) {
      //   console.log(participant.identity)
      //   const parent = document.getElementById('video-container');
      //   const child = document.createElement('p');
      //   parent.append(child);
      //   // // append this track to the participant's div and render it on the page
      //   // const participantDiv = document.getElementById(participant.identity);
      //   // console.log(participantDiv)
      //   // // track.attach creates an HTMLVideoElement or HTMLAudioElement
      //   // // (depending on the type of track) and adds the video or audio stream
      //   // participantDiv.appendChild(track.attach());
      //   // // document.getElementById('remote-media-div').appendChild(track.attach());
      // }
      function displayTrack(track) {
        // append this track to the participant's div and render it on the page
        const participantDiv = document.getElementById(participant.identity);
        // console.log(participantDiv)
        // track.attach creates an HTMLVideoElement or HTMLAudioElement
        // (depending on the type of track) and adds the video or audio stream
        participantDiv.append(track.attach());
      }

      // check if the trackPublication contains a `track` attribute. If it does,
      // we are subscribed to this track. If not, we are not subscribed.
      if (trackPublication.track) {
        // const remoteDiv = document.getElementById("div");
        displayTrack(trackPublication.track);
      }

      // listen for any new subscriptions to this track publication
      trackPublication.on("subscribed", displayTrack);
    },
    handleDisconnectedParticipant(participant) {
      // stop listening for this participant
      participant.removeAllListeners();
      // remove this participant's div from the page
      const participantDiv = document.getElementById(participant.identity);
      participantDiv.remove();
    },
    async joinVideoRoom(roomName, token) {
      // join the video room with the Access Token and the given room name
      const room = await Twilio.connect(token, {
        room: roomName,
      });
      return room;
    },
  },
}
</script>
