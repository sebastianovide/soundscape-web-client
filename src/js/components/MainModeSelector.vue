<template>
  <button
    class="nav-button"
    id="btn_callouts"
    title="Start tracking"
    :class="activeMode == 'callouts' ? 'active' : ''"
    @click="toggleMode('callouts')"
  >
    <i class='fas'>&#xf124;</i>
  </button>
  <button
    class="nav-button"
    id="btn_near_me"
    title="Announce places near me"
    :class="activeMode == 'near_me' ? 'active' : ''"
    @click="toggleMode('near_me')"
  >
    ⓘ
  </button>
</template>

<script setup>
import mode_exit_wav from "/sounds/mode_exit.wav";
import mode_enter_wav from "/sounds/mode_enter.wav";

import { inject, ref } from 'vue';
import { playSpatialSpeech } from '../audio/sound.js';
import { getLocation, watchLocation } from "../spatial/geo.js";
import { startCompassListener } from "../spatial/heading.js";

const announcer = inject('announcer');
const audioQueue = inject('audioQueue');
const locationProvider = inject('locationProvider');

const activeMode = ref(null);
var watchPositionHandler = null;
var wakeLock = null;

// Use location from URL if specified, otherwise use device location services
async function getRelevantLocation() {
  return new Promise((resolve, reject) => {
    var searchParams = new URLSearchParams(window.location.search);
    var lat = parseFloat(searchParams.get("lat"));
    var lon = parseFloat(searchParams.get("lon"));
    var head = parseFloat(searchParams.get("heading"));
    if (!isNaN(lat) && !isNaN(lon) && !isNaN(head)) {
      resolve({ latitude: lat, longitude: lon, heading: head });
    } else {
      getLocation()
        .then((coords) => {
          resolve({
            latitude: coords.latitude,
            longitude: coords.longitude,
            heading: 0, // only available from change event handler?
          });
        })
        .catch((error) => {
          reject(error);
        });
    }
  });
}

// When mode button is clicked:
//   If a mode is currently active, end that mode
//   If mode button was different from current mode, start new mode
async function toggleMode(newMode) {
  // required for iOS Safari: first speech must be directly triggered by user action
  playSpatialSpeech(" ");

  // Clear queued audio
  if (activeMode.value) {
    audioQueue.stopAndClear();
    audioQueue.addToQueue({ soundUrl: mode_exit_wav });
  }

  // Remove any location and orientation change event handlers
  locationProvider.events.removeEventListener(
    "locationUpdated",
    announcer.locationChanged
  );
  if (watchPositionHandler) {
    navigator.geolocation.clearWatch(watchPositionHandler);
    window.removeEventListener(
      "deviceorientation",
      locationProvider.updateOrientation
    );
    watchPositionHandler = null;
  }

  // Stop here if the intent was to end the current mode
  if (activeMode.value == newMode) {
    activeMode.value = null;

    if (wakeLock) {
      // Release the Wake Lock
      wakeLock.release().then(() => {
        wakeLock = null;
      });
    }

    return;
  }

  // Button clicked was different from current mode -- start new mode
  activeMode.value = newMode;

  // Request a Wake Lock
  if ("wakeLock" in navigator && !wakeLock) {
    try {
      wakeLock = await navigator.wakeLock.request("screen");
      console.log("Wake Lock is active!");

      // Listen for Wake Lock being released
      //   (when no active modes OR when page loses focus)
      wakeLock.addEventListener("release", () => {
        console.log("Wake Lock released.");
      });
    } catch (err) {
      // The Wake Lock request has failed - usually system related, such as battery.
      console.log(err);
    }
  }

  // play mode-enter sound
  audioQueue.addToQueue({ soundUrl: mode_enter_wav });

  switch (newMode) {
    case "callouts":
      locationProvider.events.addEventListener(
        "locationUpdated",
        announcer.locationChanged
      );
      startCompassListener(locationProvider.updateOrientation);

      watchPositionHandler = watchLocation(locationProvider.updateLocation);
      break;

    case "near_me":
      getRelevantLocation()
        .then((coords) => {
          console.log(coords);
          locationProvider.updateLocation(coords.latitude, coords.longitude);
          locationProvider.updateOrientation({ alpha: coords.heading });

          // Speak nearest road
          announcer.calloutNearestRoad(coords.latitude, coords.longitude);

          // Call out nearby features once
          announcer
            .calloutAllFeatures(coords.latitude, coords.longitude)
            .then((anythingToSay) => {
              if (!anythingToSay) {
                audioQueue.addToQueue({
                  text: "Nothing to call out right now",
                });
              }
            });
        })
        .catch((error) => {
          if (error.code == error.PERMISSION_DENIED) {
            alert(
              "Could not get your location. If you did not see a permission request, make sure your browser is not configured to always block location services."
            );
          } else {
            console.error("Error getting current position: " + error.message);
          }
        });
      break;
  }
}

// Reacquire Wake Lock when page regains focus
document.addEventListener("visibilitychange", async () => {
  if (wakeLock !== null && document.visibilityState === "visible") {
    wakeLock = await navigator.wakeLock.request("screen");
    console.log("Wake Lock reacquired.");
  }
});
</script>

<style scoped>
button {
  background-color: #e74c3c;
}

button.active {
  background-color: #18b3c3;
}
</style>