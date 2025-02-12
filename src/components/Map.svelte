<!-- This Source Code Form is subject to the terms of the Mozilla Public
   - License, v. 2.0. If a copy of the MPL was not distributed with this
   - file, You can obtain one at https://mozilla.org/MPL/2.0/. -->
<script>
  export let currentRoute;
  export let params;
  import Sidebar from "./Sidebar.svelte";
  import {
    highlighted_territories,
    lock_highlighted,
    map_type,
    sidebarOpen,
    turn,
    turns,
    modal,
    team_territory_counts,
    user,
    prompt_move,
    my_move,
  } from "../state/state.js";

  import { settings } from "../state/settings";

  import MapBase from "./MapBase.svelte";
  import { bind } from "svelte-simple-modal";
  import Leaderboard from "./Leaderboard.svelte";
  import {
    faHistory,
    faShip,
    faSearchMinus,
    faSearchPlus,
    faFlag,
    faMap,
    faThermometerHalf,
    faRankingStar,
    faBullseye,
    faChevronUp,
    faChevronDown,
  } from "@fortawesome/free-solid-svg-icons";
  import { FontAwesomeIcon } from "fontawesome-svelte";
  import { onDestroy, onMount } from "svelte";
  import { getDay, getTurnInfo } from "../utils/loads.js";
  import { normalizeTerritoryName } from "../utils/normalization.js";
  import MyMove from "./MyMove.svelte";
  import Clock from "./Clock.svelte";
  import { setupMapPanZoom } from "../utils/map";
  import TerritoryTurn from "./TerritoryTurn.svelte";
  import { getMove } from "../utils/auth";
  var lockClick = false;
  var zooming = false;
  var bottomMenu = faChevronUp;
  var mapLoaded = false;
  var showMyMoveNextDraw = false;

  const showMove = () => {
    if ($user != null) {
      modal.set(bind(MyMove, { territoryInfo: territoryInfo }));
    } else {
      modal.set(bind(Login));
    }
  };

  onMount(() => {
    if (currentRoute.hash.indexOf("#leaderboard") != -1) {
      showLeaderboard(params);
    } else if (currentRoute.hash.indexOf("#MyMove") != -1) {
      if (mapLoaded) {
        showMove();
      } else {
        showMyMoveNextDraw = true;
      }
    }
    if (currentRoute.hash.indexOf("#history_") != -1) {
      //try {
      console.log(currentRoute.hash);
      let matches = decodeURIComponent(currentRoute.hash).match(
        /#history_([A-z 0-9%]*)_([0-9]*)_([0-9]*)/
      );
      console.log(matches);
      modal.set(
        bind(TerritoryTurn, {
          territory: matches[1],
          season: matches[2],
          day: matches[3],
        })
      );
      /*} catch {
        console.log("Attempted to paint History, but failed");
      }*/
    }
    setupMapPanZoom(handleMouseOver, handleWindowKeyDown);
    window.maphandle.on("panend", function () {
      lockClick = false;
    });
    // Set up regions at start according to the default settings
    document.getElementById("Regions").style.display = $settings.regions_default
      ? "flex"
      : "none";
    // Set up bridges at start according to the default settings
    document.getElementById("Bridges").style.display = $settings.bridges_default
      ? "flex"
      : "none";
  });

  function handleWindowKeyDown(event) {
    if (event.key === "Escape") {
      lock_highlighted.set(false);
      $highlighted_territories.style.fill =
        $highlighted_territories.info.primaryColor;
    }
  }

  function handleMouseOver(e) {
    if (
      !lockClick &&
      (e.type == "click" ||
        e.type == "mousedown" ||
        (e.type == "touchend" && !zooming)) &&
      $lock_highlighted &&
      e.target == document.getElementById("map")
    ) {
      lock_highlighted.set(false);
      try {
        $highlighted_territories.style.fill =
          $highlighted_territories.info.primaryColor;
        highlighted_territories.set(null);
      } catch {
        // We know that this failed, but don't throw an error.
      }
      return;
    }
    if (e.target.info == undefined) return;
    if (e.type == "mouseover") {
      if ($lock_highlighted) return;
      try {
        e.target.style.fill = e.target.info.secondaryColor;
        highlighted_territories.set(e.target);
      } catch {
        // We know that this failed, but don't throw an error.
      }
    } else if (e.type == "mouseout") {
      if ($lock_highlighted) return;
      try {
        e.target.style.fill = e.target.info.primaryColor;
        highlighted_territories.set(null);
      } catch {
        // We know that this failed, but don't throw an error.
      }
    } else if (
      !lockClick &&
      (e.type == "click" ||
        e.type == "mousedown" ||
        (e.type == "touchend" && !zooming))
    ) {
      try {
        $highlighted_territories.style.fill =
          $highlighted_territories.info.primaryColor;
      } catch {
        // This is an issue :(
      }
      sidebarOpen.set(true);
      lock_highlighted.set(true);
      if (e.type == "touchend") {
        var changedTouch = e.changedTouches[0];
        var elem = document.elementFromPoint(
          changedTouch.clientX,
          changedTouch.clientY
        );
      } else {
        var elem = e.target;
      }
      try {
        elem.style.fill = elem.info.secondaryColor;
        highlighted_territories.set(elem);
      } catch {
        // Page not ready yet
      }
    }
  }
  const showLeaderboard = () =>
    modal.set(bind(Leaderboard, { turnToUse: $turn }));

  onDestroy(() => {
    map_type.set("owners");
    highlighted_territories.set(null);
  });
  var territoryInfo;

  var preloadedMaps = {};
  async function preloadMaps() {
    let turn = await getTurnInfo(null);
    let season = turn.season;
    let totalTurns = $turns.filter((a) => a.season === season);
    for (var i = 0; i < totalTurns.length; i++) {
      console.log(`Fetching turn ${i + 1} of ${totalTurns.length}`);
      if ($turns[i].season === season) {
        preloadedMaps[i] = await getDay($turns[i].id);
      }
    }
  }

  async function loadMap() {
    try {
      // Clear the map visually
      document.querySelectorAll("#map #Territories path").forEach((e) => {
        e.info = null;
        e.style.fill = "rgba(128, 128, 128, 0)";
      });
      mapLoaded = false;
      // Fetch territory ownership
      territoryInfo = await getDay($turn);
      recolorMap(territoryInfo);
      mapLoaded = true;
      if (showMyMoveNextDraw) {
        showMyMoveNextDraw = false;
        showMove();
      }
    } catch (e) {
      console.log(`Map failed, reason: ${e}`);
    }
  }

  async function recolorMap(territoryInfo) {
    var owners = {};
    territoryInfo.forEach((terr) => {
      if ($map_type == "owners") {
        if (owners[terr.attributeInformation.owner] == null) {
          owners[terr.attributeInformation.owner] = 0;
        }
        owners[terr.attributeInformation.owner]++;
      }
      if (terr.name == "Bermuda" && $map_type == "owners") {
        // Yeet old lines
        document.querySelectorAll("[chaos=true]").forEach(function (node) {
          node.parentNode.removeChild(node);
        });
        try {
          terr.attributeInformation.neighbors.filter(function (obj) {
            return drawChaosLine(normalizeTerritoryName(obj.name));
          });
        } catch {
          // Bermuda doesn't have any neighbors...
          console.log("Bermuda does not have any neighbors");
        }
      }
      try {
        document
          .querySelector("#map")
          .querySelector(`path#${terr.normalizedName}`).style.fill =
          terr.primaryColor;
        document
          .querySelector("#map")
          .querySelector(`path#${terr.normalizedName}`).info = terr;
      } catch {
        console.log(`Couldn't find territory ${terr.normalizedName}`);
      }
    });
    team_territory_counts.set(owners);
    try {
      $highlighted_territories.style.fill =
        $highlighted_territories.info.secondaryColor;
    } catch {
      // Page not ready yet
    }
  }

  function toggleRegions() {
    document.getElementById("Regions").style.display =
      document.getElementById("Regions").style.display == "none"
        ? "flex"
        : "none";
  }

  function toggleBridges() {
    document.getElementById("Bridges").style.display =
      document.getElementById("Bridges").style.display == "none"
        ? "flex"
        : "none";
  }

  function toggleBottomMenuMobile() {
    document
      .getElementById("map-controls-bottom")
      .classList.toggle("hideOnMobile");
    bottomMenu = bottomMenu == faChevronUp ? faChevronDown : faChevronUp;
  }

  function pulseTerritory(territory, map_is_loaded) {
    try {
      if (
        $settings.pulse_territories != true ||
        !map_is_loaded ||
        document.getElementById("map") === null
      )
        return;
      let pulsing = document
        .querySelector("#map")
        .getElementsByClassName("map-animated-highlight");
      for (var i = 0; i < pulsing.length; i++) {
        pulsing[i].classList.remove("map-animated-highlight");
      }
      if (territory == null || territory === '""') return;
      document
        .querySelector("#map")
        .getElementById(normalizeTerritoryName(territory.replace(/['"]+/g, "")))
        .classList.add("map-animated-highlight");
    } catch (e) {
      console.log(`Failed flashing territory ${e}`);
    }
  }
  async function drawPin(territory, map_is_loaded) {
    try {
      if (
        $settings.pin_move != true ||
        !map_is_loaded ||
        document.getElementById("map") === null
      )
        return;
      let pin = document.getElementById("map").getElementById("move_pin");
      if (territory == null || territory === '""') {
        pin.style.display = "none";
        return;
      }
      let scale = 2;
      var terr = document
        .querySelector("#map")
        .getElementById(normalizeTerritoryName(territory.replace(/['"]+/g, "")))
        .getBBox();
      pin.style.display = "block";
      var pin_bbox = pin.getBBox();
      let x_offset =
        (terr.x + 0.5 * terr.width) / scale - pin_bbox.width / scale;
      let y_offset = (terr.y + 0.5 * terr.height) / scale - pin_bbox.height;
      pin.setAttribute(
        "transform",
        `scale(${scale}) translate(${x_offset}, ${y_offset})`
      );
    } catch (e) {
      console.log(`Error pinning: ${e}`);
    }
  }
  async function drawChaosLine(territory_name) {
    var end = document
      .getElementById("map")
      .getElementById("Bermuda")
      .getBBox();
    var start = document
      .getElementById("map")
      .getElementById(territory_name)
      .getBBox();
    var start_y = start.y + 0.5 * start.height;
    var start_x = start.x + 0.5 * start.width;
    var end_y = end.y + 0.5 * end.height;
    var end_x = end.x + 0.5 * end.width;
    var newpath = document.createElementNS(
      "http://www.w3.org/2000/svg",
      "path"
    );
    newpath.setAttributeNS(
      null,
      "d",
      `M ${start_x},${start_y} ${end_x},${end_y}`
    );
    newpath.setAttributeNS(
      null,
      "style",
      `fill:none;stroke:var(--main-foreground-color);stroke-width:1px;stroke-linecap:butt;stroke-linejoin:miter;stroke-opacity:1;`
    );
    newpath.setAttributeNS(null, "chaos", "true");
    document
      .getElementById("map")
      .getElementById("Bridges")
      .appendChild(newpath);
  }

  async function promptMoveCheck(user) {
    if (user == null) return;
    if (
      $settings.prompt_move == false &&
      $settings.pulse_territories == false
    ) {
      $prompt_move = false;
      return;
    }
    let move = await getMove();
    if ($settings.pulse_territories && $turn == null && move != "") {
      pulseTerritory(move);
    }
    if (move == '""' && $settings.prompt_move == true) {
      $prompt_move = true;
    } else {
      $prompt_move = false;
    }
  }

  $: promptMoveCheck($user);
  $: loadMap($turn, $map_type);
  $: drawPin($my_move, mapLoaded);
  $: pulseTerritory($my_move, mapLoaded);
</script>

<Sidebar {territoryInfo} />
<div class="map-container">
  <div class="map-controls">
    <div
      id="map-controls-bottom"
      class="map-controls map-controls-bottom hideOnMobile"
    >
      <!--     {#if $prompt_move}
        <center class="note"
          >Click <b style="font-size:0.5em">&#127919;</b> to submit your move
          <b style="font-size:0.5em">&#10549;</b></center
        >
      {/if}-->
      <button onclick="window.maphandle.zoomTo(500, 500, 1.5);" title="zoom in">
        <FontAwesomeIcon icon={faSearchPlus} />
        <b class={$settings.show_labels ? "" : "hidden"}>Zoom In</b>
      </button>
      <button
        onclick="window.maphandle.zoomTo(500, 500, 0.75);"
        title="zoom out"
      >
        <FontAwesomeIcon icon={faSearchMinus} />
        <b class={$settings.show_labels ? "" : "hidden"}> Zoom Out </b>
      </button>
      <button
        onclick="window.maphandle.moveTo(0,0); window.maphandle.zoomTo(0, 0, 1/window.maphandle.getTransform()['scale']);"
        title="reset map"
      >
        <FontAwesomeIcon icon={faHistory} />
        <b class={$settings.show_labels ? "" : "hidden"}> Reset Map </b>
      </button>
      <button on:click={toggleRegions} title="regions">
        <FontAwesomeIcon icon={faFlag} />
        <b class={$settings.show_labels ? "" : "hidden"}>Regions</b>
      </button>
      <!--{#key $user}
        {#if $user != null}
          <a on:click={showMove} href="#MyMove">
            <button
              title="your move"
              style:animation={$prompt_move
                ? "5000ms ease-in-out infinite color-change"
                : ""}
              class="hideOnMobile"
            >
              <FontAwesomeIcon icon={faBullseye} />
              <b class={$settings.show_labels ? "" : "hidden"}> Your Move </b>
            </button>
          </a>
        {/if}
      {/key}-->
      <button on:click={toggleBridges} title="bridges">
        <FontAwesomeIcon icon={faShip} />
        <b class={$settings.show_labels ? "" : "hidden"}>Bridges</b>
      </button>
      <button on:click={showLeaderboard} title="leaderboard">
        <FontAwesomeIcon icon={faRankingStar} />
        <b class={$settings.show_labels ? "" : "hidden"}> Leaderboard </b>
      </button>
    </div>
    <div class="map-controls map-controls-mobile-bottom showOnMobile">
      <button on:click={toggleBottomMenuMobile} title="Expand Controls Menu">
        <FontAwesomeIcon icon={bottomMenu} /><b
          class={$settings.show_labels ? "" : "hidden"}
        >
          &nbsp;Controls
        </b>
      </button>
      <!--{#key $user}
        {#if $user != null}
          <a href="#MyMove" on:click={showMove}>
            <button
              title="your move"
              style:animation={$prompt_move
                ? "5000ms ease-in-out infinite color-change"
                : ""}
              style="margin-left:0.3em;"
            >
              <FontAwesomeIcon icon={faBullseye} />
              <b class={$settings.show_labels ? "" : "hidden"}>
                &nbsp;Your Move
              </b>
            </button>
          </a>
        {/if}
      {/key}-->
    </div>
  </div>
  <div class="map-controls top-control">
    <label
      title="owners map"
      style:background={`${
        $map_type == "owners" ? "var(--accent-1)" : "var(--accent-2)"
      }`}
    >
      <input bind:group={$map_type} type="radio" value={"owners"} />
      <FontAwesomeIcon icon={faMap} /><b class="hideOnMobile"
        ><b class={$settings.show_labels ? "" : "hidden"}> &nbsp;Owners </b></b
      >
    </label>
    <select bind:value={$turn} title="select day">
      <option value={null}
        ><FontAwesomeIcon icon={faThermometerHalf} />Latest</option
      >
      {#each $turns.slice(1, $turns.length) as day}
        <option value={day.id}>{day.season}/{day.day}</option>
      {/each}
    </select>
    <label
      title="heat map"
      style:background={`${
        $map_type == "heat" ? "var(--accent-1)" : "var(--accent-2)"
      }`}
    >
      <input bind:group={$map_type} type="radio" value={"heat"} />
      <b class="hideOnMobile"
        ><b class={$settings.show_labels ? "" : "hidden"}> Heatmap&nbsp; </b></b
      ><FontAwesomeIcon icon={faThermometerHalf} />
    </label>
  </div>
  <div class="notices"><Clock /></div>
  <div class="map-wrap"><MapBase /></div>
</div>

<style>
  .notices {
    color: var(--main-foreground-color);
    position: absolute;
    top: calc(var(--navbar-height));
    right: 0%;
    margin: 5px;
    z-index: 2;
    white-space: nowrap;
  }
  .map-wrap {
    overflow: hidden;
  }
  .top-control {
    position: absolute;
    top: calc(var(--navbar-height));
    margin-left: auto;
    margin-right: auto;
    left: 50%;
    transform: translate(-50%, 0%);
    z-index: 3;
    white-space: nowrap;
    bottom: unset !important;
  }

  .map-controls {
    position: absolute;
    bottom: calc(0.1 * var(--navbar-height));
    margin-left: auto;
    margin-right: auto;
    left: 50%;
    transform: translate(-50%, 0%);
    z-index: 3;
    white-space: nowrap;
  }

  .map-controls button,
  .map-controls label,
  .map-controls select {
    color: var(--accent-fg);
    background: var(--accent-2);
    font-weight: bold;
    height: 2em;
    border: none;
    padding: 0.3em;
    font-size: 1.3em;
    line-height: 1em;
  }

  .map-controls button,
  .map-controls label {
    min-width: 2em;
    border-radius: 0.3em;
    cursor: pointer;
  }

  .map-controls button:hover,
  .map-controls label:hover,
  .map-controls button:focus,
  .map-controls label:focus {
    background: var(--accent-1);
  }

  .map-controls input[type="radio"] {
    display: none;
  }

  .map-controls select {
    float: left;
  }

  .map-controls label {
    cursor: pointer;
    float: left;
    padding: 0.3em;
    justify-content: center;
    display: flex;
    align-items: center;
  }

  .map-controls label:first-of-type {
    border-top-right-radius: 0%;
    border-bottom-right-radius: 0%;
  }

  .map-controls label:last-of-type {
    border-top-left-radius: 0%;
    border-bottom-left-radius: 0%;
  }

  :global(#map #Territories path) {
    cursor: pointer;
  }

  .map-container {
    width: 100%;
    height: 100%;
    overflow: hidden;
  }

  @keyframes -global-color-change {
    0% {
      background-color: teal;
    }
    20% {
      background-color: gold;
    }
    40% {
      background-color: indianred;
    }
    60% {
      background-color: violet;
    }
    80% {
      background-color: green;
    }
    100% {
      background-color: teal;
    }
  }

  :global(.map-animated-highlight) {
    animation: fading 4s infinite;
  }

  @keyframes fading {
    0% {
      opacity: 0.2;
    }
    50% {
      opacity: 1;
    }
    100% {
      opacity: 0.2;
    }
  }
  .showOnMobile {
    display: none;
  }
  .hideOnMobile {
    display: unset;
  }
  @media only screen and (max-width: 1000px) {
    .hideOnMobile {
      display: none !important;
    }
    .showOnMobile {
      display: flex;
    }
    .map-controls-bottom {
      display: flex;
      flex-direction: column;
      position: relative;
      bottom: 3.3em;
      max-height: 60vh;
      overflow: auto;
    }
    .map-controls-bottom button {
      padding: 0.5em;
      height: 2em !important;
    }
  }

  .hidden {
    display: none;
  }

  .map-controls a {
    text-decoration: none;
  }
</style>
