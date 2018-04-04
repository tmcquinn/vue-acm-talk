<template>
    <div id="app">
        <div  class="container-fluid" v-if="showAuthButton">
            <button class="btn"><a target="_blank" v-bind:href="twitchAuthURI">Click Here to authorize</a></button>
        </div>

        <div v-else>
            <AppHeader v-bind:loaded="loaded" v-bind:profile-img="userData.logo" v-bind:name="userData.name"/>
            <div class="container-fluid" v-if="loaded">
                <div class="col">
                    <div class="form-group form-row input-buffer">
                        <input type="text" class="form-control input-width" id="search" v-model="gameOfInterest" placeholder="Enter a game name...">
                        <button class="btn" v-on:click="getStreamsList">Find</button>
                    </div>

                    <div class="row">
                        <div class="col arrange-items" v-for="item in streamList" v-bind:key="item._id">
                            <div class="card card-width">
                                <img class="card-img-top" :src="item.preview.medium"/>
                                <div class="card-body">
                                    <img height="50" width="50" :src="item.channel.logo" />
                                    <a class="card-title" target="_blank" :href="item.channel.url">{{item.channel.name}}</a>
                                    <p>Viewers: {{item.viewers}} | Followers: {{item.channel.followers}}</p>
                                    <p>{{item.channel.status}}</p>
                                    <div v-if="isFollowing(item.channel._id)">
                                        <i class="fas fa-heart fa-2x" style="color: #FFB6C1"></i>
                                    </div>
                                    <div v-else>
                                        <i v-on:click="follow(item.channel._id)" class="far fa-heart fa-2x" style="color: #FFB6C1"></i>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>

            <div class="loading" v-else>
                <i class="fas fa-spinner fa-pulse fa-10x"></i>
            </div>
        </div>
    </div>
</template>

<script>
// import jsonp from 'jsonp'
import env from '../environment.json';
import AppHeader from './components/AppHeader.vue';

import axios from 'axios';

export default {
    name: 'app',
    components: {
        AppHeader
    },

    data ()  {
        return {
            twitchAuthURI: "",
            showAuthButton: true,
            authToken: '',
            userToken: '',
            gameOfInterest: "Overwatch",
            streamLimit: 100,
            noStreams: false,
            totalStreams: -1,
            totalPages: -1,
            currentPage: 1,
            userData: {},
            streamList: [],
            followerStreamList: [],
            loaded: false
            // I only follow like 10 peeps D;
        }
    },

    methods: {
        /* Using the get total streams api call, we will get 1 stream to get the total */
        getStreamsList() {
            this.loaded = false;
            var config = { "headers": { 
                "Accept": "applicaiton/vnd.twitchtv.v5+json",
                "Client-ID": `${env.client_id}`,
            }}
            axios.get(`https://api.twitch.tv/kraken/streams/?game=${this.gameOfInterest}&limit=1&language=en&stream_type=live`, config)
                .then((response) => {
                    this.totalStreams = response.data._total;
                    if (response.data._total === 0) {
                        this.noStreams = true;
                    }
                    else {
                        this.getBottomStreams()
                    }
                }).catch(function(err) {
                    console.log(err);
                })
        },

        getBottomStreams() {
            if(!this.noStreams) {
                var config = { "headers": { 
                "Accept": "applicaiton/vnd.twitchtv.v5+json",
                "Client-ID": `${env.client_id}`,
            }}
                // totalStreams - limit * page = offset
                this.currentOffset = this.totalStreams - (this.streamLimit * this.currentPage);
                axios.get(`https://api.twitch.tv/kraken/streams/?game=${this.gameOfInterest}&limit=${this.streamLimit}&language=en&stream_type=live&offset=${this.currentOffset}`, config)
                    .then((response) => {
                        this.streamList = response.data.streams;
                        this.getFollowedLiveStreams();
                    })
            }
        },

        getCurrentUser() {
            var config = { "headers": { 
                "Accept": "applicaiton/vnd.twitchtv.v5+json",
                "Client-ID": `${env.client_id}`,
                "Authorization": `OAuth ${this.userToken}`
                }
            }

            axios.get('https://api.twitch.tv/kraken/user', config)
                .then((response) => {
                    this.userData = response.data;
                })

        },

        getFollowedLiveStreams() {
            var config = { "headers": { 
                "Accept": "applicaiton/vnd.twitchtv.v5+json",
                "Client-ID": `${env.client_id}`,
                "Authorization": `OAuth ${this.userToken}`
                }
            }

            axios.get('https://api.twitch.tv/kraken/streams/followed?stream_type=live', config)
                .then((response) => {
                    if(response.data._total > this.totalStreams) {
                        console.log("need more results")
                    }
                    this.followerStreamList = response.data.streams;
                    this.loaded = true;
                })
        },

        isFollowing(_id) {
            console.log("Checking if follower");
            for (var i = 0; i < this.followerStreamList.length; i++) {                    
                if (_id === this.followerStreamList[i].channel._id) {
                    return true;
                }
            }
            return false;
        },

        follow(_id) {
            var config = { "headers": { 
                "Accept": "applicaiton/vnd.twitchtv.v5+json",
                "Client-ID": `${env.client_id}`,
                "Authorization": `OAuth ${this.userToken}`
                }
            }

            console.log("sending request");
            axios.put(`https://api.twitch.tv/kraken/users/${this.userData._id}/follows/channels/${_id}`, null ,config)
                .then(() => {
                    this.getFollowedLiveStreams();
                })
        }
    },

    /* 
     * Created is a function that is called when the component is loaded.  
     */

    created() {
        if (this.$route.params.token === undefined) {
            // Construct the URI
            this.twitchAuthURI = `https://id.twitch.tv/oauth2/authorize?client_id=${env.client_id}&redirect_uri=http://localhost:8080&response_type=token+id_token&scope=openid+user_read+user_follows_edit+chat_login&state=fpmuirwruorjynkdv0ud5tw2fqws3c&callback=%3F`;
        }
        // This should trigger if token is in the url or after an auth
        else {
            this.authToken = this.$route.params.token;
            this.userToken = this.$route.params.access;
            this.showAuthButton = false;

            this.getCurrentUser();
            this.getStreamsList();

        }
    }
}
</script>

<style>
#app {
    font-family: 'Avenir', Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
}
#header {
    align-items: center;
}
.input-buffer {
    margin-top: 15px;
    justify-content: center;
}

.input-width {
    max-width: 500px;
}

.arrange-items {
    display:flex;
    flex-direction: row;
    flex-wrap: wrap;
    justify-content: space-between;
}

.arrange-col {
    display:flex;
    flex-direction: column;
    flex-wrap: wrap;
    flex-grow: 0;
    width: 350px;
}

.card-width {
    width:300px;
    margin-bottom: 25px;
}

.fa-spiiner {
    font-size: 50px;
}

.loading {
    display: flex;
    justify-content: center;
    align-items: center;
    width: 100vw;
    height: 100vh;
}
</style>
