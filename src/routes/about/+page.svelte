<script>
  import { onMount } from "svelte";
  import { writable } from "svelte/store";
  import SpotifyWebApi from "spotify-web-api-js";
  import { Button } from "$lib/components/ui/button";
  import { Input } from "$lib/components/ui/input";
  import * as Accordion from "$lib/components/ui/accordion";
  import PocketBase from "pocketbase";


  const pb = new PocketBase("https://song-app.pockethost.io");

  const delay = (ms) => new Promise((res) => setTimeout(res, ms));

  let searchResultTracks = [];
  let searchQuery = "";

  let spotifyApi = new SpotifyWebApi();

  let loginState = false;
  let currentTrack = null;
  let isPaused = false;
  let playlistData = [];

  let player = null;
  let accessToken = null;
  let playlists = [];
  let localQueue = [];
  let isChanging = false;
let playlistSongs=[];


  const clientId = "da22f1252f514bb59be6b5588fddaf25";
  const redirectUri =
    "https://music-app-ruby-six.vercel.app/about/"; // Update with your redirect URI
  const scopes =
    "user-read-private user-read-playback-state user-modify-playback-state user-read-currently-playing streaming app-remote-control user-read-email"; // Adjust scopes as needed
  const loginUrl = new URL("https://accounts.spotify.com/authorize");
  loginUrl.searchParams.set("client_id", clientId);
  loginUrl.searchParams.set("redirect_uri", redirectUri);
  loginUrl.searchParams.set("scope", scopes);
  loginUrl.searchParams.set("response_type", "token");


  async function setCurrentSong() {
    let currentTrackId = await spotifyApi.getMyCurrentPlayingTrack();
    console.log(currentTrackId.item.id);
    currentTrack = await spotifyApi.getTrack(currentTrackId.item.id);
  }
  async function getUserPlaylistsDetails(userId) {
    try {
        const playlistsResponse = await spotifyApi.getUserPlaylists(userId);
        const fetchedPlaylists = playlistsResponse.items;


        for (let playlist of fetchedPlaylists) {
            const details = await spotifyApi.getPlaylist(playlist.id);
            playlistData = [...playlistData,details]
        }
    } catch (error) {
        console.error('Failed to get playlists details:', error);
    }
}
  async function initializePlayer() {
    player = new Spotify.Player({
      name: "Supermusic Player",
      getOAuthToken: (cb) => {
        cb(accessToken);
      },
      volume: 0.7,
    });

    // Ready
    player.addListener("ready", ({ device_id }) => {
      console.log("Ready with Device ID", device_id);
      spotifyApi.transferMyPlayback([device_id], { play: false });
    });

    // Not Ready
    player.addListener("not_ready", ({ device_id }) => {
      console.log("Device ID has gone offline", device_id);
    });

    player.addListener("initialization_error", ({ message }) => {
      console.error(message);
    });

    player.addListener("authentication_error", ({ message }) => {
      console.error(message);
      throw redirect(308, '/');
    });

    player.addListener("account_error", ({ message }) => {
      console.error(message);
    });
    player.addListener("player_state_changed", (state) => {
      if (state.position === 0 && isChanging == false) {
        if (localQueue.length > 0) {
          isChanging = true;
          playNextSongInQueue();
          waitReset();
          
        }
        console.log("Song has ended");
        //getQueue();
        //setCurrentSong();
      }
      if (state.position > 0) {
        isChanging = false;
      }
      console.log(state.position);
    });

    player.connect();
  }
  async function waitReset() {
    await delay(3000);
    //console.log(currentTrack);
    isChanging = false;
    //setCurrentSong();
    console.log(currentTrack);
  }
    console.log("onmounted")
    let hashParams = window.location.hash;
    if (hashParams) {
      loginState = true;
      let searchParams = new URLSearchParams(hashParams.substring(1));
      accessToken = searchParams.get("access_token");
      if (accessToken) {
        spotifyApi.setAccessToken(accessToken);
        initializePlayer();
        getUserPlaylistsDetails();
       // const data = await spotifyApi.getUserPlaylists();
        //playlists = data;
        //await delay(3000);
        //getQueue();
        setCurrentSong();
        isPaused = true;
      }
    }
  async function searchMusic() {
    let data = await spotifyApi.searchTracks(searchQuery);
    searchResultTracks = data.searchResultTracks.items;
  }

  async function playTrack(uri, track) {
    isChanging = true;
    let options = {
      uris: [uri],
    };
    console.log(track.name);
    spotifyApi.play(options, (error, response) => {
      if (error) {
        console.error("Error starting playback:", error);
      } else {
        console.log("Playback started successfully:", response);
        currentTrack = track;
        //playstate = true;
        isPaused = false;
        waitReset();
      }
    });
  }
  async function playNextSongInQueue(){
    let nextSong = localQueue[0];
          let options = {
            uris: [nextSong.uri],
          };
          spotifyApi.play(options, (error, response) => {
            if (error) {
              console.error("Error starting playback:", error);
            } else {
              console.log("Playback started successfully:", response);
              localQueue = localQueue.slice(1);
              isPaused = false;
              currentTrack = nextSong;
            }
          });
        }
  async function getPlaylistSongs(playlistid){
    let playlistSongs = [];
    try {
        let playlistSongsData = await spotifyApi.getPlaylistTracks(playlistid);
        console.log(playlistSongs);
        playlistSongs = playlistSongsData.items;
        console.log(playlistSongs);
    } catch (error) {
        console.error(error);
    }
    console.log(playlistSongs);
    return playlistSongs;
}
async function playPlaylist(playlist) {
    try {
      console.log(playlist);
      let response = await spotifyApi.getPlaylistTracks(playlist.id);
      console.log(response);
      let playlistTracks = response.items.map((item) => item.track);
      console.log(playlistTracks);
      let playlisttrack = null;
      for (playlisttrack of playlistTracks) {
        //await spotifyApi.queue(playlisttrack.uri);
        //localQueue.push(playlisttrack);
        localQueue = [...localQueue, playlisttrack];
        console.log(localQueue);
      }
      spotifyApi.play();
      isPaused = false;
      let currentTrackId = await spotifyApi.getMyCurrentPlayingTrack();
      console.log(currentTrackId.item.id);
      currentTrack = await spotifyApi.getTrack(currentTrackId.item.id);
      console.log(currentTrack);
      console.log("queueupdate");

      //getQueue();
      //console.log(currentTrack.item)
    } catch (error) {
      console.error(
        "Failed to get playlist searchResultTracks or add them to the queue",
        error,
      );
    }
  }


