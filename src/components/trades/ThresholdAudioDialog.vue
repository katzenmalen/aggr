<template>
  <Dialog @clickOutside="close" class="pane-dialog -auto" @mousedown="clickOutsideClose = false" @mouseup="clickOutsideClose = true">
    <template v-slot:header>
      <div>
        <div class="title">
          <div>Threshold</div>
        </div>

        <div class="subtitle">{{ thresholdId }} {{ formatAmount(threshold.amount) }}</div>
      </div>
    </template>
    <div class="d-flex mb16">
      <p class="help-text mx0 -center">
        <i class="icon-info mr4"></i><span v-text="`Play a song when a ${formatAmount(this.min)} - ${formatAmount(this.max)} trade occur.`"></span>
      </p>
      <button class="btn -text mlauto" @click="showHelp = !showHelp"><i class="icon-down" :class="{ 'icon-up': this.showHelp }"></i> help</button>
    </div>
    <div class="help-block mb16" v-if="showHelp">
      Write a sequence of sounds using the play() or playurl() function

      <blockquote>
        play(<br /><span class="ml8" v-tippy title="Frequency (hz)"><code class="-filled">frequency</code>: number</span>,<br />
        <span class="ml8" v-tippy title="Gain (volume, 0 is muted and 1 is max, anything above 1 will sound saturated)"
          ><code class="-filled">gain</code>: number</span
        >,<br />
        <span class="ml8" v-tippy title="FadeOut (gain to endGain duration)"><code class="-filled">fadeOut</code>: number</span>,<br />
        <span class="ml8" v-tippy title="Delay song by n second"><code class="-filled">delay</code>?: number</span>,<br />
        <span class="ml8" v-tippy title="FadeIn (startGain to gain duration)"><code class="-filled">fadeIn</code>?: number</span>,<br />
        <span class="ml8" v-tippy title="HoldDuration (time at gain)"><code class="-filled">holdDuration</code>?: number</span>,<br />
        <span class="ml8" v-tippy title="Oscillator (type of wave, either sine, square, triangle, or sawtooth)"
          ><code class="-filled">osc</code>?: number</span
        >,<br />
        <span class="ml8" v-tippy title="StartGain (cannot be 0, but should be close to 0 like 0.0001)"
          ><code class="-filled">startGain</code>?: number</span
        >,<br />
        <span class="ml8" v-tippy title="EndGain (cannot be 0, but should be close to 0 like 0.0001)"
          ><code class="-filled">endGain</code>?: number</span
        ><br />
        )
      </blockquote>

      <blockquote>
        playurl(<br /><span class="ml8" v-tippy title="URL (.mp3|.wav|.ogg) (https://archive.org/details/audio)"
          ><code class="-filled">url</code>: string</span
        >,<br />
        <span class="ml8" v-tippy title="Gain (volume, 0 is muted and 1 is max, anything above 1 will sound saturated)"
          ><code class="-filled">gain</code>: number</span
        >,<br />
        <span class="ml8" v-tippy title="HoldDuration (time at gain)"><code class="-filled">holdDuration</code>?: number</span>,<br />
        <span class="ml8" v-tippy title="Delay song by n second"><code class="-filled">delay</code>?: number</span>,<br />
        <span class="ml8" v-tippy title="StartTime (start audio at 'x' seconds)"><code class="-filled">startTime</code>: number</span>,<br />
        <span class="ml8" v-tippy title="FadeIn (startGain to gain duration)"><code class="-filled">fadeIn</code>?: number</span>,<br />
        <span class="ml8" v-tippy title="FadeOut (gain to endGain duration)"><code class="-filled">fadeOut</code>: number</span>,<br />
        <span class="ml8" v-tippy title="StartGain (cannot be 0, but should be close to 0 like 0.0001)"
          ><code class="-filled">startGain</code>?: number</span
        >,<br />
        <span class="ml8" v-tippy title="EndGain (cannot be 0, but should be close to 0 like 0.0001)"
          ><code class="-filled">endGain</code>?: number</span
        ><br />
        )
      </blockquote>
    </div>
    <div class="form-group mb16">
      <label for>
        When buy
      </label>
      <div class="d-flex">
        <button v-if="buyAudio !== threshold.buyAudio" class="btn -green mr4" @click="testOriginal('buy', $event)" title="Original" v-tippy>
          <i class="icon-volume-high"></i>
        </button>
        <textarea
          cols="20"
          rows="4"
          ref="behaveBuy"
          class="form-control"
          :value="buyAudio"
          @blur="setInput($event.target.value, 'buy')"
          spellcheck="false"
        ></textarea>
        <button class="btn -green ml4" @click="testCustom('buy', $event)" @dblclick="doTheLoop('buy')" title="Custom">
          <i class="icon-volume-high"></i>
        </button>
      </div>

      <small class="help-text">
        <i class="icon-info -lower mr4"></i>
        <code v-if="buyAudio.startsWith('play(')">play(frequency,gain,fadeOut,delay,fadeIn,holdDuration,osc,startGain,endGain)</code>
        <code v-else-if="buyAudio.startsWith('playurl(')">playurl(url,gain,holdDuration,delay,startTime,fadeIn,fadeOut,startGain,endGain)</code>
      </small>

      <p v-if="buyError" class="form-feedback"><i class="icon-warning mr4"></i> {{ buyError }}</p>
    </div>
    <div class="form-group mb16">
      <label for>
        When sell
      </label>
      <div class="d-flex">
        <button v-if="sellAudio !== threshold.sellAudio" class="btn -red mr4" @click="testOriginal('sell', $event)" title="Original" v-tippy>
          <i class="icon-volume-high"></i>
        </button>
        <textarea
          cols="20"
          rows="4"
          ref="behaveSell"
          class="form-control"
          :value="sellAudio"
          @blur="setInput($event.target.value, 'sell')"
          spellcheck="false"
        ></textarea>
        <button class="btn -red ml4" @click="testCustom('sell', $event)" @dblclick="doTheLoop('sell')">
          <i class="icon-volume-high"></i>
        </button>
      </div>

      <small class="help-text">
        <i class="icon-info -lower mr4"></i>
        <code v-if="sellAudio.startsWith('play(')">play(frequency,gain,fadeOut,delay,fadeIn,holdDuration,osc,startGain,endGain)</code>
        <code v-else-if="sellAudio.startsWith('playurl(')">playurl(url,gain,holdDuration,delay,startTime,fadeIn,fadeOut,startGain,endGain)</code>
      </small>

      <p v-if="sellError" class="form-feedback"><i class="icon-warning mr4"></i> {{ sellError }}</p>
    </div>
    <footer>
      <a href="javascript:void(0);" class="btn -text mrauto" @click="restartWebAudio()">Stop all</a>
      <a href="javascript:void(0);" class="btn -text mr8" @click="close(false)">Cancel</a>
      <button class="btn -large" @click="saveInputs()"><i class="icon-check mr4"></i> Save</button>
    </footer>
  </Dialog>
