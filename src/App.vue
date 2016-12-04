<template>
  <div id="app">
    <div class="col-md-5">
      <div class="panel panel-default">
        <div class="panel-heading">Debug console</div>
        <div class="panel-body" style="height: 200px; overflow-y: auto;" id="debugScroll">
          <ol style="padding: 0; margin: 0; ">
            <li style="display: list-item;" v-for="(item, index) in debugLogs">
              {{ item.message }}
            </li>
          </ol>
        </div>
      </div>
    </div>
    <div class="col-md-7">
      <div class="pull-right" style='margin: 20px 0'>
        <button v-if="!isLogin" type="button" v-on:click="login" class="loginBtn btn btn-primary">Login</button>
        <button v-if="isLogin" type="button" v-on:click="logout" class="loginBtn btn btn-primary">Logout</button>
      </div>
      <div v-if="isLogin">
        <div class="btn-group" style="margin: 20px 0">
          <button type="button" :disabled='channel===`ADMIN-${myName()}`' class="btn btn-primary" @click='listenChannel(`ADMIN-${myName()}`)'>Admin</button>
          <button type="button" :disabled='channel==="GENERAL"' class="btn btn-info" @click='listenChannel("GENERAL")' >General</button>
        </div>

        <div class="checkbox">
          <label>
            <input type="checkbox" @change="toggleAdmin" :checked='isAdmin()'>
            Admin
          </label>
        </div>





          <div class="panel panel-default" >
            <div class="panel-heading">{{channel}}</div>
            <div class="panel-body" style=" padding: 0; height: 400px; overflow-y: auto;" id="stickyScroll">

              <div class="col-xs-3" style="border-right: 1px solid #ddd; padding: 20px; height: 100%">
                <ol style="padding: 0; margin: 0; ">
                  <li style="display: list-item;" v-for='(user, index) in channelUsers'>{{user.name}} {{isMind(user) ? '(you)': ''}}</li>
                </ol>
              </div>
              <div class="col-xs-9" style="padding: 20px; height: 100%; overflow-y:auto;">
              <div v-for='item in messageList'>
                <div class="bubble well " :class="{'pull-right': isMind(item), 'pull-left': !isMind(item), green: isMind(item)  }">
                  <div class="name" style="font-size: 1rem">{{item.name}}</div>
                  <div class="cleanfix"></div>
                  <div class="pull-right msg" style="margin: 5px">{{getEncrtpedMessageFromMsgObject(item)}}</div>
                  <div class="cleanfix"></div>
                  <span class="pull-right" style="font-size: .8rem;">{{new Date(item.timestamp).toString()}}</span>
                </div>
                <div class="clearfix"></div>
              </div>
              </div>
            </div>
          </div>
          <form @submit='submit'>
            <div class="form-group">
              <div class="input-group">
                <input v-model="message"  type="text" class="form-control" placeholder="What do you say">
                <span class="input-group-btn">
                  <button class="btn btn-default" type="button">Send</button>
                </span>
              </div><!-- /input-group -->
            </div>
          </form>
        </div>


    </div>
  </div>
</template>

