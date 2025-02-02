<template>

  <div class="d-flex flex-column flex-grow-1">

    <div class="row" style="min-height: 35vh; max-height: 200px; overflow-y: auto;" ref="messageBox">
        <div class="col-12" v-for="message in orderedMessages">
          <div v-if="message.sender_name === 'blinest'">
            <small>{{ message.message }}</small>
          </div>
          <div v-else class="message">
            <div class="row" v-if="!muted.includes(message.sender_id)">
              <div class="col-auto d-none d-md-block pr-0">
                <span v-if="message.sender_profile_picture" class="avatar" :style="'background-image: url(/images/players/' + message.sender_profile_picture + ');'">
                </span>
                <span v-else class="avatar" :style="'background-color:' + $options.filters.stringToColour(message.sender_name) + ';'">
                 {{ message.sender_name | initialize }}
                </span>
              </div>
              <div class="col pl-0">
                <small><b>{{ message.sender_name }}</b> {{ message.created_at | moment("calendar") }}</small>
                <p class="message-text">
                  <span class="message-votes">
                    <i @click="rateDown(message.id)" class="text-danger fas fa-thumbs-down pointer" title="Signaler ce message"></i>
                    <i @click="mute(message.sender_id)" class="text-danger fas fa-volume-mute pointer" title="Mute (les messages de cet utilisateur n'apparaîtront plus.)"></i>
                    <template v-for="moderator in game.moderators" v-if="moderator.id === game.currentUser.id">
                      <i @click="blockUser(message.sender_id, message.sender_name)" class="text-danger fas fa-ban pointer" title="Bloquer l'utilisateur pour 24 heures"></i>
                      <i @click="deleteMessage(message.id)" class="text-danger fas fa-trash pointer" title="Supprimer ce message"></i>
                    </template>
                  </span>
                  <span v-html="$options.filters.stringParser(message.message)"></span>
                </p>
              </div>
            </div>
          </div>
        </div>
    </div>
    <div class="row">
        <div class="col-12">
            <div class="input-group mt-3">
                <input type="text"
                        id="chatInput"
                        ref="textarea"
                        maxlength="255"
                        class="form-control"
                        placeholder=""
                        v-on:keyup.enter="sendMessage"
                        aria-label="New message"
                        aria-describedby="button-addon2"
                        v-model="newMessage"
                        v-on:click="hideEmojiPicker">
                <div class="input-group-append">
                    <button class="btn btn-secondary" :class="{ 'triggered': showEmojiPicker }" @mousedown.prevent="toggleEmojiPicker">
                        <i class="far fa-smile-wink"></i>
                    </button>
                </div>

                <picker
                  v-show="showEmojiPicker"
                  :showPreview="false"
                  :showSearch="false"
                  :sheetSize="32"
                  :i18n="i18n"
                  @select="addEmoji"
                />

            </div>
        </div>
    </div>

  </div>

</template>

