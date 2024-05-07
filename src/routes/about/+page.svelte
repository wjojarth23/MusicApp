<script>
  import { onMount } from "svelte";
  import { writable } from "svelte/store";
  import SpotifyWebApi from "spotify-web-api-js";
  import { Button } from "$lib/components/ui/button";
  import { Input } from "$lib/components/ui/input";
  import * as Accordion from "$lib/components/ui/accordion";
  import PocketBase from "pocketbase";
  const pb = new PocketBase("https://feb2024-team5.pockethost.io");
  const delay = (ms) => new Promise((res) => setTimeout(res, ms));
  let tracks = [];
  let searchQuery = "";
  let spotifyApi = new SpotifyWebApi();
  let loginState = false;
  let currentTrack = null;
  let isToggled = false;
  let player = null;
  let accessToken = null;
  let playlists = [];
  let localQueue = [];
  let isChanging = false;
  let selectedPlaylist = null;

  const clientId = "da22f1252f514bb59be6b5588fddaf25";
  const redirectUri =
    "https://4b5870aa-6a5a-490a-9f0a-02334b872cea-00-a5xkmb2gsb82.janeway.replit.dev/about/"; // Update with your redirect URI
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

  async function initializePlayer() {
    player = new Spotify.Player({
      name: "Web Playback SDK Quick Start Player",
      getOAuthToken: (cb) => {
        cb(accessToken);
      },
      volume: 0.5,
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
    });

    player.addListener("account_error", ({ message }) => {
      console.error(message);
    });
    player.addListener("player_state_changed", (state) => {
      if (state.position === 0 && isChanging == false) {
        if (localQueue.length > 0) {
          isChanging = true;
          let nextSong = localQueue[0];
          let options = {
            uris: [nextSong.uri],
          };
          //console.log(track.name);
          spotifyApi.play(options, (error, response) => {
            if (error) {
              console.error("Error starting playback:", error);
            } else {
              console.log("Playback started successfully:", response);
              //currentTrack = localQueue[1];
              //playstate = true;
              isToggled = false;
              localQueue = localQueue.slice(1);
              //setCurrentSong();
              console.log(nextSong);
              console.log(response);
              currentTrack = nextSong;
              waitReset();
              //delay(3000);
              //isChanging = false;
            }
          });
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
  onMount(async () => {
    let hashParams = window.location.hash;
    if (hashParams) {
      loginState = true;
      let searchParams = new URLSearchParams(hashParams.substring(1));

      // Get the access token
      accessToken = searchParams.get("access_token");
      if (accessToken) {
        console.log(typeof accessToken);

        // Log the access token
        console.log("Access Token: " + accessToken);
        await spotifyApi.setAccessToken(accessToken);
        console.log("queue...");
        await initializePlayer();

        const data = await spotifyApi.getUserPlaylists();
        playlists = data;
        await delay(3000);
        //getQueue();
        setCurrentSong();
        isToggled = true;
      }
    }
  });
  async function searchMusic() {
    let data = await spotifyApi.searchTracks(searchQuery);
    tracks = data.tracks.items;
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
        isToggled = false;
        waitReset();
      }
    });
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
      isToggled = false;
      let currentTrackId = await spotifyApi.getMyCurrentPlayingTrack();
      console.log(currentTrackId.item.id);
      currentTrack = await spotifyApi.getTrack(currentTrackId.item.id);
      console.log(currentTrack);
      console.log("queueupdate");

      //getQueue();
      //console.log(currentTrack.item)
    } catch (error) {
      console.error(
        "Failed to get playlist tracks or add them to the queue",
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
        {#each tracks as track (track.id)}
          <div>
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
              isToggled = !isToggled;
              player.togglePlay();
            }}
            >{#if isToggled}
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
          >
        </div>
      {/if}
    </div>
    <div class="flex relative flex-col gap-4 h-screen w-80">
      <Button class="w-80">Create Playlist</Button>
      {#if playlists.items}
        <div class="w-80">
          <Accordion.Root>
            {#each playlists.items as playlist (playlist.id)}
              <Accordion.Item>
                <Accordion.Header>
                  <div class="flex gap-4 items-center z-50">
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
                    <Button
                      variant="ghost"
                      on:click={() => (selectedPlaylist = playlist.id)}
                      ><svg
                        xmlns="http://www.w3.org/2000/svg"
                        width="16"
                        height="16"
                        fill="currentColor"
                        class="bi bi-music-note-list"
                        viewBox="0 0 16 16"
                      >
                        <path
                          d="M12 13c0 1.105-1.12 2-2.5 2S7 14.105 7 13s1.12-2 2.5-2 2.5.895 2.5 2"
                        />
                        <path fill-rule="evenodd" d="M12 3v10h-1V3z" />
                        <path
                          d="M11 2.82a1 1 0 0 1 .804-.98l3-.6A1 1 0 0 1 16 2.22V4l-5 1z"
                        />
                        <path
                          fill-rule="evenodd"
                          d="M0 11.5a.5.5 0 0 1 .5-.5H4a.5.5 0 0 1 0 1H.5a.5.5 0 0 1-.5-.5m0-4A.5.5 0 0 1 .5 7H8a.5.5 0 0 1 0 1H.5a.5.5 0 0 1-.5-.5m0-4A.5.5 0 0 1 .5 3H8a.5.5 0 0 1 0 1H.5a.5.5 0 0 1-.5-.5"
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
                </Accordion.Header>
                <Accordion.Content>
                  <div>selected</div>
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
              <div class="flex gap-2 w-80 h-12 items-center">
                <img
                  src={queueItem.album.images[0].url}
                  width="25"
                  height="25"
                  class="rounded-md"
                />{queueItem.name}
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