</template>

<script>
import DialogMixin from '../../mixins/dialogMixin'
import Behave from 'behave-js'
import { formatAmount } from '@/utils/helpers'
import audioService from '@/services/audioService'
import dialogService from '@/services/dialogService'

export default {
  props: {
    paneId: {
      required: true,
      type: String
    },
    thresholdId: {
      required: true,
      type: String
    }
  },
  mixins: [DialogMixin],
  data: () => ({
    buyAudio: '',
    sellAudio: '',
    buyError: null,
    sellError: null,
    showHelp: false
  }),
  computed: {
    threshold: function() {
      if (this.thresholdId === 'liquidations') {
        return this.$store.state[this.paneId].liquidations
      }

      return this.$store.state[this.paneId].thresholds.find(t => t.id === this.thresholdId)
    },
    amounts: function() {
      return this.$store.state[this.paneId].thresholds.map(t => t.amount)
    },
    audioPitch: function() {
      return this.$store.state[this.paneId].audioPitch
    },
    audioVolume: function() {
      return this.$store.state[this.paneId].audioVolume
    },
    index: function() {
      return this.amounts.indexOf(this.threshold.amount)
    },
    min: function() {
      return this.threshold.amount
    },
    max: function() {
      return this.amounts[this.index + 1] || this.amounts[this.index] * 2
    }
  },
  created() {
    this._behaves = []

    if (!this.threshold) {
      return this.$nextTick(() => this.close(false))
    }

    this.buyAudio = this.threshold.buyAudio || ''
    this.sellAudio = this.threshold.sellAudio || ''
  },
  mounted() {
    this.$nextTick(() => this.initBehave())
  },
  beforeDestroy() {
    if (this.looping) {
      clearTimeout(this.looping)
      this.looping = false
    }

    for (const behave of this._behaves) {
      behave.destroy()
    }
  },
  methods: {
    setInput(input, side) {
      this[side + 'Audio'] = input
    },
    saveInputs() {
      if (!this.validate(true)) {
        return
      }

      this.$store.commit(this.paneId + '/SET_THRESHOLD_AUDIO', {
        id: this.thresholdId,
        buyAudio: this.buyAudio,
        sellAudio: this.sellAudio
      })

      this.close(true)
    },
    validate(alert = false) {
      for (const side of ['buy', 'sell']) {
        const litteral = this[side + 'Audio']

        try {
          this.getAdapter(litteral, side)
        } catch (error) {
          if (alert) {
            dialogService.confirm({
              message: `Please check that ${side} audio script is syntactically correct.`,
              ok: 'OK',
              cancel: false
            })
          }

          return false
        }
      }

      return true
    },
    testCustom(side, event) {
      if (this.looping) {
        clearTimeout(this.looping)
        this.looping = false
      }

      const litteral = this[side + 'Audio']

      this.test(litteral, side, event)
    },
    testOriginal(side, event) {
      if (this.looping) {
        clearTimeout(this.looping)
        this.looping = false
      }

      const litteral = this.threshold[side + 'Audio']

      this.test(litteral, side, event)
    },
    doTheLoop(side) {
      this.looping = setTimeout(() => {
        this.test(this.threshold[side + 'Audio'], side)

        this.doTheLoop(side)
      }, Math.random() * 100)
    },
    restartWebAudio() {
      audioService.reconnect()
    },
    async test(litteral, side, event) {
      let percent = 1
      let level
      let amount

      for (let i = 0; i < this.amounts.length; i++) {
        if (this.amounts[i] < this.threshold.amount) {
          continue
        }

        if (amount && amount < this.amounts[i]) {
          break
        }

        const range = this.max - this.min

        if ((event && event.shiftKey) || !range) {
          amount = this.min
        } else {
          amount = this.min + Math.random() * range
          percent = amount / this.amounts[1]
        }

        level = i
      }

      if (amount) {
        ;(await this.getAdapter(litteral, side))(audioService, percent, side, level)

        if (event) {
          this.$store.dispatch('app/showNotice', {
            id: 'testing-threshold-audio',
            type: side === 'buy' ? 'success' : 'error',
            title: 'NOW PLAYING : ' + formatAmount(amount) + ' ' + side.toUpperCase() + ' trade',
            timeout: 1000
          })
        }
      }
    },
    formatAmount(amount) {
      return formatAmount(amount)
    },
    async getAdapter(litteral, side) {
      try {
        const adapter = await audioService.buildAudioFunction(litteral, side, this.audioPitch, this.audioVolume, true)

        this[side + 'Error'] = null

        return adapter
      } catch (error) {
        this[side + 'Error'] = error.message

        throw error
      }
    },
    initBehave() {
      for (const el of [this.$refs.behaveBuy, this.$refs.behaveSell]) {
        this._behaves.push(
          new Behave({
            textarea: el,
            replaceTab: true,
            softTabs: true,
            tabSize: 2,
            autoOpen: true,
            overwrite: true,
            autoStrip: true,
            autoIndent: true,
            fence: false
          })
        )
      }
    }
  }
}
</script>
