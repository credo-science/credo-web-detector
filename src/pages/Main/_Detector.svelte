<script lang="ts">
  import { onDestroy, onMount } from "svelte";
  import Select from "./Select/Select.svelte";
  import { CameraAnalyzer } from "../../api/camera/camera.analyzer";
  import { camera } from "../../api/camera/camera.store";
  import { geoposition } from "../../api/geolocation/location-store";
  import { WakeLocker } from "../../util/wake-locker";
  import type { CameraError } from "../../api/camera/camera-error";
  import type { Camera } from "../../api/camera/camera.store";
  import type { DetectionAlgorithmResult } from "../../api/camera/detection.algorithm";
  import { DetectionSaver } from "../../api/detection/detection-saver";
  import Timer from "./_Timer.svelte";
  import { throttle } from "../../util/throttle";
  import { fade } from "svelte/transition";
  import {
    framesAnalyzed,
    newParticleCaught,
    particlesCaught,
    playSound,
  } from "./detector.store";
  import FramesAnalyzed from "./_FramesAnalyzed.svelte";
  import ParticlesCaughtCounter from "./_ParticlesCaughtCounter.svelte";

  var audio = new Audio("assets/poof.mp3");

  const analyzer = new CameraAnalyzer();

  let dialog;
  let brightnessBar;

  let select: boolean = false;
  let isRunning = false;
  let brightness = 100;

  $: detect(isRunning, $camera);

  onMount(() => {
    geoposition.startWatchingPosition();
  });

  onDestroy(() => {
    analyzer.stop();
    camera.closeStream();
    geoposition.stopWatchingPosition();
  });

  const detect = (isRunning: boolean, camera: Camera) => {
    if (!camera.stream || camera.error) {
      WakeLocker.release();
    }
    if (isRunning) {
      if (camera.error) {
        handleError(camera.error);
        return;
      }
      if (camera.stream) {
        startAnalyzer(camera.stream);
        WakeLocker.request();
      }
    }
  };

  function handleError(error: CameraError) {
    isRunning = false;
    showDialog();
  }

  function showDialog() {
    stop();
    dialog.show();
  }

  function startAnalyzer(stream: MediaStream) {
    analyzer.start(analyzerCallBack, stream.getVideoTracks()[0]);
  }

  const updateBrightness = function (b) {
    brightness = b / 2.55;
  };

  const analyzerCallBack = (data: DetectionAlgorithmResult) => {
    framesAnalyzed.increment();

    throttle(updateBrightness, 250, data.brightness);

    if (data.particleImg) {
      if ($playSound) {
        audio.play();
      }
      particlesCaught.increment();
      newParticleCaught.set(true);
      // ring ring
      DetectionSaver.save(data.particleImg, $geoposition.position);
    }
  };

  function start() {
    if (!$camera.id) {
      showDialog();
      return;
    }
    camera.requestStream();
    isRunning = true;
  }

  function stop() {
    isRunning = false;
    analyzer.stop();
    camera.closeStream();
    brightness = 100;
  }
</script>

<style>
  sl-dialog {
    --width: 400px;
  }

  sl-dialog::part(body) {
    padding: 0;
  }

  sl-dialog::part(header) {
    display: none;
  }

  .card {
    display: flex;
    background: var(--sl-color-white);
    border-radius: var(--sl-border-radius-large);
    min-height: 13rem;
  }

  @media only screen and (max-width: 600px) {
    .card {
      flex-direction: column;
      border-radius: var(--sl-border-radius-large);
      margin-bottom: 1rem;
    }

    .control {
      position: fixed !important; /* bugfix */
      height: 2rem !important;
      bottom: 0;
      left: 0;
      right: 0;
      z-index: +100;
    }
  }

  @media only screen and (min-width: 601px) {
    .control {
      min-width: 7rem;
      border-top-left-radius: var(--sl-border-radius-large);
      border-bottom-left-radius: var(--sl-border-radius-large);
    }
  }

  .info {
    width: 100%;
    padding: var(--sl-spacing-large);
  }

  .control {
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
    left: -1px;
    padding: var(--sl-spacing-x-large);
    background: var(--sl-color-primary-15);
  }

  .settings {
    position: absolute;
    top: var(--sl-spacing-x-small);
    right: var(--sl-spacing-large);
    display: flex;
    justify-content: flex-end;
    font-size: 24px;
  }

  .text-icon {
    position: relative;
    top: 5px;
    color: var(--sl-color-primary-50);
    font-size: 20px;
  }

  .starting-spinner {
    position: relative;
    top: 3px;
    margin-right: 5px;
  }

  .indicator {
    font-size: 20px;
    font-weight: 700;
  }

  .indicator > sl-icon {
    position: relative;
    font-size: 24px;
    top: 5px;
    margin-right: 5px;
  }

  .status-icon {
    position: relative;
    top: 2px;
    margin-right: 5px;
  }
