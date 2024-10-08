<script setup>
import OscillatorUI from './components/OscillatorUI.vue';
import FilterUI from './components/FilterUI.vue';
import AmpUI from './components/AmpUI.vue';
import WaveDisplay from './components/WaveDisplay.vue';
import parameterDescriptor from "./parameterDescriptor.js";
import SpectrumAnalyzer from './components/spectrum_analyzer.vue';
import DistortionUI from './components/DistortionUI.vue';
import TremoroUI from './components/LfoUI.vue'; 
import VibratoUI from './components/VibratoUI.vue';
import DelayUI from './components/DelayUI.vue';
import MidiHandler from './MidiHandler.js';
</script>

<template>
  <div class="whole">
    <div class="title">
      <a class="logo" href="https://www.korg.com/jp/" target="_blank">
        <img class="logo-img" src="./assets/korg.jpg" />
      </a>
      <h2 class="title-text">
        Internship Synthesizer
      </h2>
    </div>
    <div>
      <button id="start-button" v-show="!isStarted" @click="setup">
        start
      </button>
    </div>
    <div v-show="isStarted">
      <div>
        <input id="noteOn-button" type="button" value="Note On" @mousedown="noteOn" @mouseup="noteOff" />
      </div>
      <div>
        <WaveDisplay ref="wave" :analyser="analyser" />
      </div>
      <div>
        <SpectrumAnalyzer ref="spectrum" :analyser="analyser" />
      </div>
      <div>
        <OscillatorUI @parameterChanged="onParameterChanged" />
        <DistortionUI @parameterChanged="onParameterChanged" />
        <TremoroUI @parameterChanged="onParameterChanged"/>
        <vibratoUI @parameterChanged="onParameterChanged" />
        <DelayUI @parameterChanged="onParameterChanged" />
        <FilterUI @parameterChanged="onParameterChanged" />
        <AmpUI @parameterChanged="onParameterChanged" />
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: "MainUI",
  data() {
    return {
      isStarted: false,
      context: null,
      sampleRate: 48000,
      synthesizer: null,
      analyser: null,
      waveDataSize: 512,
      params: parameterDescriptor.parameters
    }
  },
  methods: {
    setup(){
      
      this.setupWorklet();
      this.setupMidi();
    },
    setupMidi() {
      MidiHandler.init();
      MidiHandler.setHandleMidiCallback(this.processMidiMessage)
    },
    processMidiMessage(message) {
      
      console.log(message.data);
      const  [status, data1, data2] = message.data;
      switch(status){
        case 144:
          this.noteOn();
          const freq = 440 * 2 ** ((data1-69) / 12);
          const msg = {
            id: this.params.frequency.id,
            value: freq
          };
          this.onParameterChanged(msg);
          break;
        case 128:
          this.noteOff();
          break;
      
      }
      
      
        
    },
    setupWorklet() {
      console.log("start")
      this.isStarted = true
      window.AudioContext = window.AudioContext || window.webkitAudioContext;
      const context = new AudioContext()
      this.sampleRate = context.sampleRate
      this.context = context

      let options = {
        processorOptions: {
          sampleRate: this.context.sampleRate,
        },
        numberOfInputs: 0,
        numberOfOutputs: 1,
        outputChannelCount: [1]
      }
      context.audioWorklet.addModule('/SynthesizerWorklet.js').then(() => {
        this.synthesizer = new AudioWorkletNode(context, 'synthesizer-worklet', options)

        console.log(this.synthesizer)

        this.analyser = this.context.createAnalyser();
        this.analyser.maxDecibels = 0;
        this.analyser.fftSize = this.waveDataSize;
        this.synthesizer.connect(this.analyser).connect(context.destination)

        setInterval(this.draw, 1000 / 10) //  オシロスコープ描画用タイマー
      })
    },
    onParameterChanged(value) {
      const data = { type: "param", value: value }
      this.synthesizer.port.postMessage(data)
    },
    noteOn() {
      console.log("noteOn")
      const data = { type: "noteOn", value: true }
      this.synthesizer.port.postMessage(data)
    },
    noteOff() {
      const data = { type: "noteOn", value: false }
      this.synthesizer.port.postMessage(data)
    },
    draw() {
      this.$refs.wave.drawWave()
      this.$refs.spectrum.drawWave()
    },
   


  },
}
</script>

<style scoped></style>
