<template>
  <div>
    <b-form id="room-name-form" :style="formRoom ? 'display: none' : 'display: block'">
      Enter a Room Name to join: <input id="room-name-input" v-model="nameRoom" name="room_name"/>
      <button @click.prevent="startRoom()">Join Room</button>
    </b-form>
    <div id="video-container">
      <!-- <div v-if="type === 'local'" :id="identityLocal" :class="type === 'local' ? 'local' : ''"></div>
      <div v-if="type === 'remote'" :id="identityRemote" :class="type !== 'local' ? 'remote' : ''"></div> -->
      <div class="card-option" :style="formRoom ? 'display: block' : 'display: none'">
        <div style="display: flex; justify-content: space-between;">
          <div style="color: #6e6b7b;">15:00 - 15:25</div>
          <div class="card-time">
            <font-awesome-icon icon="fa-solid fa-circle" style="font-size: 8px; color: #6FC2C6; margin-right: 0.5rem;"/>
            <span>13:23</span>
          </div>
        </div>
        <div style="display: flex;">
          <div style="color: #6e6b7b;">นพ.สตรีฟ โรโนอัวล์</div>
          <div class="disease">โรคทั่วไป</div>
        </div>
        <div style="display: flex; float: right; align-items: center; margin-top: 16px;">
          <div class="ciclie-icon">
            <font-awesome-icon class="icon-custom" icon="fa-solid fa-rotate" />
          </div>
          <div class="ciclie-icon-phone" @click.prevent="onDisconected()">
            <font-awesome-icon class="icon-phone" icon="fa-solid fa-phone-flip" />
          </div>
          <div class="ciclie-icon">
            <font-awesome-icon class="icon-custom" icon="fa-solid fa-microphone" />
          </div>
          <div class="ciclie-icon" @click.prevent="onMuteUnmuteClick(TrackType.Video)">
            <font-awesome-icon v-if="!isVideoMuted" class="icon-custom" icon="fa-solid fa-video" />
            <font-awesome-icon v-if="isVideoMuted" class="icon-custom" icon="fa-solid fa-video-slash" />
          </div>

        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Twilio, { Room } from 'twilio-video'
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
      isVideoMuted: false,
      TrackType: {
        Audio: "Audio",
        Video: "Video",
      },
      room: Room,
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

      this.room = await this.joinVideoRoom(this.nameRoom, token).then(res => res)
      console.log(this.room);

      // render the local and remote participants' video and audio tracks
      this.handleConnectedParticipant(this.room.localParticipant, "local");
      const getType = this.room.localParticipant ? "local" : "remote";
      console.log(getType)
      this.room.participants.forEach(this.handleConnectedParticipant);
      this.room.on("participantConnected", this.handleConnectedParticipant);

      this.room.on("participantDisconnected", this.handleDisconnectedParticipant);
      window.addEventListener("pagehide", () => this.room.disconnect());
      window.addEventListener("beforeunload", () => this.room.disconnect());


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

      function displayTrack(track) {
        // append this track to the participant's div and render it on the page
        const participantDiv = document.getElementById(participant.identity);

        // track.attach creates an HTMLVideoElement or HTMLAudioElement
        // (depending on the type of track) and adds the video or audio stream
        participantDiv.append(track.attach());
      }

      // check if the trackPublication contains a `track` attribute. If it does,
      // we are subscribed to this track. If not, we are not subscribed.
      if (trackPublication.track) {

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
    attachTrack(track, participant) {
      const $media = this.$(
        `div#${participant.sid} > ${track.kind}`,
        this.$participants
      );
      $media.css("opacity", "");
      track.attach($media.get(0));
      if (track.kind === "video" && participant === this.activeParticipant) {
        track.attach(this.$activeVideo.get(0));
        this.$activeVideo.css("opacity", "");
      }
    },
    detachTrack(track, participant) {
      const $media = this.$(
        `div#${participant.sid} > ${track.kind}`,
        this.$participants
      );
      const mediaEl = $media.get(0);
      $media.css("opacity", "0");
      track.detach(mediaEl);
      mediaEl.srcObject = null;
      if (track.kind === "video" && participant === this.activeParticipant) {
        const activeVideoEl = this.$activeVideo.get(0);
        track.detach(activeVideoEl);
        activeVideoEl.srcObject = null;
        this.$activeVideo.css("opacity", "0");
      }
    },
    trackPublished(publication, participant) {
      if (publication.track) {
        this.attachTrack(publication.track, participant);
      }
      publication.on("subscribed", (track) => {
        this.attachTrack(track, participant);
      });
      publication.on("unsubscribed", (track) => {
        this.detachTrack(track, participant);
      });
    },
    mute(muteUnmuteOptions) {
      if (!this.room || !this.room.localParticipant) {
        throw new Error("You must be connected to a room to mute tracks.");
      }
      if (muteUnmuteOptions.video) {
        this.room.localParticipant.videoTracks.forEach((publication) => {
          publication.track.disable();
        });
      }
    },
    unmute(muteUnmuteOptions) {
      if (!this.room || !this.room.localParticipant) {
        throw new Error("You must be connected to a room to unmute tracks.");
      }
      if (muteUnmuteOptions.video) {
        this.room.localParticipant.videoTracks.forEach((publication) => {
          publication.track.enable();
        });
      }
    },
    onMuteUnmuteClick(trackType) {
      if (trackType === this.TrackType.Video) {
        const options = { audio: false, video: true };
        if (this.isVideoMuted) {
          // console.log("ปิดเสียงวิดิโอ")
          this.unmute(options); // เปิดเสียง
        } else {
          // console.log("เปิดเสียงวิดิโอ")
          this.mute(options); // ปิดเสียง
        }
        this.isVideoMuted = !this.isVideoMuted
      }
    },
    // ออกจากห้อง
    onDisconected(){
        this.room.localParticipant.tracks.forEach(publication => {
          const attachedElements = publication.track.detach();
          attachedElements.forEach(element => element.remove());
        });
        this.room.disconnect();
    },

  }
}
    
</script>
<style scoped lang="scss">
.card-option {
  background: rgb(245 245 245 / 70%);
  border-radius: 16px;
  box-shadow: 0 4px 30px rgb(0 0 0 / 10%);
  backdrop-filter: blur(0px);
  position: absolute;
  padding: 0.5rem 1rem;
  margin: 1rem;
  width: -webkit-fill-available;
  bottom: 0px;
  z-index: 9999;
}

.ciclie-icon-phone {
  width: 52px;
  height: 52px;
  background-color: #EA5455;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50px;
  margin-left: 2rem;
}

.ciclie-icon {
  width: 44px;
  height: 44px;
  background-color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50px;
  margin-left: 2rem;
}

.icon-phone {
  transform: rotate(225deg);
  font-size: 20px;
  color: #fff;
}

.icon-custom {
  font-size: 16px;
  color: #6E6B7B;
}

.disease {
  color: #6e6b7b;
  border: 1px solid #A8A8A8;
  border-radius: 6px;
  padding: 2px 12px;
  font-size: 12px;
  margin-left: 1rem;
}

.card-time {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 6px 14px;
  background-color: #fff;
  border-radius: 30px;
  font-size: 14px;
}
</style>
