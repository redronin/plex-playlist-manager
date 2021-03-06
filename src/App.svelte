<script>
  import interact from 'interactjs';
  import {onMount} from 'svelte';
  import {plexToken, plexUser, plexLibraries, plexPlaylists, currLibrary, currPlaylist} from './lib/stores.js';

  import Navbar  from './partials/Navbar.svelte';
  import Sidebar from './partials/Sidebar.svelte';
  import WelcomePage from './WelcomePage.svelte';
  import LibraryPanel from './LibraryPanel.svelte';
  import PlaylistPanel from './PlaylistPanel.svelte';
  const Plex = document.plex; // only for console access

  $: sidepanel = false;
  $: {
    if ($plexToken != null) {
      if ($currLibrary == null) {
        loadLibraries();
        loadPlaylists();
      } 
    }
  }
  $: {
    if ($plexPlaylists != null) {
      updateCurrentPlaylist();
    }
  };

  function updateCurrentPlaylist() {
    if ($currPlaylist == null ) { return }
    for (let i=0; i < $plexPlaylists.length; i++) {
      if ($currPlaylist.ratingKey === $plexPlaylists[i].ratingKey) {
        console.log("updated current playlist")
        $currPlaylist = $plexPlaylists[i];
      }
    }
  };

  // login - login and store credentials
  async function login(event) {
    console.log("[App] Log in ", event);
    let email = event.detail.email;
    let password = event.detail.password;
    await Plex.Server.login(email, password);
    initPlex();
    loadLibraries();
  };

  // logout - clear credentials, clear cache
  async function logout() {
    Plex.Server.logout();
    initPlex();
  };

  async function initPlex() {
    console.log("[App] Reset and init stores.js from Plex.params")
    $plexLibraries = [];
    $currLibrary   = null;
    loadPlex();
  };

  // Load stores.js from Plex.params
  function loadPlex() {
    $plexToken = Plex.token;
    $plexUser  = Plex.params();
  }

  async function loadLibraries() {
    if ($plexToken == null) return;
    let data = await Plex.Library.all();
    $plexLibraries = data;
    $currLibrary = $plexLibraries[0];
  };

  async function loadPlaylists() {
    if ($plexToken == null) return;
    let resp = await Plex.Playlist.all();
    $plexPlaylists = resp.Metadata;
  };


  onMount(() => {
    initPlex();
    interact('.mediaCell').draggable({
      hold: 0,  // need for ipad when having scrollable content
      inertia: {
        smoothEndDuration: 200,
        resistance: 10,
        minSpeed: 200,
        endSpeed: 100,
      },
      manualStart: true,
      listeners: {
        start (event) {
          let el = event.target;
          el.classList.add('active');
          el.classList.remove('tapped');
          el.classList.remove('doubletapped');
        },
        move (event) {
          let el = event.target;
          let x = (parseFloat(el.getAttribute('data-x')) || 0) + event.dx;
          let y = (parseFloat(el.getAttribute('data-y')) || 0) + event.dy;
          el.style.transform = `translate(${x}px, ${y}px)`;
          el.setAttribute('data-x',x);
          el.setAttribute('data-y',y);
        },
        end(event) {
          let el = event.target;
          el.remove();
        }
      }
    }).on('move', function(event){
      const { currentTarget, interaction } = event;
      let element = currentTarget;

      // If we are dragging an item from the sidebar, its transform value will be ''
      // We need to clone it, and then start moving the clone
      if (
        interaction.pointerIsDown &&
        !interaction.interacting() &&
        currentTarget.style.transform === ""
      ) {
        element = currentTarget.cloneNode(true);

        let domRect = currentTarget.getBoundingClientRect();
        // Add absolute positioning so that cloned object lives
        // right on top of the original object
        element.style.position = "absolute";
        element.style.zIndex = 9999;
        element.style.left = `${domRect.left}px`;
        element.style.top = `${domRect.top}px`;

        // Add the cloned object to the document
        const container = document.body;
        container && container.appendChild(element);
      }

      // Start the drag event
      interaction.start({ name: "drag" }, event.interactable, element);
    });

    interact('.mediaCell').on('tap', (e) => {
      e.currentTarget.classList.toggle('tapped');      
      e.currentTarget.classList.remove('doubletapped');
    });
    interact('.mediaCell').on('doubletap', (e) => {
      e.currentTarget.classList.toggle('doubletapped');
      e.currentTarget.classList.remove('tapped');
    });
    interact('.droppable').dropzone({
      accept: '.mediaCell',
      overlap: 'pointer',
      listeners: {
        dropactivate (event) {
        },
        dropdeactivate (event) {
        },
        dragenter(event) { 
          event.target.classList.add('active-drop');
        },
        dragleave(event) {
          event.target.classList.remove('active-drop');
        },
        drop(event) {
          let mediaEl = event.relatedTarget;
          let listEl  = event.target;
          listEl.classList.remove('active-drop');
          listEl.classList.add('dropped');
          setTimeout(() => listEl.classList.remove('dropped'), 600);          
          addToPlaylist(listEl.dataset.id, mediaEl.dataset.id);
        }
      }
    });    
  });

  async function addToPlaylist(playlistId, mediaId) {
    let resp = await Plex.Playlist.addItem(playlistId, mediaId);
    let playlist = resp.Metadata[0];
    for (let i=0;i<$plexPlaylists.length; i++) {
      if ($plexPlaylists[i].ratingKey === playlist.ratingKey) {
        $plexPlaylists[i] = playlist;
      }
    }
    $plexPlaylists = [...$plexPlaylists];
  };

  function toggleSidePanel() {
    sidepanel = !sidepanel;
  };

  async function createPlaylist() {
    let resp = await Plex.Playlist.create("New Playlist", "");
    $currPlaylist = resp.Metadata[0];
    await loadPlaylists();
  };

</script>

<Navbar on:logout={logout} />
{#if $plexToken}
<Sidebar on:newPlaylist={createPlaylist} />
{/if}
<div class='playlist-panel' class:active={$currPlaylist != null}>
  {#if ($currPlaylist != null)}
  <PlaylistPanel playlist={$currPlaylist} on:loadPlaylists={loadPlaylists} />
  {/if}
</div>

{#if ($plexToken != null) }
  {#if ($currLibrary != null)}      
    <div class='library-panel' class:active={$currPlaylist != null}>
      <LibraryPanel library={$currLibrary} />
    </div>
  {/if}
{:else}
  <WelcomePage on:login={login} />
{/if}


<style lang='scss'>

</style>
