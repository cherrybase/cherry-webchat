<template>
  <div class="sc-message--text" :style="messageColors">
    <template>
      <div class="sc-message--toolbox" :style="{background: messageColors.backgroundColor}">
        <button v-if="showEdition && me && message.id" :disabled="isEditing" @click="edit">
          <IconBase :color="isEditing ? 'black' : messageColors.color" width="10" icon-name="edit">
            <IconEdit />
          </IconBase>
        </button>
        <button v-if="showDeletion && me && message.id" @click="$emit('remove')">
          <IconBase :color="messageColors.color" width="10" icon-name="remove">
            <IconCross />
          </IconBase>
        </button>
        <slot name="text-message-toolbox" :message="message" :me="me"> </slot>
      </div>
    </template>
    <slot :message="message" :messageText="messageText" :messageColors="messageColors" :me="me">
      <p class="sc-message--text-content" v-html="messageText"></p>
      <p v-if="message.data.meta" class="sc-message--meta" :style="{color: messageColors.color}">
        {{ message.data.meta }}
      </p>
      <p v-if="message.isEdited" class="sc-message--edited">
        <IconBase width="10" icon-name="edited">
          <IconEdit />
        </IconBase>
        edited
      </p>
    </slot>
  </div>
</template>

<script>
import {mapState} from '../store'
import IconBase from './../components/IconBase.vue'
import IconEdit from './../components/icons/IconEdit.vue'
import IconCross from './../components/icons/IconCross.vue'
import escapeGoat from 'escape-goat'
import Autolinker from 'autolinker'
import store from '../store/'
import debounce from "debounce";

const fmt = require('msgdown')

const playBackgroundVoice = debounce((message, source) => {
  let attachment = message.data?.attachments?.find(a => a.mediaSubType === "BG_VOICE")
  if(!attachment) return;

  let id = 'bg_voice_';
  let voicePlayIcon = "fa-volume-off" // "fa-play"
  let voiceStopIcon = "fa-volume-up" // "fa-stop"
  let bg_voice = document.getElementById(id);
  let bg_voice_control = document.getElementById("bg_voice_control");

  if(bg_voice){
    // console.log("playBackgroundVoice coming from "+source+" "+message.id, bg_voice);
    // bg_voice.src = 'https://www.bensound.com/bensound-music/bensound-moose.mp3';
    bg_voice.src = attachment.mediaURL;
  }else{
    // console.log("playBackgroundVoice coming from "+source+" "+message.id, bg_voice);
    bg_voice = document.createElement('audio');
    bg_voice.id = id;
    // bg_voice.src = 'https://www.bensound.com/bensound-music/bensound-moose.mp3';
    bg_voice.src = attachment.mediaURL;
    document.body.appendChild(bg_voice);
  }

  bg_voice.volume = 0.2;
  bg_voice.play();

  bg_voice_control.classList.remove(voicePlayIcon)
  bg_voice_control.classList.add(voiceStopIcon)

  bg_voice.onended = function () {
    bg_voice_control.classList.remove(voiceStopIcon)
    bg_voice_control.classList.add(voicePlayIcon)
  }
}, 500)

export default {
  components: {
    IconBase,
    IconCross,
    IconEdit
  },
  props: {
    message: {
      type: Object,
      required: true
    },
    messageColors: {
      type: Object,
      required: true
    },
    messageStyling: {
      type: Boolean,
      required: true
    }
  },
  computed: {
    messageText() {
        if(this.message.data.text){
            const escaped = escapeGoat.escape(this.message.data.text)
            //console.log("escaped",escaped,this.message.data.text);
            return Autolinker.link(this.messageStyling ? fmt(escaped, {italic: {delmiter: '///', tag: 'em'}}) : escaped, {
                className: 'chatLink',
                truncate: {length: 50, location: 'smart'}
            })
        }
        return null;
    },
    me() {
      return this.message.author === 'me'
    },
    isEditing() {
      return (store.state.editMessage && store.state.editMessage.id) === this.message.id
    },
    ...mapState(['showDeletion', 'showEdition'])
  },
  methods: {
    edit() {
      store.setState('editMessage', this.message)
    }
  },
  mounted(){
    if(this.message.id && this.message.isLast && this.message.author !== "me"){
      playBackgroundVoice(this.message, "mounted")
    }
  },
  watch: {
    'message': {
      handler (val, oldVal) {
        if(val.id && val.isLast && val.author !== "me"){
          playBackgroundVoice(val, "watch")
        }else{
          let bg_voice = document.getElementById('bg_voice_')
          if(bg_voice)
            bg_voice.src = ""
        }
      }
    }
  },
}
</script>

<style scoped lang="scss">
.sc-message--text {
  padding: 10px;
  border-radius: 10px;
  font-weight: 300;
  font-size: 14px;
  line-height: 16px;
  position: relative;
  -webkit-font-smoothing: subpixel-antialiased;
  .sc-message--text-content {
    white-space: pre-wrap;
    margin: 0;
    padding: 0;
  }
  
  ul,li{
    margin: 0;
    padding: 0;
    list-style: none;
  }

  ul{
    padding:10px;
  }

  &:hover .sc-message--toolbox {
    left: -20px;
    opacity: 1;
  }
  .sc-message--toolbox {
    transition: left 0.2s ease-out 0s;
    white-space: normal;
    opacity: 0;
    position: absolute;
    left: 0px;
    width: 25px;
    top: 0;
    button {
      background: none;
      border: none;
      padding: 0px;
      margin: 0px;
      outline: none;
      width: 100%;
      text-align: center;
      cursor: pointer;
      &:focus {
        outline: none;
      }
    }
  }
  code {
    font-family: 'Courier New', Courier, monospace !important;
  }
}

.sc-message--content.sent .sc-message--text {
  color: white;
  background-color: #4e8cff;
  max-width: calc(100% - 120px);
  word-wrap: break-word;
}
@media screen and (max-width: 479px){
    .sc-message--content.sent .sc-message--text {
        max-width: 100%;
    }
}
.sc-message--content.received .sc-message--text {
  color: #263238;
  background-color: #f4f7f9;
  margin-right: 40px;
}

a.chatLink {
  color: inherit !important;
}
</style>
<style>
a.chatLink {
  color: inherit !important;
}
.sc-message--text .sc-message--toolbox /deep/ svg {
      margin-left: 5px;
}
</style>
