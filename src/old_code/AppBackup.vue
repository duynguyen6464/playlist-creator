<template>
  <div id="app" ref="app">
    <header class="header">
      <div class="left">
        <h1>Playlist Creator</h1>
      </div>
      <div class="right">
        <button
          v-if="!accessToken"
          class="button login-button"
          @click="login">
          Login
        </button>
        <div
          v-if="currentUser.id && currentUser.images"
          class="user-info text">
          <img
            :src="currentUser.images[0].url"
            alt=""
            class="image">
            {{currentUser.display_name || currentUser.id }}
        </div>
      </div>
    </header>
    <div class="playlist-creator">
      <div class="search-area vertical-space">
        <search @selected-artist="updateCurrentSelectedArtist"></search>
      </div>
      <div class="songs vertical-space" v-if="selectedArtist && selectedArtist.id">
        <div class="browse-albums-and-songs vertical-space column">
          <div class="selected-artist-info">
            <artist-info
              :selected-artist="selectedArtist"
              :related-artists="relatedArtists"
              :top-tracks="currentSelectedArtistTopTracks"
              @select-artist="updateCurrentSelectedArtist">
            </artist-info>
          </div>
          <div class="row">
            <div class="current-album-songs column limit-height" v-if="currentSelectedArtistAlbumTracks.length > 0">
              <div class="text vertical-space">
                <img
                  v-if="currentSelectedAlbum.images.length > 0"
                  class="image"
                  :src="currentSelectedAlbum.images[1].url" :alt="currentSelectedAlbum.name">
                  {{currentSelectedAlbum.name}}
              </div>
              <song-list
                :songs="currentSelectedArtistAlbumTracks">
              </song-list>
            </div>
            <div class="albums column limit-height">
              <album-browser
                :albums="currentSelectedArtistAlbums"
                @select-album="selectAlbum">
              </album-browser>
            </div>
          </div>
        </div>
      </div>
      <!-- end of current-album -->
      <playlist-manager
        v-if="this.accessToken">
      </playlist-manager>
      <!-- end of playlist-manager -->
      <player></player>
    </div>
  </div>
</template>

<script>
import {store} from './stores'
import { mapGetters } from 'vuex'
import Search from './components/search'
import SongList from './components/song-list'
import Player from './components/player'
import AlbumBrowser from './components/album-browser'
import ArtistInfo from './components/artist-info'
import PlaylistManager from './components/playlist-manager'
import VisualRangeNumber from './components/visual-range-number'
import {uniqBy} from 'lodash'
import spotifyApi from './loaders/spotifyApi'
import {
  generateRandomString,
  getAccessToken
} from './core/spotify-service'

export default {
  name: 'app',
  data () {
    return {
      selectedArtist: {},
      currentSelectedArtistTopTracks: [],
      currentSelectedArtistAlbums: [],
      currentSelectedAlbum: {},
      currentSelectedArtistAlbumTracks: [],
      relatedArtists: [],
      userPlaylists: []
    }
  },
  watch: {
    accessToken () {
      // session expired
      if (this.accessToken === '') {
        if (window.confirm('Your session is over, do you wish to login again?')) {
          this.login()
        }
      }
    },
    async currentSelectedAlbum () {
      this.currentSelectedAlbum
      // album tracks
      if (this.currentSelectedAlbum.id) {
        spotifyApi.getAlbumTracks(this.currentSelectedAlbum.id)
        .then(data => {
          // this.currentSelectedArtistAlbumTracks = data.items
          return data.items
        })
        .then(tracks => {
          // GET tracks via tracks API, return tracks with album
          spotifyApi.getTracks(tracks.map(t => t.id))
          .then(({tracks}) => {
            this.currentSelectedArtistAlbumTracks = tracks
          })
          .catch(err => { console.error(err) })
        })
        .catch(err => { console.error(err) })
      }
    },
    selectedArtist () {
      if (this.selectedArtist.id) {
        // reset currentSelectedAlbum
        this.currentSelectedAlbum = {}
        this.currentSelectedArtistAlbumTracks = []
        // TODO change 'BR' to geolocation data

        // top tracks
        spotifyApi.getArtistTopTracks(this.selectedArtist.id, 'US')
        .then(data => { this.currentSelectedArtistTopTracks = data.tracks })
        .catch(err => { console.error(err) })

        // albums
        // TODO check if has next url
        spotifyApi.getArtistAlbums(this.selectedArtist.id, {limit: 50})
        .then(data => {
          this.currentSelectedArtistAlbums = uniqBy(data.items.filter(a => a.album_type === 'album'), 'name')
        })
        .catch(err => { console.error(err) })

        // related artists
        spotifyApi.getArtistRelatedArtists(this.selectedArtist.id)
        .then(data => { this.relatedArtists = data.artists })
        .catch(err => { console.error(err) })
      }
    }
  },
  mounted () {
    // store.dispatch('resetAll')
    const accessToken = getAccessToken() // comes from URL
    if (accessToken) {
      store.dispatch('saveAccessToken', {accessToken})
      spotifyApi.setAccessToken(accessToken)
      spotifyApi.getMe()
      .then((me) => {
        store.dispatch('saveCurrentUser', {currentUser: me})
        return me
      })
      .catch(e => { store.dispatch('cleanAccess') })
    } else {
      spotifyApi.setAccessToken(this.accessToken) // from vuex
    }

    // set user playlists
    if (this.currentUser.id) {
      spotifyApi.getUserPlaylists(this.currentUser.id, {limit: 50})
      .then(data => {
        // get only playlists owned by the user
        const playlists = data.items.filter(i => i.owner.id === this.currentUser.id)
        store.dispatch('saveUserPlaylists', {playlists})
        return playlists
      })
      .then(playlists => {
        // playlist might not exist and the user just reload
        let playlistWasDeleted = playlists.filter(p => p.id === this.playlistObject.id).length === 0
        if (this.playlist.length !== 0 && playlistWasDeleted) {
          window.alertify.warning('Your playlist was deleted on Spotify, create a new one')
          store.dispatch('resetPlaylistStore')
        }
      })
      .catch(e => {
        console.warn(e)
        store.dispatch('cleanAccess')
      })
    }
  },
  methods: {
    selectAlbum (album) {
      this.currentSelectedAlbum = album
    },
    updateCurrentSelectedArtist (artist) {
      this.selectedArtist = artist
    },
    login () {
      const stateKey = 'spotify_auth_state'
      const clientId = '49275dd30324422b8bbba8bdea0e7b8c' // Your client id
      let redirectUri = window.location.origin + '/' // Your redirect uri
      const state = generateRandomString(16)
      window.localStorage.setItem(stateKey, state)
      const scope = 'user-read-private user-read-email playlist-modify-public playlist-modify-private playlist-read-private'
      let url = 'https://accounts.spotify.com/authorize'
      url += '?response_type=token'
      url += '&client_id=' + encodeURIComponent(clientId)
      url += '&scope=' + encodeURIComponent(scope)
      url += '&redirect_uri=' + encodeURIComponent(redirectUri)
      url += '&state=' + encodeURIComponent(state)
      window.location = url
    }
  },
  computed: {
    ...mapGetters([
      'accessToken',
      'currentUser',
      'playlist',
      'playlistName',
      'playlistObject'
    ])
  },
  components: {
    Search,
    SongList,
    Player,
    AlbumBrowser,
    ArtistInfo,
    PlaylistManager,
    VisualRangeNumber
  }
}
</script>
<style lang="sass">
@import './assets/colors'

