<script>
  import BufferSpectrogramRenderer from "./components/BufferSpectrogramRenderer.svelte";
  import Ribbon                    from "./components/Ribbon.svelte";
  import RibbonTab                 from "./components/RibbonTab.svelte";

  /**
    @type {AudioContext}
    */
  let actx = null;
  
  /**
   * @type {Uint8Array}
  */
  let rbuf   = null;
  let rbufp  = 0;
  let rbufs  = 5000;
  let rbufiv = 10;
  /** @type {AnalyserNode} */
  let rbufa  = null;
  let rbuft  = null;
  let rbufi  = null;
  let rbufw  = 4096;   // frequency bin count

  let audioInputs = [];
  function refreshAudioInputs() {
    return navigator.mediaDevices.enumerateDevices().then((devs) => {
      let inps = [];
      for (let d of devs) {
        if (d.kind == "audioinput") {
          inps.push(d);
        }
      }
      audioInputs = inps;
    })
  }

  function selectAudioInput(id) {
    navigator.mediaDevices.getUserMedia({
      audio: id == null ? true : {
        deviceId: id
      }
    }).then((stream) => {
      rbufi = actx.createMediaStreamSource(stream);

      rbufi.connect(rbufa);

      rbufa.fftSize = rbufw * 2;
      rbuf          = new Uint8Array(rbufa.frequencyBinCount * (rbufs + 1)); 
      rbuft         = setInterval(recordAudioFrame, rbufiv);
    })
  }

  function attachAudioContext() {
    navigator.mediaDevices.getUserMedia({
      audio: true
    }).then((v) => {
      actx = new AudioContext();

      rbufa = actx.createAnalyser();

      refreshAudioInputs();
      selectAudioInput();
    });
  }

  function recordAudioFrame() {
    rbufa.getByteFrequencyData(rbuf.subarray(rbufa.frequencyBinCount * rbufp, rbufa.frequencyBinCount * (rbufp + 1)));
    
    rbufp++;

    if (rbufp >= rbufs) {
      rbuf.copyWithin(0, rbufa.frequencyBinCount);

      rbufp--;
    }
  }

  attachAudioContext();
</script>
<div class="root">
  <Ribbon>
    <RibbonTab label="Home">
      <button on:click={function() {
        if (actx == null) {
          attachAudioContext();
        }
      }}>
        {#if actx == null}
          Attach
        {:else}
          Attached
        {/if}
      </button>
      <span>Input device: </span>
      <select on:input={function(ev) {
        selectAudioInput(ev.target[ev.target.options.selectedIndex]);
      }}>
        {#each audioInputs as dev}
          <option data-devid={dev.deviceId}>{dev.label}</option>
        {/each}
      </select>
    </RibbonTab>
  </Ribbon>
  <main>
    <div class="sidestrip">
      <button>S</button>
      <button>P</button>
    </div>
    <div style:display="flex" class="mainwin">
      <!-- renderer -->
      <div style:display="flex" class="renderertab">
        <BufferSpectrogramRenderer
          bufHeight={rbufw}
          bufLength={rbufs}
          bufInterval={rbufiv}
          buf={rbuf}>
        </BufferSpectrogramRenderer>
      </div>
    </div>
  </main>
</div>
<style>
  :global(body) {
    margin: 0;

    height: 100vh;
    width:  100vw;

    overflow: hidden;
  }

  .root {
    height: 100vh;
    width:  100vw;

    display: flex;

    flex-direction: column;
  }

  main {
    display: flex;

    flex-grow: 1;
  }

  .mainwin {
    display: flex;

    flex-grow: 1;
    flex-direction: column;
  }

  .sidestrip {
    display: flex;

    flex-direction: column;
  }

  .renderertab {
    flex-grow: 1;
  }
</style>