<template>
  <div class="Korero">
    <div class="header">
      <h4>Kōrero Māori Word Count (Demo))</h4>
      <button id="startVAD" v-bind:class="{ vad: vadOn, on: recordingOn, 'loading': loadRecording}">
        <i class="fa fa-microphone" ></i>
        <i class="fa fa-stop" ></i>
        <i class="fa fa-spinner fa-spin"></i>
      </button>
      <div id="vadStatus" v-bind:class="{ active: vadOn, 'mobile': isMobile()}">
        <div>
          <canvas id="canvas" v-if="!isMobile()"></canvas>
          <span v-if="isMobile() && vadOn && recordingOn">Voice Detected</span>
          <span v-if="isMobile() && recordingOn && !vadOn">Listening</span>
          <span v-if="isMobile()">Hit Record</span>
        </div>
      </div>
   

    <div id="te_reo_sensivity"  class="sensitivity_box">
        Sensitivity:
        <input
          type="range"
          min="0"
          v-bind:max="sensitivity_max"
          v-bind:value="te_reo_sensivity"
          v-on:input="setTeReoSensivity($event)">
          {{ te_reo_sensivity }}
      </div>
<!--       <div id="vad_sensitivity" class="sensitivity_box">
        Voice Activity Sensitivity:
        <input
          type="range"
          min="0"
          v-bind:max="sensitivity_max"
          v-bind:value="vad_sensitivity"
          v-on:input="setVadSensivity($event)">
          {{ vad_sensitivity }}
      </div> -->
  </div>

    <div class="body">
      <div id="transcriptions">
        <div v-for="(item, index) in transcriptions" class='transcription' v-bind:class="[item.status]" v-if="item.status != 'Failed'"> 
          <button class="delete" v-if="item.status != 'Transcribing'" v-on:click="deleteObject(index)"><i class="fa fa-times"></i></button>
          <div class='text'>
            <i class="fa fa-spinner fa-spin" v-bind:class="[item.status]" v-if="item.status == 'Transcribing'" ></i>
          
            <div v-if="item.status != 'Transcribing'">{{item.display_count}}</div>
        
          </div>
        </div>
      </div>
      
    </div>

    <div class="footer">
      <div class='status'> {{status}}</div>
    </div>
  </div>
</template>

