<template>
    <div>
        <el-button type="info" @click="startFfmpeg" icon="el-icon-star-on" />
        <el-button type="info" @click="startReceivingVideoAudio" icon="el-icon-star-on" />

        <h2>Feature1: The video resolution is changeable in runtime</h2>
        <el-button type="success" v-show="showPlayButton" @click="playAudio" icon="el-icon-service" circle/>
        <h2>Feature2: Audio transport, if you click the audio button, you will sound 'Gong, Gong, Gong...'</h2>
        <el-button type="success" @click="testDataChannel">Test Data Channel, Send A PING</el-button>
        <h2>Feature3: Data channel, click test button to test data channel communication</h2>
        <el-button type="success" @click="changeBandWidth">Change Bandwidth</el-button>
        <h2>Feature4: Change bandwidth</h2>
        <video id="video" controls muted autoplay/>
        <audio id="audio" autoplay style="width: 0;height: 0;"></audio>
    </div>
</template>
<script>
  import adapter from 'webrtc-adapter'
  window.webrtcAdapter = adapter
  export default {
    name: 'UseDevice',
    beforeRouteLeave(to, from, next) {
      this.$socket.emit('stopUse')
      next()
    },
    data() {
      return {
        supportAudio: true,
        peerConnect: undefined,
        targetSideDataChannel: undefined,
        sdpConstraints: {
          mandatory: {
            OfferToReceiveAudio: true,
            OfferToReceiveVideo: true
          }
        },
        showPlayButton: false
      }
    },
    sockets: {
      connect: function () {
        this.beginUse()
      },
      status: function () {
        this.generatePeerConnection()
      },
      offerSdp: function (sdp) {
        this.peerConnect.setRemoteDescription(new RTCSessionDescription(sdp))
        this.sdpConstraints.mandatory.OfferToReceiveAudio = this.supportAudio
        this.peerConnect.createAnswer((description) => {
          this.peerConnect.setLocalDescription(description)
          this.$socket.emit('answerSdp', description)
        }, () => {
        }, this.sdpConstraints)
      },
      onCandidate: function (candidate) {
        this.peerConnect.addIceCandidate(new RTCIceCandidate(candidate))
      }
    },
    mounted() {
      if (this.$store.getters.isConnected) {
        this.beginUse()
      }
    },
    methods: {
      beginUse() {
        this.$socket.emit('beginUse')
      },
      startFfmpeg() {
        this.$socket.emit('startFfmpeg')
      },
      startReceivingVideoAudio() {
          this.$socket.emit('startReceivingVideoAudio')
          this.$socket.emit('beginWebRtc')
      },
      changeBandWidth() {
        this.$message('bitrate change to: 100000')
        this.$socket.emit('changeBandwidth', 100000)
      },
      testDataChannel() {
        this.targetSideDataChannel.send('ping')
      },
      generatePeerConnection() {
        if (this.peerConnect) {
          this.peerConnect.close()
        }
        this.peerConnect = new RTCPeerConnection({
          iceServers: [
            {
              urls: ['turn:127.0.0.1:3478'],
              username: 'guest',
              credential: 'somepassword'
            }
          ],
          iceTransportPolicy: this.$route.query.relay === 'true' ? 'relay' : 'all',
          iceCandidatePoolSize: '0'
        })
        this.peerConnect.onicecandidate = this.onIceCandidate
        let mySideDataChannel = this.peerConnect.createDataChannel('test')
        mySideDataChannel.onmessage = (e) => {
          this.$message('received a data channel data: ' + e.data)
        }
        this.peerConnect.ondatachannel = (e) => {
          this.targetSideDataChannel = e.channel
        }
        this.peerConnect.ontrack = (e) => {
          const track = e.track
          const stream = e.streams[0]
          if (track.kind === 'video') {
            const video = document.getElementById('video')
            video.muted = true
            video.autoplay = true
            try {
              video.srcObject = stream
            } catch (error) {
              video.srcObject = stream
            }
            video.play().catch(e => {
              console.log(e)
            })
          }
          if (track.kind === 'audio') {
            const audio = document.getElementById('audio')
            try {
              audio.srcObject = stream
            } catch (error) {
              audio.src = URL.createObjectURL(stream)
            }
            audio.play().catch(() => {
              this.showPlayButton = true
            })
          }
        }
      },
      playAudio () {
        const audio = document.getElementById('audio')
        audio.play().then(() => {
          this.showPlayButton = false
        }).catch(() => {
          this.showPlayButton = true
        })
      },
      onIceCandidate(event) {
        if (event && event.candidate) {
          this.$socket.emit('onCandidate', event.candidate)
        }
      }
    }
  }
</script>