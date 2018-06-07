
<template>
  <b-container fluid>
    <b-row class="pt-2 header container-fluid">
      <b-col align-self="center" class="logo">
        <a href="https://zacharyseguin.ca"><img src="../assets/logo.svg" alt="Zachary Seguin" /></a>
      </b-col>
      <!-- Controls -->
      <b-col align-self="center" cols="9">
        <b-row>
          <b-col align-self="center" cols="2">
            <MediaControls />
          </b-col> 
          <b-col align-self="center">
            <NowPlaying />
          </b-col>
        </b-row>
      </b-col>
    </b-row>
    <b-row class="main">
      <!-- Sidebar -->
      <b-col>
        <b-button variant="link" v-on:click.prevent="unauthorize()">Sign out of Apple Music</b-button>

        <!-- User library -->
        <b-list-group>
          <b-list-group-item href="#" v-on:click.prevent="load('user:albums')">Albums</b-list-group-item>
          <b-list-group-item href="#" v-on:click.prevent="load('user:artists')">Artists</b-list-group-item>
          <b-list-group-item href="#" v-on:click.prevent="load('user:songs')">Songs</b-list-group-item>
        </b-list-group>
        <!-- User playlists -->
        <h2 class="text-uppercase heading">Playlists</h2>
        <Playlists :playlists="userPlaylists" />
      </b-col>
      
      <!-- Main content -->
      <b-col cols="9" v-if="results">
        <div v-if="results.type === 'albums'">
          <Albums :albums="results.albums" title="Albums" />
        </div>
        <div v-if="results.type === 'album'">
          <SongCollection :collection="results.album" />
        </div>
        <div v-if="results.type === 'artists'">
          <Artists :artists="results.artists" title="Artists" />
        </div>
        <div v-if="results.type === 'artist'">
          <Albums :albums="results.artist.relationships.albums.data" :title="results.artist.attributes.name" />
        </div>
        <div v-if="results.type === 'songs'">
          <Songs :songs="results.songs" title="Songs" />
        </div>
        <div v-if="results.type === 'playlist'">
          <SongCollection :collection="results.playlist" />
        </div>
      </b-col>
      <b-col cols="9" v-else>
      </b-col>
    </b-row>
  </b-container>
</template>

<script>
import EventBus from '../event-bus';
import Playlists from './Playlists.vue';
import NowPlaying from './NowPlaying.vue';
import MediaControls from './MediaControls.vue';
import Albums from './Albums.vue';
import Artists from './Artists.vue';
import Songs from './Songs.vue';
import SongCollection from './SongCollection.vue';

export default {
  name: 'Player',
  props: {
    musicKit: Object,
  },
  data: function() {
    return {
      userPlaylists: [],
      results: null,
    }
  },
  methods: {
    load: function(content) {
      var req = {};

      switch (content) {
        case 'user:albums': req = { albums: null };
                           break;

        case 'user:artists': req = { artists: null };
                           break;

        case 'user:songs': req = { songs: null };
                           break;
      }

      EventBus.$emit('load', req);
    },
    unauthorize: function() {
      EventBus.$emit('unauthorize');
    }
  },
  created: function() {
    // Application events
    this.onQueue = (descriptor) => {
      console.log(descriptor);
      this.musicKit.setQueue(descriptor)
        .then(
          (q) => { 
            // Store the sourceId of each item, so we can update the UI as the song plays
            q.items.forEach(i => i.sourceId = i.id);

            this.musicKit.player.changeToMediaItem(q.item(descriptor.startPosition || 0)).then(
              () => null,
              err => console.error(err))
          },
          (err) => console.error(err)
        );
    }
    EventBus.$on('queue', this.onQueue);

    this.onLoad = (description) => {
      // Clear previous results
      this.results = null;
      window.scrollTo(0, 0);

      // We have an explicit result to show

      // User albums
      if ('albums' in description) {
        var albums = [];
        let getAlbums = (start = 0) => {
          this.musicKit.api.library.albums(description.albums, {
            offset: start,
            limit: 100
          }).then((inAlbums) => {
            albums = albums.concat(inAlbums);

            // Set the results, if not already done. Otherwise, update the songs.
            if (!this.results) {
              this.results = {
                type: 'albums',
                albums: albums
              };
            } else {
              this.results.albums = albums;
            }

            if (inAlbums.length == 100) {
              getAlbums(start + 100);
            }
            
          });
        };

        getAlbums();
      }
      else if ('artists' in description) {
        var artists = [];
        let getArtists = (start = 0) => {
          this.musicKit.api.library.artists(description.artists, {
            offset: start,
            limit: 100
          }).then((inArtists) => {
            artists = artists.concat(inArtists);

            // Set the results, if not already done. Otherwise, update the songs.
            if (!this.results) {
              this.results = {
                type: 'artists',
                artists: artists
              };
            } else if (this.results.type) {
              this.results.artists = artists;
            }

            if (inArtists.length == 100) {
              getArtists(start + 100);
            }
            
          });
        };

        getArtists();
      }
      // User songs
      else if ('songs' in description) {
        var songs = [];
        let getSongs = (start = 0) => {
          this.musicKit.api.library.songs(description.songs, {
            offset: start,
            limit: 100
          }).then((inSongs) => {
            songs = songs.concat(inSongs);

            // Set the results, if not already done. Otherwise, update the songs.
            if (!this.results) {
              this.results = {
                type: 'songs',
                songs: songs
              };
            } else {
              this.results.songs = songs;
            }

            if (inSongs.length == 100) {
              getSongs(start + 100);
            }
            
          });
        };

        getSongs();
      }
      // Album
      else if ('album' in description) {
        this.musicKit.api.library.album(description.album).then(res => {
          this.results = {
            type: 'album',
            album: res
          };
        });
      }
      else if ('artist' in description) {
        this.musicKit.api.library.artist(description.artist, { include: 'albums' }).then(res => {
          this.results = {
            type: 'artist',
            artist: res
          };
        });
      }
      // Playlist
      else if ('playlist' in description) {
        this.musicKit.api.library.playlist(description.playlist).then(res => {
          this.results = {
            type: 'playlist',
            playlist: res
          };
        });
      }
    }
    EventBus.$on('load', this.onLoad);

    // Load user playlists
    let getPlaylists = (start = 0) => {
      this.musicKit.api.library.playlists(null, {
        offset: start,
        limit: 100
      }).then((playlists) => {
        this.userPlaylists = this.userPlaylists.concat(playlists);

        if (playlists.length == 100) {
          getPlaylists(start + 100);
        }
      });
    };

    getPlaylists();
  },
  destroyed: function() {
    EventBus.$off('queue', this.onQueue);
    EventBus.$off('load', this.onLoad);
  },
  components: {
    Playlists,
    NowPlaying,
    MediaControls,
    Songs,
    Albums,
    Artists,
    SongCollection
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.main {
  padding-top: 100px;
  padding-bottom: 20px;
}

.logo img {
  height: 100%;
  max-height: 40px;
}

.header {
  position: fixed;
  z-index: 100;
  background: #f8f8f8;
  box-shadow: 1px 1px 1px rgba(0, 0, 0, .2);
  padding: 8px 5px;
}

.heading {
  font-size: 18px;
  font-weight: normal;
  padding-top: 10px;
  color: #666;
}
</style>