body
  color: $base-color
  background: $base-background
  margin: 0
  padding: 0

#app
  font-family: 'Avenir', Helvetica, Arial, sans-serif
  -webkit-font-smoothing: antialiased
  -moz-osx-font-smoothing: grayscale
  display: flex
  flex-direction: column
  max-width: 100%
  margin: 0.5rem

.browse-albums-and-songs
  min-height: 400px
  overflow: auto

.playlist-creator
  display: flex
  flex-direction: column
  margin-bottom: 50px

.albums
  flex: 2

.selected-artist-info
  min-height: 300px

.user-info
  display: flex
  align-items: center
  img
    border-radius: 50%
    margin-right: 0.5rem
    width: 50px
    height: 50px

// utils
.vertical-space
  margin: 20px 0

.sm-vertical-space
  margin: 7px 0

.column
  display: flex
  flex-direction: column
  flex: 1 1 auto

.row
  display: flex
  flex-direction: row

.clickable
  cursor: pointer
  &:hover
    text-decoration: underline

.checkbox
  transform: scale(1.4)
  padding: 4px
  margin: 4px

.header
  display: flex
  width: 100%
  align-items: center
  justify-content: space-between

.button
  cursor: pointer
  border: none
  font-size: 1.6rem
  border-radius: 5px
  padding: 5px 10px
  outline: none

.input
  height: 2.3rem
  font-size: 1.7rem
  max-width: 100%
  padding: 0
  margin: 0
  line-height: 1
  box-sizing: border-box

.limit-height
  max-height: 400px
  overflow: auto

.image
  width: 70px
  height: 70px

.media-wrapper
  display: flex
  width: 100%
  max-height: 400px
  overflow-x: scroll
  flex-wrap: wrap

.media
  display: flex
  align-items: center
  font-size: 1.3rem
  margin-bottom: 0.5rem
  margin-right: 1.2rem
  &:last-child
    margin-bottom: 0
    margin-right: 0
  img
    margin-right: 5px

.text
  font-size: 1.4rem

.circle
  border-radius: 50%

.mr-5
  margin-right: 5px
  &:last-child
    margin-right: 0

.spotify-green
  background: #1ED760
  color: black
  &:hover
    background: #31f778
</style>

<style type="text/css">
::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}
::-webkit-scrollbar-button {
  width: 0px;
  height: 0px;
}
::-webkit-scrollbar-thumb {
  background: #dddddd;
  border: 0px none #ffffff;
  border-radius: 50px;
}
::-webkit-scrollbar-thumb:hover {
  background: #909090;
}
::-webkit-scrollbar-thumb:active {
  background: #3f3f3f;
}
::-webkit-scrollbar-track {
  background: #666666;
  border: 0px none #ffffff;
  border-radius: 46px;
}
::-webkit-scrollbar-track:hover {
  background: #666666;
}
::-webkit-scrollbar-track:active {
  background: #333333;
}
::-webkit-scrollbar-corner {
  background: transparent;
}

/* Media querys*/
@media (max-width: 600px) {
  .row {
    flex-wrap: wrap;
  }
  .column {
    width: 100%;
  }
}
</style>