<script>

  import { Picker } from 'emoji-mart-vue'
  const badWords = require('leo-profanity');
  badWords.loadDictionary('fr');

  export default {

    name: 'ChatApplication',

    props: ['game'],

    components: { Picker },

    data: () => {
      return {
        users: [],
        muted: [],
        messages: [],
        chatOpen: false,
        chatUserID: null,
        loadingMessages: false,
        newMessage: '',
        showEmojiPicker: false,
        i18n: {
          search: 'Recherche',
          notfound: 'Aucune Emoji trouvé',
          categories: {
            search: 'Résultats',
            recent: 'Favoris',
            people: 'Smileys & Personnes',
            nature: 'Animaux & Nature',
            foods: 'Nourriture & Boisson',
            activity: 'Activités',
            places: 'Voyage & Lieux',
            objects: 'Objets',
            symbols: 'Symboles',
            flags: 'Drapeaux',
            custom: 'Personnalisés',
          }
        }

      }
    },
    created () {
      let app = this
      app.loadMessages()
    },
    mounted() {
      this.listenForNewMessage();
    },
    updated() {
      let element = this.$refs.messageBox;
      element.scrollTop = element.scrollHeight;
    },
    methods: {

      mute(user_id) {
        this.muted.push(user_id);
      },

      toggleEmojiPicker () {
        this.showEmojiPicker = !this.showEmojiPicker
      },

      hideEmojiPicker() {
        if (this.showEmojiPicker)
        this.showEmojiPicker = false;
      },

      addEmoji (emoji) {
        const textarea = this.$refs.textarea;
        const cursorPosition = textarea.selectionEnd
        const start = this.newMessage.substring(0, textarea.selectionStart)
        const end = this.newMessage.substring(textarea.selectionStart)
        const text = start + emoji.native + end
        this.newMessage = text;
        textarea.focus();
        this.toggleEmojiPicker()
      },

      newline() {
        this.value = `${this.value}\n`;
      },

      listenForNewMessage() {
        Echo.channel('chat-' + this.game.id)
          .listen('MessageSent', (data) => {
            $("#chatInput").attr("disabled", false);
            this.messages.push(data.message)
          })
          .listen('MessageDelete', (data) => {
            console.log(data);
            this.messages.splice(this.messages.findIndex(f => f.id === data.message.id), 1);
          })
      },

      loadMessages () {
        let app = this
        app.loadingMessages = true
        app.messages = []
        axios.post('/messages', {
          game_id: app.game.id
        }).then((resp) => {
          app.messages = resp.data
          app.loadingMessages = false
        })
      },

      sendMessage (blinest) {
        let app = this;
        $("#chatInput").attr("disabled", true);
        if (app.newMessage !== '') {
          axios.post('/messages/send', {
            bot: (blinest == true) ? true : false,
            game_id: app.game.id,
            message: app.newMessage
          }).then((resp) => {
            //app.messages.push(resp.data)
            app.newMessage = ''
          })
        }
      },

      rateDown(message_id) {
        axios.post('/messages/vote', {
          message_id: message_id,
          type: 'report'
        }).then((resp) => {
          console.log(resp.data);
        })
      },

      deleteMessage(message_id) {
        axios.post('/messages/delete', {
          message_id: message_id,
        }).then((resp) => {
          console.log("Message supprimé");
        })
      },

      blockUser(user_id, user_name) {
        if (window.confirm("Attention, si tu confirme, l'utilisateur sera banni pendant une heure. Les seuls motifs pour bannir un utilisateur sont le non respect de la loi, les propos haineux, le harcelement, les menaces ou l'usurpation d'identité.")) {
          axios.get('/moderator/user/' + user_id + '/block').then((resp) => {
            //console.log(resp.data);
            this.newMessage = "L'utilisateur " + user_name + " a été banni.";
            this.sendMessage(true);
          });
        }
      }

    },

    computed: {
      orderedMessages: function () {
        return _.orderBy(this.messages, 'created_at', 'asc')
      }
    },

    filters: {

      stringParser: function(text) {
          var urlRegex = /(\b(https?|ftp|file):\/\/[-A-Z0-9+&@#\/%?=~_|!:,.;]*[-A-Z0-9+&@#\/%=~_|])/ig;
          var niceText = badWords.clean(text);
          return niceText.replace(urlRegex, function(url) {
              return '<a href="' + url + '" target="_blank">' + url + '</a>';
          });
      },

      initialize: function(value) {
        let rgx = new RegExp(/(\p{L}{1})\p{L}+/, 'gu');
        let initials = [...value.matchAll(rgx)] || [];

        initials = (
          (initials.shift()?.[1] || '') + (initials.pop()?.[1] || '')
        ).toUpperCase();

        return initials;
      },

      stringToColour: function(str) {
        var hash = 0;
        for (var i = 0; i < str.length; i++) {
          hash = str.charCodeAt(i) + ((hash << 5) - hash);
        }
        var colour = '#';
        for (var i = 0; i < 3; i++) {
          var value = (hash >> (i * 8)) & 0xFF;
          colour += ('00' + value.toString(16)).substr(-2);
        }
        return colour;
      }

    }


  };

</script>

<style scoped>

  .emoji-mart {
    position: absolute;
    bottom: 106%;
    right: 0;
    left: auto;
    bottom: 100%;
    z-index: 40;
  }
  .emoji-trigger {
    position: absolute;
    top: 10px;
    right: 10px;
    cursor: pointer;
    height: 20px;
  }
  .emoji-trigger path {
    transition: 0.1s all;
  }
  .emoji-trigger:hover path {
    fill: #000000;
  }
  .emoji-trigger.triggered path {
    fill: darken(#FEC84A, 15%);
  }

  .message {
    position: relative;
    margin: 0 0 1.2rem 0;
    word-break: break-word;
  }

  .message p {
    margin: 0;
    padding: 0;
    font-weight: 100;
    font-size: 1rem; 
  }

  .avatar {
    display:inline-block;
    font-size:1em;
    width:2.5em;
    height:2.5em;
    line-height:2.5em;
    text-align:center;
    border-radius:50%;
    vertical-align:middle;
    margin-right:1em;
    color:white;
    background-size: cover;
  }

</style>