<script>
import recorder from '../lib/streaming_recorder.js';
const ApiAuth = require('../../api_auth');
const api_auth = ApiAuth.api_auth;
// import 'ws-audio-api';
// const streamer = WSAudioAPI.Streamer();
//const axios = require('axios');
export default {
  name: 'Kōrero',
  data () {
    return {

      vadOn: false,
      recordingOn: false,
      recorder: null,
      buttonText: 'Start',
      transcriptions: [],
      icon: 'fa-microphone',
      loadRecording: false,
      te_reo_sensivity: 85,
      vad_sensitivity: 10,
      sensitivity_max: 100,
      status: '',
      totalCount : 0
    }
  },
  methods: {
    deleteObject: function (e) {
      this.transcriptions.splice(e, 1)
    },
    stopRecording: function () {
      try{this.recorder.stop();}
      catch(e){}
      this.recorder=null;
      this.vadOn = false;
      this.recordingOn = false
    },   
    isMobile() {
      if(/Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Mobile|SamsungBrowser|Opera Mini/i.test(navigator.userAgent)) {
        console.log(navigator.userAgent)
        return true
      } else {
        return false
      }
    },
    setTeReoSensivity: function(e) {
      this.te_reo_sensivity=e.target.value;
      if (this.sensitivity_sock.readyState == 1)  {
        var tmp = parseInt(this.te_reo_sensivity)
        var sensitivity = "" + tmp / this.sensitivity_max;
        // if the server received 'Text' that can be 
        // interpreted as a float, it assumes its sensititvy 
        this.sensitivity_sock.send(sensitivity);
      }

    },
    setVadSensivity: function(e) {
      this.vad_sensitivity=e.target.value;
    },
    processResult: function(text) {
      var current = this.transcriptions[0];
      if (current) {
        if (text=='EOS') {
          current.total_count = current.total_count + current.active_count;
          current.active_count = 0;
        }
        else {  
          current.active_count = parseInt(text);
          current.status = 'Success';
        }

        this.animateCount();
      }
    },
    animateCount: function() {
      var current = this.transcriptions[0];
      if (current) {
          if (current.display_count == current.total_count) {
            current.active=false;
            return;
          }

          if (current.display_count < current.total_count) {
            current.display_count += 1;
            current.active=true;
          }

          if (current.display_count > current.display_count) {
            current.display_count -= 1;
            current.active=true;
          }

          setTimeout(this.animateCount, 150);
      }
    },

    initWebSocket: function() {

        var socket_url = process.env.ASR_WEBSOCKET_ENDPOINT + "/word_count";
        this.status = "Connecting to " + socket_url;
        this.socket = new WebSocket(socket_url);
        this.socket.binaryType = 'arraybuffer';
        var self = this;
        this.socket.onmessage = function(message) {
          //console.log(message);
          self.status = "Connected to  " + socket_url;
          if (message.isTrusted) {
              self.processResult(message.data);
          }
        };
        this.socket.onclose = function () {
          self.status = "Last connected " + socket_url;
          console.log('Connection to server closed');
        };
        this.socket.onerror = function (err) {
           self.status = "Error getting data " + err;
           console.log('Getting audio data error:', err);
        };
    },

    ensureWebSocket: function() {
      self = this;
      if (this.socket == null) {
        this.initWebSocket();
      }
      else {
        var readyState = this.socket.readyState;

        if (readyState == WebSocket.CLOSED) {
          this.initWebSocket();
        }

        if (readyState == WebSocket.CLOSING) {
          this.socket.onclose = function () {
            self.initWebSocket();
          }
        } 
        // otherwise, if WebSocket.CONNECTING || WebSocket.OPEN 
        // just let nature take its course
      }
    },

    initSensitivitySocket: function() {

        var socket_url = process.env.ASR_WEBSOCKET_ENDPOINT + "/word_count_sensitivity"
        console.log( "Connecting to " + socket_url);
        this.sensitivity_sock = new WebSocket(socket_url);
        this.sensitivity_sock.binaryType = 'arraybuffer';
        this.sensitivity_sock.onmessage = function(message) {
          console.log("Connected to  " + socket_url);
        };
    }

  },
  mounted: function () {

    this.initSensitivitySocket();
    startVAD.onclick = event => {
      if (this.recorder==null) {
        this.loadRecording = true;
        var transcription = { 
          status: 'Transcribing',
          display_count: 0, 
          total_count: 0,
          active_count: 0
        }
        this.transcriptions.unshift(transcription)

        this.initWebSocket();

        this.recorder = new recorder({
         
          processAudio: (buffer) => {
            if (this.socket.readyState == 1)  {
              this.socket.send(buffer);
            }
          },

          afterRecording: (stream) => {
            if (this.socket.readyState == 1)  {
              this.socket.send('EOS');
            }
          },

          pauseRecording: () => {
            streamer.stop();
            console.log('paused')
          },

          micFailed: () => {
            console.log('failed')
          },
          
          voiceStop: () => {
            this.vadOn = false
          },

          voiceStart: () => {
            this.vadOn = true
            this.ensureWebSocket()
          },

          canvasID: this.isMobile() ? null : 'canvas',
          bitRate         : 64,
          sampleRate      : 44100,
          // format          : this.format
        })
      }

      if (this.recordingOn) {
        this.recorder.stop();
        this.recorder=null;
        this.vadOn = false;
        this.recordingOn = false
        // this.buttonText = 'Start'
      } else {
        this.recorder.start()
        this.recordingOn = true
        this.buttonText = 'Stop'
        this.loadRecording = false
      }

    }
  }
}


</script>

<style scoped>

