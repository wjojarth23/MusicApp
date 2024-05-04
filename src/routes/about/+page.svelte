<script>
  import { onMount } from "svelte";
  import { writable } from "svelte/store";
  import SpotifyWebApi from "spotify-web-api-js";
  import { Button } from "$lib/components/ui/button";
  import { Input } from "$lib/components/ui/input";
  let tracks = [];
  let searchQuery = "";
  let spotifyApi = new SpotifyWebApi();
  let loginState = false;
  let currentTrack = null;
  let isToggled = false;
  let player = null;
  let playlists = [];
  const clientId = "da22f1252f514bb59be6b5588fddaf25";
  const redirectUri =
    "https://4b5870aa-6a5a-490a-9f0a-02334b872cea-00-a5xkmb2gsb82.janeway.replit.dev/about/"; // Update with your redirect URI
  const scopes =
    "user-read-private user-read-playback-state user-modify-playback-state user-read-currently-playing streaming app-remote-control user-read-email"; // Adjust scopes as needed

  const loginUrl = new URL("https://accounts.spotify.com/authorize");
  function loadPlaylists(value) {
    console.log("value", value);
  }
  loginUrl.searchParams.set("client_id", clientId);
  loginUrl.searchParams.set("redirect_uri", redirectUri);
  loginUrl.searchParams.set("scope", scopes);
  loginUrl.searchParams.set("response_type", "token");

  onMount(async () => {
    let hashParams = window.location.hash;
    if (hashParams) {
      loginState = true;
      let searchParams = new URLSearchParams(hashParams.substring(1));

      // Get the access token
      let accessToken = searchParams.get("access_token");
      console.log(typeof accessToken);

      // Log the access token
      console.log("Access Token: " + accessToken);
      spotifyApi.setAccessToken(accessToken);

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

      player.connect();
    }

    try {
      const data = await spotifyApi.getUserPlaylists();
      playlists = data;
    } catch (error) {
      console.error("Error fetching playlists:", error);
    }
  });
  async function searchMusic() {
    let data = await spotifyApi.searchTracks(searchQuery);
    tracks = data.tracks.items;
  }

  async function playTrack(uri, track) {
    let options = {
      uris: [uri],
    };

    spotifyApi.play(options, (error, response) => {
      if (error) {
        console.error("Error starting playback:", error);
      } else {
        console.log("Playback started successfully:", response);
        currentTrack = track;
        playstate = true;
      }
    });
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
    <div class="grid gap-4 w-2/3">
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
          class="flex fixed gap-6 left-0 bg-neutral-100 bottom-0 w-full h-20 outline-slate-200 p-4 justify-center items-center"
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
    <div class="flex flex-col">
      <Button>Create Playlist</Button>{#if playlists.items.length > 0}
        <ul>
          {#each playlists.items as playlist (playlist.id)}
            <li>{playlist.name}</li>
          {/each}
        </ul>
      {:else}
        <p>No playlists found.</p>
      {/if}
    </div>
  </div>
{/if}

<style>
  *:focus {
    outline: none;
  }
</style>