<script>
import shortid from 'shortid';
var provider = new firebase.auth.FacebookAuthProvider();
export default {
  name: 'app',
  components: {
  },
  data () {
    return {
      msg: 'Welcome to Your Vue.js App',
      isLogin: false,
      user: null,
      token: null,
      debugLogs: [],
      message: '',
      channel: null,
      messageList: [],
      channelref: null,
      channelUsers: [],
      keysValue : null,
    }
  },
  methods: {
    isAdmin(){
      var user = firebase.auth().currentUser;
      var channelUser = _.find(this.channelUsers, function(i){return i.name===user.displayName});
      console.log({channelUser});
      return channelUser && !!channelUser.meta.isAdmin;
    },
    myName:function(){
      return firebase.auth().currentUser.displayName
    },
    isMind: function(value){
      return value.name === firebase.auth().currentUser.displayName
    },
    login: function(){
      var that = this;
      firebase.auth().signInWithPopup(provider).then(function(result) {
        // This gives you a Facebook Access Token. You can use it to access the Facebook API.
        var token = result.credential.accessToken;
        // The signed-in user info.
        var user = result.user;
        console.log({user});
        that.token = token;
        that.updateKey()
        if(!that.channel){
          that.listenChannel("GENERAL")
        }


      }).catch(function(error) {
        // Handle Errors here.
        var errorCode = error.code;
        var errorMessage = error.message;
        // The email of the user's account used.
        var email = error.email;
        // The firebase.auth.AuthCredential type that was used.
        var credential = error.credential;
        // ...
      });
    },
    logout: function(){
      var that = this;
      var user = firebase.auth().currentUser;
      this.messageList= [];
      this.channelUsers= [];
      if(this.channelref){
        this.channelref.off()
        firebase.database().ref('chatroom').child(this.channel).off();
        firebase.database().ref('chatroom').child(this.channel).child(user.displayName).remove();
      }

      firebase.auth().signOut().then(function() {
        console.log("logout");
        that.isLogin = false;
      }, function(error) {
        alert(error);
      });
    },
    submit: function(e){
      e.preventDefault();
      var that = this;
      this.debugLogs.push({message: `Start Send message [${this.message}] Proccess`})
      this.debugLogs.push({message: `Start generate nonce using 'shortId'`})
      // var nonce = shortid.generate();
      // this.debugLogs.push({message: `nonce: ${nonce}`})
      var user = firebase.auth().currentUser;

      var message = this.message;
      var keysValue = this.keysValue
      var messageMap = {}

      this.channelUsers.forEach(function(item,index){
        messageMap[item.name] = that.getEncrtpedMessage(message, new Uint8Array(item.meta.pk), keysValue.secretKey)
      })

      this.debugLogs.push({message: `messageMap from '${this.message}' to each user  ready submit!`})
      this.debugLogs.push({message: 'please trun of console window of browser to look enrtyped message map'})
      console.log({messageMap});

      firebase.database().ref('message').child(this.channel).push({
        name: user.displayName,
        timestamp: new Date().valueOf(),
        messageMap,
        senderPublicKey: keysValue.publicKey
      })
      this.message = '';
    },
    listenChannel: function(channel){
      var user = firebase.auth().currentUser;
      this.debugLogs.push({message: `Channel Change - ${channel}`, timestamp: new Date()})
      var that = this;
      this.messageList= [];
      this.channelUsers= [];
      if(this.channelref){
        this.channelref.off()
        firebase.database().ref('chatroom').child(this.channel).off();
        firebase.database().ref('chatroom').child(this.channel).child(user.displayName).update({online: false});
      }
      this.channel = channel;
      this.channelref = firebase.database().ref(`message/${this.channel}`)

      firebase.database().ref('chatroom').child(this.channel).child(user.displayName).update({pk: that.keysValue.publicKey, online: true})
      this.channelref.on('child_added', function(snap){
        if(snap.val().messageMap[user.displayName]){
          that.messageList.push(snap.val())
          that.stickBottom("#stickyScroll")
        }
      })
      firebase.database().ref('chatroom').child(this.channel).on('child_added', function(snap){
        that.channelUsers.push({name: snap.key, meta: Object.assign({},snap.val()) })
        that.stickBottom("#stickyScrollUser")
      })
      firebase.database().ref('chatroom').child(this.channel).on('child_changed', function(snap){
        var index = _.findIndex(that.channelUsers, function(i){return i.name===snap.key});
        that.debugLogs.push({message: `User ${snap.key} renew his info`})
        that.channelUsers[index]={name: snap.key, meta: Object.assign({},snap.val()) }
        that.stickBottom("#stickyScrollUser")
      })
      firebase.database().ref('chatroom').child(this.channel).on('child_removed', function(oldChildSnapshot) {
        that.debugLogs.push({message: `User ${oldChildSnapshot.key} leaved`})
        var newArr = that.channelUsers.filter(function(i){return i.name!==oldChildSnapshot.key})
        that.channelUsers = newArr;
      });
    },
    stickBottom: _.debounce(function(key){
      $(key).animate({ scrollTop: $(key).prop("scrollHeight")}, 300);
    }, 100),
    getEncrtpedMessage: function(msg, pk, sk){
      // console.log("getEncrtpedMessage");
      var m = nacl.util.decodeUTF8(msg);
      var n = nacl.randomBytes(nacl.secretbox.nonceLength);
      // console.log({m, n, pk, sk});
      var box = nacl.box(m, n, pk, sk)
      // console.log("msg", nacl.util.encodeBase64(box));
      return {
        message: nacl.util.encodeBase64(box),
        nonce: n
      }
      //
      // var message = nacl.box.open(box, n, pk, sk);
      // console.log("message", nacl.util.encodeUTF8(message));
    },
    desrptMessage: function(box, pk, sk, n){
      // console.log({box, pk, sk, n});
      // console.log("nacl.box.open(box, n, pk, sk)", nacl.box.open(box, n, pk, sk));
      var message = nacl.util.encodeUTF8(nacl.box.open(box, n, pk, sk));
      // console.log("message", message);
      return message
    },
    getEncrtpedMessageFromMsgObject: function(item){
      var user  = firebase.auth().currentUser;
      var myMessageObject = item.messageMap[user.displayName];
      // console.log("myMessageObject.message", myMessageObject.message);
      // console.log("myMessageObject.nonce", myMessageObject.nonce);
      var pk = new Uint8Array(item.senderPublicKey);
      var sk = this.keysValue.secretKey;
      // console.log({pk, sk});
      var n = new Uint8Array(myMessageObject.nonce);
      var box = nacl.util.decodeBase64(myMessageObject.message);
      return this.desrptMessage(box, pk, sk, n);
    },
    updateKey: function(){
      var user = firebase.auth().currentUser;
      if(!store.get('sk')){
        // console.log("nacl.box.keyPair()", );
        var keys = nacl.box.keyPair();

        this.debugLogs.push({message: `RSA key setup`})
        // console.log("keys.publicKey", keys.publicKey);
        this.keysValue = keys;
        firebase.database().ref('user').child(user.displayName).update({pk: keys.publicKey})
        store.set('sk', nacl.util.encodeBase64(keys.secretKey))
      }else{
        var sk = store.get('sk')
        var keyPair = nacl.box.keyPair.fromSecretKey(nacl.util.decodeBase64(store.get('sk')));
        this.keysValue = keyPair;
        this.debugLogs.push({message: `RSA key restore`})
      }
    },
    toggleAdmin(){
      var user = firebase.auth().currentUser;
      firebase.ref('chatroom').child(user.displayName).update(admin)
    }
  },
  created: function(){
    const that  = this;
    // setInterval(function () {
    //   $("#debugScroll").animate({ scrollTop: $("#debugScroll").prop("scrollHeight")}, 100);
    // }, 1000);
    firebase.auth().onAuthStateChanged(function(user) {
      if (user) {
        that.isLogin = true;
        that.user = user;
        console.log("signed in !");
        that.updateKey()
        that.debugLogs.push({message: "Signed in !", timestamp: new Date()})
        if(!that.channel){
          that.listenChannel("GENERAL")
        }


        // firebase.database().ref('user').child(user.displayName).update({
        //   name: user.displayName,
        //   email : user.email,
        //   photoUrl : user.photoURL,
        //   uid : user.uid,
        //   online: true,
        // })
        // User is signed in.
      } else {
        that.isLogin = false;
        that.user = null;
        that.debugLogs.push({message: "Checked not user logined", timestamp: new Date()})
      }
    });


  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  /*text-align: center;*/
  color: #2c3e50;
  margin-top: 60px;
}

h1, h2 {
  font-weight: normal;
}

ul {
  list-style-type: none;
  padding: 0;
}

li {
  display: inline-block;
  margin: 0 10px;
}

a {
  color: #42b983;
}
.bubble.green{
  background-color: #DCF8C6;
}
</style>


<!-- var EncrtpedMessage = that.getEncrtpedMessage(message, keysValue.publicKey, keysValue.secretKey)
console.log({EncrtpedMessage});

console.log(that.desrptMessage(nacl.util.decodeBase64(EncrtpedMessage.message), keysValue.publicKey, keysValue.secretKey, EncrtpedMessage.nonce)); -->