.Korero {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  font-size: 14px;
}
button.delete{
  position: absolute;
  right: 0px;
  top: 0px;
  display: block;
  border: 0px;
  margin: 2px;
  font-size: 0.7em;
  color: #696969;
  cursor: pointer;
  outline: none;
  background: none;
}
.transcription{
  position: relative;
  width: 100%;
  margin: 20px 30px 20px 30px;
  max-width: 500px;
  /*margin-top: 20px; */
  font-size: 1.5em;
  border-radius: 8px;
  /*box-shadow: 0px 2px 6px -2px rgba(0,0,0,1)  */
  border: 1px solid black;
}
.transcription.Transcribing{
  border: 0px;
}
.transcription .text {
  padding: 15px 20px;
  font-size: 4em;
}
.transcription:nth-of-type(1){
  margin-top: 40px;
}

.transcription .active {
  color: lightcoral;
}

div.transcription.Success div.text [class*=fa]{
  display: none;
}
.Korero, .header, .body{
  display: flex;
  flex-direction: column;
  justify-content: center;
}
.body{
  height: 100vh;
  margin-top: -225px;
}
.header{
  padding-bottom: 15px;
  z-index: 1;
  background-color: white;
  padding-left: 15px;
  padding-right: 15px;  
}
.footer .status {
  margin-top: -15px;
  margin-bottom: 15px;
  font-style: italic;
}

#transcriptions{
  z-index: 0;
  display: flex;
  flex-direction: column;
  overflow-y: scroll;
  flex-wrap: nowrap;
  -webkit-overflow-scrolling: touch;
  -ms-overflow-style: -ms-autohiding-scrollbar; 
  align-items: center;  
  margin-top: 232px;
  height: 100vh;    
  padding-left: 15px;
  padding-right: 15px;
}

#vadStatus{
  display: flex;
  width: 100%;
  max-width: 500px;  
  height: 100px;
  align-self: center;
  align-items: center;
}
#vadStatus canvas{
  z-index: 1;
  display: block;
  height: 100%;
  width: 100%;
  background-color: #333;
}
#vadStatus div{
  height: 100%;
  width: 100%;
  border: 4px solid black;
  border-radius: 4px;
}
#vadStatus.active div{
  border-color: red;
}
#vadStatus.mobile{
  height: 28px;
  max-width: 150px;  
}
#vadStatus.mobile div span{
  display: block;
  padding: 4px;
}
#vadStatus.mobile div{
  color: white;
  font-weight: 700;
  background-color: black;
}
#vadStatus.mobile.active div{
  background-color: red;
}
#startVAD {
  z-index: 2;
  align-self: center;
  margin: 15px;
  outline: none;
  background-color: rgb(40,40,40);
  color: white;
  width: 80px;
  height: 80px;
  border: 2px solid black;
  border-radius: 40px;
  padding: 8px 16px;
  cursor: pointer;
}
#startVAD [class*=fa]{
  font-size: 34px !important;
}
#startVAD.on {
  background-color: rgba(255, 40, 40, 0.75);
  border-color: rgb(200, 40, 40);
}
#startVAD.on.vad {
  background-color: rgba(255, 40, 40, 1);
}
#startVAD.on .fa-microphone{
  display: none;
}
#startVAD .fa-stop, #startVAD .fa-spinner{
  display: none;
}
#startVAD.on .fa-stop{
  display: initial;
}
#startVAD.on.loading .fa-spinner{
  display: initial;
}
audio{
  outline: none;
  width: 100%;
  padding-bottom: 15px;
}
.audio{
  padding: 0px 15px 0px 15px;
}
div.sensitivity_box {
  padding: 5px;
  margin: 5px 0 10px 0;
  align-self: center;
  text-align: right;
}
div.sensitivity_box input {
  width: 200px;
}

div.confidence{
  display: inline-flex;
  flex-wrap: wrap;
}
div.char{
  /*line-height: 20px;*/
}
div.char-wrapper{
  width: 20px;
  display: inline-block;
  text-align: center;
}
div.word{
  margin-right: 20px;
}
div.word:last-child{
  margin-right: 0px;
}
div.word-wrapper{
  /*width: 20px*/
}
span.char{
  display: inline-block;
  width: 20px;
  text-align: center;
}
span.prob{
  display: inline-block;
  font-size: 8px;
  /*line-he*/ight: 0px;
}
</style>
