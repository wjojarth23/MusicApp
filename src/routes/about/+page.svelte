<script>
  import { onMount } from "svelte";
  import { writable } from "svelte/store";
  import SpotifyWebApi from "spotify-web-api-js";
  let tracks = [];
  let searchQuery = "";
  let spotifyApi = new SpotifyWebApi();
  const clientId = "da22f1252f514bb59be6b5588fddaf25";
  const redirectUri =
    "https://4b5870aa-6a5a-490a-9f0a-02334b872cea-00-a5xkmb2gsb82.janeway.replit.dev/about/"; // Update with your redirect URI
  const scopes =
    "user-read-private user-read-playback-state user-modify-playback-state user-read-currently-playing streaming"; // Adjust scopes as needed

  const loginUrl = new URL("https://accounts.spotify.com/authorize");
  loginUrl.searchParams.set("client_id", clientId);
  loginUrl.searchParams.set("redirect_uri", redirectUri);
  loginUrl.searchParams.set("scope", scopes);
  loginUrl.searchParams.set("response_type", "token");

  onMount(() => {
    let hashParams = window.location.hash;
    if (hashParams) {
      let searchParams = new URLSearchParams(hashParams.substring(1));

      // Get the access token
      let accessToken = searchParams.get("access_token");
      console.log(typeof accessToken);

      // Log the access token
      console.log("Access Token: " + accessToken);
      spotifyApi.setAccessToken(accessToken);
      console.log(spotifyApi.getAccessToken());
    }
  });
  async function searchMusic() {
    let data = await spotifyApi.searchTracks(searchQuery);
    tracks = data.tracks.items;
  }
  async function playTrack(uri) {
    let options = {
      uris: [uri],
    };

    spotifyApi.play(options, (error, response) => {
      if (error) {
        console.error("Error starting playback:", error);
      } else {
        console.log("Playback started successfully:", response);
      }
    });
  }
</script>

<h1>Login with Spotify</h1>

<a href={loginUrl.toString()}>Login to Spotify</a>
<input bind:value={searchQuery} placeholder="Search for music" />
<button on:click={searchMusic}>Search</button>

<ul>
  {#each tracks as track (track.id)}
    <li>
      <h2>{track.uri}</h2>
      <p>{track.artists[0].name}</p>
      <button on:click={() => playTrack(track.uri)}>Play</button>
    </li>
  {/each}
</ul>
<body> </body>
