# Comp3334 Chatroom

This is an experiment to setup a more secure IM system base on existing IM security issues.  
The Project using [firebase](https://firebase.google.com/) as the realtime data base and hosting server.  
Whole project is using HTTPS protection and using **public-private key Encryption** for message encryption before send to server.   

## Main tech
 - Using Vue.js as frontend 
 - [tweetnacl-js](https://github.com/dchest/tweetnacl-js) **Public-key authenticated encryption** client side encryption

## Build Setup

``` bash
# install dependencies
npm install

# serve with hot reload at localhost:8080
npm run dev

# build for production with minification
npm run build
```

For detailed explanation on how things work, consult the [docs for vue-loader](http://vuejs.github.io/vue-loader).