</style>

<sl-dialog
  in:fade={{ duration: 400 }}
  on:slShow={() => (select = true)}
  on:slHide={() => (select = false)}
  bind:this={dialog}
  noHeader="true">
  {#if select}
    <Select
      on:close={() => {
        dialog.hide();
      }} />
  {/if}
</sl-dialog>

<div class="card">

  <div class="control">
    {#if !$camera.stream && !isRunning}
      <sl-tooltip placement="top" content="Start">
        <sl-icon-button
          name="play-fill"
          type="primary"
          size="large"
          style="font-size: 2.5rem"
          circle
          on:click={() => start()} />
      </sl-tooltip>
    {/if}

    {#if !$camera.stream && isRunning}
      <sl-spinner style="--indicator-color: var(--sl-color-white);" />
    {/if}

    {#if $camera.stream}
      <sl-tooltip placement="top" content="Stop">
        <sl-icon-button
          name="stop-fill"
          type="primary"
          size="large"
          style="font-size: 2.5rem"
          circle
          on:click={() => stop()} />
      </sl-tooltip>
    {/if}
  </div>

  <div class="info">
    {#if !$camera.stream && !isRunning}
      <section in:fade>
        {#if $camera.id}
          <span class="settings">
            <sl-tooltip placement="left" content="Settings">
              <sl-icon-button
                on:click={() => {
                  dialog.show();
                }}
                name="gear-fill" />
            </sl-tooltip>
          </span>
          <h1>Everything set!</h1>
          <hr />
          <p>
            Press the
            <sl-icon class="text-icon" name="play-fill" />
            button to start detector, and catch some particles
          </p>
        {:else}
          <h1>Welcome!</h1>
          <hr />
          <p>
            Press the
            <sl-icon class="text-icon" name="play-fill" />
            button to set up your camera device
          </p>
        {/if}
      </section>
    {/if}

    {#if !$camera.stream && isRunning}
      <section in:fade>
        <h1>
          <sl-spinner class="starting-spinner" />
          Loading...
        </h1>
        <hr />

        <p>Detector is warming up</p>
      </section>
    {/if}

    {#if $camera.stream && isRunning}
      <section in:fade>
        {#if brightness > 1}
          {#if brightness > 10}
            <div in:fade>
              <h1>
                <sl-icon
                  class="status-icon"
                  style="color: var(--sl-color-warning-50);"
                  name="exclamation-triangle-fill" />
                Too bright!
              </h1>
              <hr />
              <p>
                You must cover your camera device, so that the brightness is
                below 1%
              </p>
            </div>
          {:else}
            <div in:fade>
              <h1>Almost there...</h1>
              <hr />
              <p>Cover your camera device a little bit more</p>
            </div>
          {/if}
          <span class="indicator">
            <sl-icon name="brightness-high-fill" />
            {Math.round(Math.round((brightness + Number.EPSILON) * 100) / 100)}
            %
          </span>
        {:else}
          <div in:fade>
            <h1>
              <sl-icon
                class="status-icon"
                style="color: var(--sl-color-success-50);"
                name="check-square-fill" />
              Detecting...
            </h1>
            <hr />
            <p>Detector is analyzing frames from your camera device</p>
            <p>
              Press the
              <sl-icon class="text-icon" name="stop-fill" />
              button if you wish to stop
            </p>
            <span in:fade={{ delay: 250 }} class="indicator">
              <FramesAnalyzed />
              <br />
              <Timer />
              <br />
              <ParticlesCaughtCounter />
            </span>
          </div>
        {/if}
      </section>
    {/if}
  </div>
</div>
