<template>
  <div class="messages">
    <div id='sms_contact'>{{display}}</div>

    <div id='sms_list'>
        <div class="sms" v-bind:class="{ select: key === selectMessage}" v-for='(mess, key) in messagesList' v-bind:key="mess.id">
            <span class='sms_message sms_me' 
              v-bind:class="{ sms_other : mess.owner === 0}">
                {{mess.message}}
                <span class="sms_time"><timeago :since='mess.time' :auto-update="20"></timeago></span>
            </span>
        </div>
    </div>

    <div id='sms_write'>
        <input type="text" placeholder="Envoyer un message">
        <span class='sms_send'>></span>
    </div>
  </div>
</template>

<script>
import { mapGetters, mapActions } from 'vuex'
import Modal from '@/components/Modal/index.js'

export default {
  data () {
    return {
      ignoreControls: false,
      selectMessage: -1,
      display: '',
      phoneNumber: ''
    }
  },
  methods: {
    ...mapActions(['setMessageRead', 'sendMessage', 'deleteMessage']),
    resetScroll: function () {
      this.$nextTick(() => {
        let elem = document.querySelector('#sms_list')
        elem.scrollTop = elem.scrollHeight
        this.selectMessage = -1
      })
    },
    scrollIntoViewIfNeeded: function () {
      this.$nextTick(() => {
        const elem = this.$el.querySelector('.select')
        if (elem !== null) {
          elem.scrollIntoViewIfNeeded()
        }
      })
    },
    onUp: function () {
      if (this.ignoreControls === true) return
      if (this.selectMessage === -1) {
        this.selectMessage = this.messagesList.length - 1
      } else {
        this.selectMessage = this.selectMessage === 0 ? 0 : this.selectMessage - 1
      }
      this.scrollIntoViewIfNeeded()
    },
    onDown: function () {
      if (this.ignoreControls === true) return
      if (this.selectMessage === -1) {
        this.selectMessage = this.messages.length - 1
      } else {
        this.selectMessage = this.selectMessage === this.messagesList.length - 1 ? this.selectMessage : this.selectMessage + 1
      }
      this.scrollIntoViewIfNeeded()
    },
    onEnter: function () {
      if (this.ignoreControls === true) return
      if (this.selectMessage !== -1) {
        this.onActionMessage()
      } else {
        this.$phoneAPI.getReponseText().then(data => {
          let message = data.text.trim()
          if (message !== '') {
            this.sendMessage({
              phoneNumber: this.phoneNumber,
              message
            })
          }
        })
      }
    },
    onActionMessage: function () {
      let message = this.messagesList[this.selectMessage]
      let isGPS = /^GPS: -?\d*(\.\d+), -?\d*(\.\d+)/.test(message.message)
      let choix = [{
        title: 'Effacer',
        icons: 'fa-circle-o'
      }, {
        title: 'Annuler',
        icons: 'fa-undo'
      }]
      if (isGPS === true) {
        choix = [{
          title: 'Position GPS',
          icons: 'fa-location-arrow'
        }, ...choix]
      }
      this.ignoreControls = true
      Modal.CreateModal({choix}).then(data => {
        if (data.title === 'Effacer') {
          this.deleteMessage({ id: message.id })
        } else if (data.title === 'Position GPS') {
          let val = message.message.match(/((-?)\d+(\.\d+))/g)
          this.$phoneAPI.setGPS(val[0], val[1])
        }
        this.ignoreControls = false
        this.selectMessage = -1
      })
    },
    onBackspace: function () {
      if (this.ignoreControls === true) return
      if (this.selectMessage !== -1) {
        this.selectMessage = -1
      } else {
        this.$router.push({path: '/messages'})
      }
    },
    onRight: function () {
      if (this.ignoreControls === true) return
      if (this.selectMessage === -1) {
        this.ignoreControls = true
        Modal.CreateModal({choix: [
          {title: 'Envoyer Coord GPS', icons: 'fa-location-arrow'},
          {title: 'Annuler', icons: 'fa-undo'}
        ]}).then(data => {
          if (data.title === 'Envoyer Coord GPS') {
            this.sendMessage({
              phoneNumber: this.phoneNumber,
              message: '%pos%'
            })
          }
          this.ignoreControls = false
        })
      }
    }
  },
  computed: {
    ...mapGetters(['messages']),
    messagesList () {
      return this.messages.filter(e => e.transmitter === this.phoneNumber)
    }
  },
  watch: {
    messagesList () {
      this.setMessageRead(this.phoneNumber)
      this.resetScroll()
    }
  },
  created: function () {
    this.display = this.$route.params.display
    this.phoneNumber = this.$route.params.number
    this.$bus.$on('keyUpArrowDown', this.onDown)
    this.$bus.$on('keyUpArrowUp', this.onUp)
    this.$bus.$on('keyUpEnter', this.onEnter)
    this.$bus.$on('keyUpBackspace', this.onBackspace)
    this.$bus.$on('keyUpArrowRight', this.onRight)
  },
  beforeDestroy: function () {
    this.$bus.$off('keyUpArrowDown', this.onDown)
    this.$bus.$off('keyUpArrowUp', this.onUp)
    this.$bus.$off('keyUpEnter', this.onEnter)
    this.$bus.$off('keyUpArrowRight', this.onRight)
    this.$bus.$off('keyUpBackspace', this.onBackspace)
  }
}
</script>

<style scoped>
.messages{
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    height: calc(100% - 20px);
    background-color: #DDD;    
}
#sms_contact{
    background-color: #4CAF50;
    color: white;
    height: 34px;
    line-height: 34px;
    padding-left: 5px;
}
#sms_list{
    height: calc(100% - 34px - 26px);
    overflow-y: auto;
    padding-bottom: 8px;
}

.sms_send{
    width: 20px;
    height: 20px;
    float: right;
    border-radius: 50%;
    background-color: orange;
    margin-right: 6px;
    margin-bottom: 2px;
    text-align: center;
    color: white;
    line-height: 20px;
}

.sms{
  overflow: auto;
  zoom: 1;

}

.sms_me{
    float: right;
    background-color: white;
    padding: 5px 10px;
    max-width: 90%;
    margin-right: 5%;
    margin-top: 10px;
}

.sms_other{
    background-color: #4CAF50;
    color:white;
    float: left;
    padding: 5px 10px;
    max-width: 90%;
    margin-left: 5%;
    margin-top: 10px;
}


.sms_time{
    display: block;
    font-size: 0.5em;
}

.sms_me .sms_time{
    color: #AAA;
}
.sms_other .sms_time{
    color: #555;
}
.messages{
  position: relative;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
}
.sms.select .sms_message{
  background-color: lightskyblue;
}
.sms_message{
  word-wrap: break-word;
  max-width: 80%;
}
#sms_write{
    height: 26px;
    widows: 100%;
}
#sms_write input{
    width: 78%;
    margin-left: 6%;
    border: none;
    outline: none;
    font-size: 0.6em;
    padding: 3px 5px;
    float: left;
}
.sms_send{
    width: 20px;
    height: 20px;
    float: right;
    border-radius: 50%;
    background-color: orange;
    margin-right: 6px;
    margin-bottom: 2px;
    text-align: center;
    color: white;
    line-height: 20px;
}
</style>