</script>

<svelte:head>
  <script src="https://sdk.scdn.co/spotify-player.js"></script>
</svelte:head>
{#if loginState == false}
  <div class="flex items-center justify-center h-screen">
    <Button class="flex gap-2" href={loginUrl.toString()}
      ><svg
        xmlns="http://www.w3.org/2000/svg"
        width="16"
        height="16"
        fill="currentColor"
        class="bi bi-spotify"
        viewBox="0 0 16 16"
      >
        <path
          d="M8 0a8 8 0 1 0 0 16A8 8 0 0 0 8 0m3.669 11.538a.5.5 0 0 1-.686.165c-1.879-1.147-4.243-1.407-7.028-.77a.499.499 0 0 1-.222-.973c3.048-.696 5.662-.397 7.77.892a.5.5 0 0 1 .166.686m.979-2.178a.624.624 0 0 1-.858.205c-2.15-1.321-5.428-1.704-7.972-.932a.625.625 0 0 1-.362-1.194c2.905-.881 6.517-.454 8.986 1.063a.624.624 0 0 1 .206.858m.084-2.268C10.154 5.56 5.9 5.419 3.438 6.166a.748.748 0 1 1-.434-1.432c2.825-.857 7.523-.692 10.492 1.07a.747.747 0 1 1-.764 1.288"
        />
      </svg>Login to Spotify</Button
    >
  </div>
{/if}

{#if loginState == true}
  <div class="flex gap-4">
    <div class="grid gap-4 w-full">
      <div class="flex gap-4">
        <Input bind:value={searchQuery} placeholder="Search for music" />
        <Button on:click={searchMusic}>Search</Button>
      </div>

      <div class="grid grid-cols-1 gap-2">
        {#each searchResultTracks as track (track.id)}
          <div class="flex flex-row relative">
            <Button
              variant="ghost"
              class="flex gap-4"
              on:click={() => playTrack(track.uri, track)}
            >
              <img
                src={track.album.images[0].url}
                width="30"
                height="30"
                class="rounded-md"
              />
              <h2>{track.name}</h2>
              <p>{track.artists[0].name}</p>
            </Button>
            <Button
              variant="ghost"
              class="flex gap-4 absolute right-0 bg-neutral-50"
              on:click={() => localQueue = [...localQueue, track]}
            >
            <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" class="bi bi-plus" viewBox="0 0 16 16">
              <path d="M8 4a.5.5 0 0 1 .5.5v3h3a.5.5 0 0 1 0 1h-3v3a.5.5 0 0 1-1 0v-3h-3a.5.5 0 0 1 0-1h3v-3A.5.5 0 0 1 8 4"/>
            </svg>
            </Button>
            
          </div>
        {/each}
      </div>
      {#if currentTrack != null}
        <div
          class="flex fixed gap-6 left-0 bg-neutral-100 bottom-0 w-full h-20 outline-slate-200 p-4 justify-center items-center z-50"
        >
          <img
            src={currentTrack.album.images[0].url}
            width="60"
            height="60"
            class="rounded-md"
          />
          {currentTrack.name}
          <Button
            variant="ghost"
            on:click={() => {
              isPaused = !isPaused;
              player.togglePlay();
            }}
            >{#if isPaused}
              <svg
                xmlns="http://www.w3.org/2000/svg"
                width="20"
                height="20"
                fill="currentColor"
                class="bi bi-play"
                viewBox="0 0 16 16"
              >
                <path
                  d="M10.804 8 5 4.633v6.734zm.792-.696a.802.802 0 0 1 0 1.392l-6.363 3.692C4.713 12.69 4 12.345 4 11.692V4.308c0-.653.713-.998 1.233-.696z"
                />
              </svg>
            {:else}
              <svg
                xmlns="http://www.w3.org/2000/svg"
                width="20"
                height="20"
                fill="currentColor"
                class="bi bi-pause"
                viewBox="0 0 16 16"
                ><path
                  d="M6 3.5a.5.5 0 0 1 .5.5v8a.5.5 0 0 1-1 0V4a.5.5 0 0 1 .5-.5m4 0a.5.5 0 0 1 .5.5v8a.5.5 0 0 1-1 0V4a.5.5 0 0 1 .5-.5"
                /></svg
              >
            {/if}</Button
          ><Button
          variant="ghost"
          class="flex gap-4 justify-start"
          on:click={() => spotifyApi.skipToNext()}
        ><svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" fill="currentColor" class="bi bi-skip-end-fill" viewBox="0 0 16 16">
          <path d="M12.5 4a.5.5 0 0 0-1 0v3.248L5.233 3.612C4.693 3.3 4 3.678 4 4.308v7.384c0 .63.692 1.01 1.233.697L11.5 8.753V12a.5.5 0 0 0 1 0z"/>
        </svg></Button>
        </div>
      {/if}
    </div>
    <div class="flex relative flex-col gap-4 h-screen w-80">
     
      {#if playlistData}
        <div class="w-80">
          <Accordion.Root>
            {#each playlistData as playlist, playlistIndex}
              <Accordion.Item value=playlist.id class="border-b-0 h-12">
                <Accordion.Trigger>
                  <div class="flex gap-4 items-center z-50">
                    {console.log(playlist)}
                    {playlist.name}
                    <Button
                      variant="ghost"
                      on:click={() => {
                        playPlaylist(playlist);
                      }}
                      ><svg
                        xmlns="http://www.w3.org/2000/svg"
                        width="16"
                        height="16"
                        fill="currentColor"
                        class="bi bi-play"
                        viewBox="0 0 16 16"
                      >
                        <path
                          d="M10.804 8 5 4.633v6.734zm.792-.696a.802.802 0 0 1 0 1.392l-6.363 3.692C4.713 12.69 4 12.345 4 11.692V4.308c0-.653.713-.998 1.233-.696z"
                        />
                      </svg></Button
                    >
                  
                    <Button variant="ghost"
                      ><svg
                        xmlns="http://www.w3.org/2000/svg"
                        width="16"
                        height="16"
                        fill="currentColor"
                        class="bi bi-share"
                        viewBox="0 0 16 16"
                      >
                        <path
                          d="M13.5 1a1.5 1.5 0 1 0 0 3 1.5 1.5 0 0 0 0-3M11 2.5a2.5 2.5 0 1 1 .603 1.628l-6.718 3.12a2.5 2.5 0 0 1 0 1.504l6.718 3.12a2.5 2.5 0 1 1-.488.876l-6.718-3.12a2.5 2.5 0 1 1 0-3.256l6.718-3.12A2.5 2.5 0 0 1 11 2.5m-8.5 4a1.5 1.5 0 1 0 0 3 1.5 1.5 0 0 0 0-3m11 5.5a1.5 1.5 0 1 0 0 3 1.5 1.5 0 0 0 0-3"
                        />
                      </svg></Button
                    >
                  </div>
                </Accordion.Trigger>
                <Accordion.Content>
                  {#each playlist.tracks.items as playlistSong, index}
                  {playlistSong.name}
                  {/each}
                </Accordion.Content>
              </Accordion.Item>
            {/each}
          </Accordion.Root>
        </div>
      {:else}
        <p>No playlists found.</p>
      {/if}
      <div
        class="grid grid-col-1 gap-2 absolute bottom-48 h-40 w-80 text-neutral-600 z-10"
      >
        Next Up ({localQueue.length}) :
        <div class="grid gap-4">
          {#if localQueue.length > 0}
            {#each localQueue as queueItem, songIndex}
            <div class="item-center grid">
            <Button
              variant="ghost"
              class="flex gap-4 justify-start"
              on:click={() => playTrack(queueItem.uri, queueItem)}
            >
              <img
                src={queueItem.album.images[0].url}
                width="30"
                height="30"
                class="rounded-md"
              />
              <h2>{queueItem.name}</h2>
        </Button>
        <Button
              class="absolute right-0"
              variant="ghost"
              on:click={() => {
                localQueue = localQueue.filter(
                  (_, index) => index !== songIndex,
                );
              }}
              ><svg
                xmlns="http://www.w3.org/2000/svg"
                width="16"
                height="16"
                fill="black"
                class="bi bi-x"
                viewBox="0 0 16 16"
              >
                <path
                  d="M4.646 4.646a.5.5 0 0 1 .708 0L8 7.293l2.646-2.647a.5.5 0 0 1 .708.708L8.707 8l2.647 2.646a.5.5 0 0 1-.708.708L8 8.707l-2.646 2.647a.5.5 0 0 1-.708-.708L7.293 8 4.646 5.354a.5.5 0 0 1 0-.708"
                />
              </svg></Button
            >
            </div>
            {/each}
          {/if}
        </div>
      </div>
    </div>
  </div>
{/if}
