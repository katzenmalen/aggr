<template>
  <div class="pane-trades" :class="{ '-logos': this.showLogos, '-logos-colors': !this.monochromeLogos, '-slippage': this.calculateSlippage }">
    <pane-header :paneId="paneId" />
    <ul ref="tradesContainer" class="hide-scrollbar"></ul>
    <div v-if="showPlaceholder" class="trades-placeholder hide-scrollbar">
      <div class="mt16 ml16 mr16 help-text">
        <strong>Waiting for trades</strong>

        <p>No{{ tradeType === 'both' ? 'thing' : ' ' + tradeType }} matching</p>
        <code v-for="market of pane.markets" :key="market" class="trades-placeholder__market">{{ market.replace(/^\w+:/, '') }}<br /></code>
        <p>with amount > {{ thresholds[0].amount }}</p>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Mixins } from 'vue-property-decorator'

import { formatPrice, formatAmount, slugify } from '../../utils/helpers'
import { getColorByWeight, getColorLuminance, getAppBackgroundColor, splitRgba } from '../../utils/colors'

import aggregatorService from '@/services/aggregatorService'
import workspacesService from '@/services/workspacesService'
import audioService, { AudioFunction } from '../../services/audioService'
import PaneMixin from '@/mixins/paneMixin'
import PaneHeader from '../panes/PaneHeader.vue'
import { Trade } from '@/types/test'
import { Threshold, TradeTypeFilter } from '@/store/panesSettings/trades'

const GIFS: { [keyword: string]: string[] } = {}
const PROMISES_OF_GIFS: { [keyword: string]: Promise<string[]> } = {}

interface ThresholdColorsBySide {
  threshold?: number
  range?: number
  buy: {
    from: number[]
    to: number[]
    fromLuminance: number
    toLuminance: number
  }
  sell: {
    from: number[]
    to: number[]
    fromLuminance: number
    toLuminance: number
  }
}

interface ThresholdAudiosBySide {
  buy: AudioFunction
  sell: AudioFunction
}

@Component({
  components: { PaneHeader },
  name: 'Trades'
})
export default class extends Mixins(PaneMixin) {
  showPlaceholder = true

  private _onStoreMutation: () => void

  // cache all thoses vuex property (via cacheFilters when component load / a settings is updated)
  // so they the getters don't get re evaluated on each trades
  private _tradesCount = 0
  private _thresholdsColors: ThresholdColorsBySide[]
  private _thresholdsAudios: ThresholdAudiosBySide[]
  private _liquidationsAudio: ThresholdAudiosBySide
  private _liquidationsColor: ThresholdColorsBySide
  private _liquidationThreshold: Threshold
  private _lastTradeTimestamp: number
  private _lastSide: 'buy' | 'sell'
  private _audioThreshold: number
  private _minimumThresholdAmount: number
  private _significantThresholdAmount: number
  private _activeExchanges: { [exchange: string]: boolean }
  private _tradeType: TradeTypeFilter
  private _multipliers: { [identifier: string]: number }
  private _paneMarkets: { [identifier: string]: boolean }
  private _timeAgoInterval: number
  private _enableAnimationsTimeout: number
  private _disableAnimations: boolean
  private _preferQuoteCurrencySize: boolean

  get maxRows() {
    return this.$store.state[this.paneId].maxRows
  }

  get thresholds(): Threshold[] {
    return this.$store.state[this.paneId].thresholds
  }

  get tradeType() {
    return this.$store.state[this.paneId].tradeType
  }

  get showTradesPairs() {
    return this.$store.state[this.paneId].showTradesPairs
  }

  get showLogos() {
    return this.$store.state[this.paneId].showLogos
  }

  get monochromeLogos() {
    return this.$store.state[this.paneId].monochromeLogos
  }

  get exchanges() {
    return this.$store.state.exchanges
  }

  get calculateSlippage() {
    return this.$store.state.settings.calculateSlippage
  }

  get audioThreshold() {
    return this.$store.state[this.paneId].audioThreshold
  }

  $refs!: {
    tradesContainer: HTMLElement
  }

  created() {
    this.cacheFilters()

    this.loadThresholdsGifs()
    this.prepareColorsSteps()
    this.prepareThresholdsSounds()
    this.prepareAudioThreshold()

    this._tradesCount = 0

    aggregatorService.on('trades', this.onTrades)

    this._onStoreMutation = this.$store.subscribe(mutation => {
      switch (mutation.type) {
        case 'app/EXCHANGE_UPDATED':
        case this.paneId + '/SET_THRESHOLD_MULTIPLIER':
        case this.paneId + '/TOGGLE_TRADE_TYPE':
          this.cacheFilters()
          break
        case this.paneId + '/SET_MAX_ROWS':
          while (this._tradesCount > this.maxRows) {
            this.$refs.tradesContainer.removeChild(this.$refs.tradesContainer.lastChild)
            this._tradesCount--
          }
          break
        case 'panes/SET_PANE_MARKETS':
          if (mutation.payload.id === this.paneId) {
            this.cacheFilters()
            this.refreshList()
          }
          break
        case this.paneId + '/SET_THRESHOLD_GIF':
          this.retrieveThresholdGifs(mutation.payload.value)
          this.refreshList()
          break
        case this.paneId + '/SET_THRESHOLD_AUDIO':
        case this.paneId + '/SET_AUDIO_PITCH':
        case this.paneId + '/SET_AUDIO_VOLUME':
        case 'settings/TOGGLE_AUDIO':
        case 'settings/SET_AUDIO_VOLUME':
          this.prepareThresholdsSounds()
          this.prepareAudioThreshold()
          break
        case 'settings/SET_CHART_BACKGROUND_COLOR':
        case this.paneId + '/SET_THRESHOLD_COLOR':
        case this.paneId + '/SET_THRESHOLD_AMOUNT':
        case this.paneId + '/DELETE_THRESHOLD':
        case this.paneId + '/ADD_THRESHOLD':
          this.prepareThresholdsSounds()
          this.prepareColorsSteps()
          this.refreshList()
          this.prepareAudioThreshold()
          break
        case this.paneId + '/SET_AUDIO_THRESHOLD':
        case this.paneId + '/TOGGLE_MUTED':
          this.prepareAudioThreshold()
          break
      }
    })
  }
  mounted() {
    this.startTimeAgoInterval()
  }

  beforeDestroy() {
    aggregatorService.off('trades', this.onTrades)

    this._onStoreMutation()

    clearInterval(this._timeAgoInterval)
  }

  startTimeAgoInterval() {
    let now

    const timeAgo = milliseconds => {
      if (milliseconds < 60000) {
        return Math.floor(milliseconds / 1000) + 's'
      } else if (milliseconds < 3600000) {
        return Math.floor(milliseconds / 60000) + 'm'
      } else {
        return Math.floor(milliseconds / 3600000) + 'h'
      }
    }

    this._timeAgoInterval = setInterval(() => {
      const elements = this.$el.getElementsByClassName('-timestamp')
      const length = elements.length

      if (!length) {
        return
      }

      now = +new Date()

      const topOfTheMinute = (now / 1000) % 60 < 1

      let previousRowTimeAgo

      for (let i = 0; i < length; i++) {
        if (typeof elements[i] === 'undefined') {
          break
        }

        const milliseconds = now - (elements[i] as any).getAttribute('timestamp')
        const txt = timeAgo(milliseconds)

        if (txt === previousRowTimeAgo && txt) {
          elements[i - 1].textContent = ''
          elements[i - 1].className = 'trade__time'
          continue
        }

        if (txt !== elements[i].textContent) {
          elements[i].textContent = txt

          if (!topOfTheMinute && txt[txt.length - 1] !== 's') {
            break
          }
        }

        previousRowTimeAgo = txt
      }
    }, 1000)
  }

  onTrades(trades: Trade[]) {
    for (let i = 0; i < trades.length; i++) {
      const identifier = trades[i].exchange + trades[i].pair

      if (!this._activeExchanges[trades[i].exchange] || !this._paneMarkets[identifier]) {
        continue
      }

      const trade = trades[i]
      const amount = trade.size * (this._preferQuoteCurrencySize ? trade.price : 1)
      const multiplier = this._multipliers[identifier]

      if (trade.liquidation) {
        if (this._tradeType === 'both' && amount >= this._liquidationThreshold.amount * multiplier) {
          this.appendRow(trade, amount, multiplier)
          continue
        } else if (this._tradeType === 'trades') {
          continue
        }
      } else if (this._tradeType === 'liquidations') {
        continue
      }

      if (amount >= this._minimumThresholdAmount * multiplier) {
        this.appendRow(trade, amount, multiplier)
      } else if (amount > this._audioThreshold) {
        this._thresholdsAudios[0][trade.side](audioService, amount / (this._significantThresholdAmount * multiplier), trade.side, 0)
      }
    }
  }

  appendRow(trade: Trade, amount, multiplier) {
    if (!this._tradesCount) {
      this.showPlaceholder = false
    }

    this._tradesCount++

    let animated = false

    const li = document.createElement('li')
    li.className = 'trade'
    li.className += ' -' + trade.exchange

    li.className += ' -' + trade.side

    if (trade.liquidation && this._tradeType === 'both') {
      li.className += ' -liquidation'

      const side = document.createElement('div')
      side.className = 'trade__side ' + (trade.side === 'buy' ? 'icon-bear' : 'icon-bull')

      li.appendChild(side)

      if (this._liquidationThreshold.gif && GIFS[this._liquidationThreshold.gif] && amount >= this._liquidationThreshold.amount * multiplier) {
        li.className += ' -gif'
        // get random gif for this this._liquidationThreshold

        if (!this._disableAnimations) {
          li.style.backgroundImage = `url('${
            GIFS[this._liquidationThreshold.gif][Math.floor(Math.random() * (GIFS[this._liquidationThreshold.gif].length - 1))]
          }`

          animated = true
        }
      }

      const percentToSignificant = Math.min(1, amount / this._significantThresholdAmount)

      const luminance = this._liquidationsColor[trade.side][(percentToSignificant < 0.5 ? 'from' : 'to') + 'Luminance']
      const backgroundColor = this._liquidationsColor[trade.side].to

      li.style.color =
        luminance > (animated ? 144 : 170)
          ? 'rgba(0, 0, 0, ' + Math.min(1, 0.25 + percentToSignificant) + ')'
          : 'rgba(255, 255, 255, ' + Math.min(1, 0.25 + percentToSignificant) + ')'

      li.style.backgroundColor =
        'rgb(' + backgroundColor[0] + ', ' + backgroundColor[1] + ', ' + backgroundColor[2] + ', ' + percentToSignificant + ')'

      if (amount > this._audioThreshold) {
        this._liquidationsAudio[trade.side](audioService, amount / (this._significantThresholdAmount * multiplier), trade.side, 0)
      }
    } else {
      if (trade.liquidation) {
        li.className += ' -liquidation'

        const side = document.createElement('div')
        side.className = 'trade__side ' + (trade.side === 'buy' ? 'icon-bear' : 'icon-bull')
        li.appendChild(side)
      } else {
        if (trade.side !== this._lastSide) {
          const side = document.createElement('div')
          side.className = 'trade__side icon-side'
          li.appendChild(side)
          this._lastSide = trade.side
        }
      }

      for (let i = 0; i < this.thresholds.length; i++) {
        li.className += ' -level-' + i

        if (!this.thresholds[i + 1] || amount < this.thresholds[i + 1].amount * multiplier) {
          const color = this._thresholdsColors[Math.min(this.thresholds.length - 2, i)]
          const threshold = this.thresholds[i]

          if (threshold.gif && GIFS[threshold.gif]) {
            // get random gif for this threshold
            li.className += ' -gif'

            if (!this._disableAnimations) {
              li.style.backgroundImage = `url('${GIFS[threshold.gif][Math.floor(Math.random() * (GIFS[threshold.gif].length - 1))]}`
            }

            animated = true
          }

          // percentage to next threshold
          const percentToNextThreshold = (Math.max(amount, color.threshold) - color.threshold) / color.range
          const percentToSignificant = amount / this._significantThresholdAmount

          // 0-255 luminance of nearest color
          const luminance = color[trade.side][(percentToNextThreshold < 0.5 ? 'from' : 'to') + 'Luminance']

          // background color simple color to color based on percentage of amount to next threshold
          const backgroundColor = getColorByWeight(color[trade.side].from, color[trade.side].to, percentToNextThreshold)
          li.style.backgroundColor = 'rgb(' + backgroundColor[0] + ', ' + backgroundColor[1] + ', ' + backgroundColor[2] + ')'

          // take background color and apply logarithmic shade based on amount to this._significantThresholdAmount percentage
          // darken if luminance of background is high, lighten otherwise
          li.style.color =
            luminance > (animated ? 144 : 170)
              ? 'rgba(0, 0, 0, ' + Math.min(1, 0.25 + percentToSignificant) + ')'
              : 'rgba(255, 255, 255, ' + Math.min(1, 0.25 + percentToSignificant) + ')'

          if (amount > this._audioThreshold) {
            this._thresholdsAudios[i][trade.side](audioService, percentToSignificant, trade.side, i)
          }

          break
        }
      }
    }

    const exchange = document.createElement('div')
    exchange.className = 'trade__exchange'

    if (!this.showLogos) {
      exchange.innerText = trade.exchange.replace('_', ' ')
    }

    exchange.setAttribute('title', trade.exchange)
    li.appendChild(exchange)

    if (this.showTradesPairs) {
      const pair = document.createElement('div')
      pair.className = 'trade__pair'
      pair.innerText = trade.pair.replace('_', ' ')
      pair.setAttribute('title', trade.pair)
      li.appendChild(pair)
    }

    const price = document.createElement('div')
    price.className = 'trade__price'
    price.innerText = `${formatPrice(trade.price)}`
    li.appendChild(price)

    if (trade.slippage) {
      if (this.calculateSlippage === 'price' && Math.abs(trade.slippage) / trade.price > 0.0005) {
        price.setAttribute(
          'slippage',
          (trade.slippage > 0 ? '+' : '') + formatPrice(trade.slippage) + document.getElementById('app').getAttribute('data-quote-symbol')
        )
      } else if (this.calculateSlippage === 'bps' && trade.slippage) {
        price.setAttribute('slippage', (trade.slippage > 0 ? '+' : '-') + formatPrice(trade.slippage))
      }
    }

    const amount_div = document.createElement('div')
    amount_div.className = 'trade__amount'

    const amount_quote = document.createElement('span')
    amount_quote.className = 'trade__amount__quote'
    amount_quote.innerHTML = '<span class="icon-quote"></span> <span>' + formatAmount(trade.price * trade.size) + '</span>'

    const amount_base = document.createElement('span')
    amount_base.className = 'trade__amount__base'
    amount_base.innerHTML = '<span class="icon-base"></span> <span>' + formatAmount(trade.size) + '</span>'

    amount_div.appendChild(amount_quote)
    amount_div.appendChild(amount_base)
    li.appendChild(amount_div)

    const date = document.createElement('div')
    date.className = 'trade__time'

    if (trade.timestamp) {
      const timestamp = Math.floor(trade.timestamp / 1000) * 1000

      if (timestamp !== this._lastTradeTimestamp) {
        this._lastTradeTimestamp = timestamp

        date.setAttribute('timestamp', trade.timestamp.toString())

        date.className += ' -timestamp'
      }
    }

    li.appendChild(date)

    this.$refs.tradesContainer.prepend(li)

    if (this._tradesCount > this.maxRows) {
      this._tradesCount--
      this.$refs.tradesContainer.removeChild(this.$refs.tradesContainer.lastChild)
    }
  }

  async loadThresholdsGifs() {
    for (const threshold of this.thresholds) {
      if (!threshold.gif || GIFS[threshold.gif]) {
        continue
      }

      this.retrieveThresholdGifs(threshold.gif)
    }
  }

  async retrieveThresholdGifs(gif, isDeleted = false) {
    const slug = slugify(gif)

    if (isDeleted) {
      if (GIFS[gif]) {
        this.$store.dispatch('app/showNotice', {
          title: 'Removed ' + GIFS[gif].length + ' gifs about "' + gif + '"',
          type: 'info'
        })

        delete GIFS[gif]
      }

      await workspacesService.deleteGifs(slug)

      return Promise.resolve()
    }

    const storage = await workspacesService.getGifs(slug)
    let gifs

    if (storage && +new Date() - storage.timestamp < 1000 * 60 * 60 * 24 * 7) {
      gifs = storage.data
    } else if (!PROMISES_OF_GIFS[gif]) {
      gifs = await this.fetchGifByKeyword(gif)
    }

    if (gifs) {
      GIFS[gif] = gifs
    }
  }

  async fetchGifByKeyword(gif: string) {
    if (!gif || !GIFS) {
      return
    }

    const slug = slugify(gif)

    const promise = fetch('https://g.tenor.com/v1/search?q=' + gif + '&key=LIVDSRZULELA&limit=100&key=DF3B0979C761')
      .then(res => res.json())
      .then(async res => {
        if (!res.results || !res.results.length) {
          return
        }

        GIFS[gif] = []

        for (const item of res.results) {
          GIFS[gif].push(item.media[0].tinygif.url)
        }

        this.$store.dispatch('app/showNotice', {
          title: 'Downloaded ' + GIFS[gif].length + ' gifs about "' + gif + '"',
          type: 'success'
        })

        await workspacesService.saveGifs({
          slug,
          keyword: gif,
          timestamp: +new Date(),
          data: GIFS[gif]
        })

        return GIFS[gif]
      })
      .finally(() => {
        delete PROMISES_OF_GIFS[gif]
      })

    PROMISES_OF_GIFS[gif] = promise

    return promise
  }

  prepareColorsSteps() {
    const appBackgroundColor = getAppBackgroundColor()

    const liquidationBuy = splitRgba(this._liquidationThreshold.buyColor, appBackgroundColor)
    const liquidationBuyFrom = [liquidationBuy[0], liquidationBuy[1], liquidationBuy[2], 0]
    const liquidationBuyTo = [liquidationBuy[0], liquidationBuy[1], liquidationBuy[2], 1]
    const liquidationSell = splitRgba(this._liquidationThreshold.sellColor, appBackgroundColor)
    const liquidationSellFrom = [liquidationSell[0], liquidationSell[1], liquidationSell[2], 0]
    const liquidationSellTo = [liquidationSell[0], liquidationSell[1], liquidationSell[2], 1]

    this._liquidationsColor = {
      buy: {
        from: liquidationBuyFrom,
        to: liquidationBuyTo,
        fromLuminance: getColorLuminance(liquidationBuyFrom),
        toLuminance: getColorLuminance(liquidationBuyTo)
      },
      sell: {
        from: liquidationSellFrom,
        to: liquidationSellTo,
        fromLuminance: getColorLuminance(liquidationSellFrom),
        toLuminance: getColorLuminance(liquidationSellTo)
      }
    }

    this._thresholdsColors = []
    this._minimumThresholdAmount = this.thresholds[0].amount
    this._significantThresholdAmount = this.thresholds[1].amount

    for (let i = 0; i < this.thresholds.length - 1; i++) {
      const buyFrom = splitRgba(this.thresholds[i].buyColor, appBackgroundColor)
      const buyTo = splitRgba(this.thresholds[i + 1].buyColor, appBackgroundColor)
      const sellFrom = splitRgba(this.thresholds[i].sellColor, appBackgroundColor)
      const sellTo = splitRgba(this.thresholds[i + 1].sellColor, appBackgroundColor)

      this._thresholdsColors.push({
        threshold: this.thresholds[i].amount,
        range: this.thresholds[i + 1].amount - this.thresholds[i].amount,
        buy: {
          from: buyFrom,
          to: buyTo,
          fromLuminance: getColorLuminance(buyFrom),
          toLuminance: getColorLuminance(buyTo)
        },
        sell: {
          from: sellFrom,
          to: sellTo,
          fromLuminance: getColorLuminance(sellFrom),
          toLuminance: getColorLuminance(sellTo)
        }
      })
    }
  }

  async prepareThresholdsSounds() {
    const audioPitch = this.$store.state[this.paneId].audioPitch
    const paneVolume = this.$store.state[this.paneId].audioVolume

    this._thresholdsAudios = []

    for (let i = 0; i < this.thresholds.length; i++) {
      this._thresholdsAudios[i] = {
        buy: await audioService.buildAudioFunction(this.thresholds[i].buyAudio, 'buy', audioPitch, paneVolume),
        sell: await audioService.buildAudioFunction(this.thresholds[i].sellAudio, 'sell', audioPitch, paneVolume)
      }
    }

    this._liquidationsAudio = {
      buy: await audioService.buildAudioFunction(this._liquidationThreshold.buyAudio, 'buy', audioPitch, paneVolume),
      sell: await audioService.buildAudioFunction(this._liquidationThreshold.sellAudio, 'sell', audioPitch, paneVolume)
    }
  }

  prepareAudioThreshold() {
    if (!this.$store.state.settings.useAudio || this.$store.state[this.paneId].muted || this.$store.state[this.paneId].audioVolume === 0) {
      this._audioThreshold = Infinity
      return
    }

    if (this.audioThreshold) {
      if (typeof this.audioThreshold === 'string' && /\d\s*%$/.test(this.audioThreshold)) {
        this._audioThreshold = this._minimumThresholdAmount * (parseFloat(this.audioThreshold) / 100)
      } else {
        this._audioThreshold = +this.audioThreshold
      }
    } else {
      this._audioThreshold = this._minimumThresholdAmount * 0.1
    }
  }

  cacheFilters() {
    this._preferQuoteCurrencySize = this.$store.state.settings.preferQuoteCurrencySize
    this._disableAnimations = this.$store.state.settings.disableAnimations
    this._tradeType = this.$store.state[this.paneId].tradeType
    this._liquidationThreshold = this.$store.state[this.paneId].liquidations
    this._activeExchanges = { ...this.$store.state.app.activeExchanges }
    this._multipliers = {}
    this._paneMarkets = this.$store.state.panes.panes[this.paneId].markets.reduce((output, market) => {
      const identifier = market.replace(':', '')
      const multiplier = this.$store.state[this.paneId].multipliers[identifier]

      this._multipliers[identifier] = !isNaN(multiplier) ? multiplier : 1

      output[identifier] = true
      return output
    }, {})
  }

  clearList() {
    this.$refs.tradesContainer.innerHTML = ''
    this._tradesCount = 0
    this.showPlaceholder = true
  }

  refreshList() {
    if (!this._tradesCount) {
      return this.clearList()
    }

    let resetAnimation = false
    if (!this.$store.state.settings.disableAnimations) {
      this._disableAnimations = true
      resetAnimation = true
    }

    if (this._enableAnimationsTimeout) {
      clearTimeout(this._enableAnimationsTimeout)
    }
    this._enableAnimationsTimeout = setTimeout(() => {
      if (resetAnimation) {
        this._disableAnimations = false
      }

      this._enableAnimationsTimeout = null
    }, 1000)
    const elements = this.$el.getElementsByClassName('trade')

    const pairByExchanges = this.$store.state.app.activeMarkets.reduce((obj, market) => {
      if (!obj[market.exchange]) {
        obj[market.exchange] = market.pair
      }

      return obj
    }, {})

    const trades: Trade[] = []

    for (const element of elements) {
      const exchange = element.querySelector('.trade__exchange').getAttribute('title')

      const pair = pairByExchanges[exchange]

      if (!pair) {
        console.warn('unable to map pair for exchange', exchange, pairByExchanges)
        continue
      }

      const pairElement = element.querySelector('.trade__pair') as HTMLElement

      const timestamp = element.querySelector('.trade__time').getAttribute('timestamp')
      const price = parseFloat((element.querySelector('.trade__price') as HTMLElement).innerText) || 0
      const size = parseFloat((element.querySelector('.trade__amount__base') as HTMLElement).innerText) || 0
      const side: 'buy' | 'sell' = element.classList.contains('-buy') ? 'buy' : 'sell'
      const symbol = pairElement ? pairElement.innerText : null

      const trade: Trade = {
        timestamp: (timestamp as unknown) as number,
        exchange,
        pair: symbol ? symbol : pair,
        price,
        size,
        side
      }

      if (element.classList.contains('-liquidation')) {
        trade.liquidation = true
      }

      trades.push(trade)
    }

    trades.reverse()

    this.clearList()

    const audioThresholdValue = this._audioThreshold
    this._audioThreshold = Infinity

    for (const trade of trades) {
      const identifier = trade.exchange + trade.pair
      const amount = trade.size * (this._preferQuoteCurrencySize ? trade.price : 1)
      const multiplier = this._multipliers[identifier] || 1

      this.appendRow(trade, amount, multiplier)
    }

    this._audioThreshold = audioThresholdValue
  }
}
</script>

<style lang="scss">
.pane-trades {
  font-weight: 400;

  ul {
    margin: 0;
    padding: 0;
    overflow: auto;
    max-height: 100%;
    transform: translateZ(0);
    -webkit-transform: translateZ(0);
  }

  &.-large {
    font-weight: 500;
    .trade {
      padding-top: 3px;
      padding-bottom: 5px;
    }
  }

  &.-logos {
    .trade__exchange {
      flex-basis: 0;
      flex-grow: 0.25;
      overflow: visible;
      text-align: right;
      margin: 0;
      padding: 0;
      font-size: 100%;

      &:before {
        font-family: 'icon';
        display: inline-block;
        vertical-align: -2px;
        font-weight: 400;
      }
    }

    @each $exchange, $icon in $exchanges {
      .-#{$exchange} .trade__exchange:before {
        content: $icon;
      }
    }

    &.-logos-colors {
      .trade__exchange {
        height: 1em;
        background-position: right;

        &:before {
          display: none;
        }
      }

      @each $exchange, $icon in $exchanges {
        .-#{$exchange} .trade__exchange {
          background-image: url('../../assets/exchanges/#{$exchange}.svg');
        }
      }
    }
  }
}

.trades-placeholder {
  text-align: center;
  text-align: center;
  overflow: auto;
  max-height: 100%;

  &__market {
    color: var(--theme-color-200);
  }
}

.trade {
  display: flex;
  flex-flow: row nowrap;
  background-position: center center;
  background-size: cover;
  background-blend-mode: overlay;
  position: relative;
  align-items: center;
  padding: 1px 0.5em 3px;

  &:after {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    opacity: 0;
    background-color: white;
    animation: 1s $ease-out-expo highlight;
    pointer-events: none;
  }

  &.-empty {
    justify-content: center;
    padding: 1em;

    &:after {
      display: none;
    }
  }

  &.-liquidation {
    .trade__side {
      font-weight: 400;

      &:after {
        margin-left: 1rem;
      }
    }
  }

  &.-sell {
    background-color: lighten($red, 35%);
    color: $red;

    .icon-side:before {
      content: unicode($icon-down);
    }
  }

  &.-buy {
    background-color: lighten($green, 50%);
    color: $green;

    .icon-side:before {
      content: unicode($icon-up);
    }
  }

  &.-level-0 {
    height: 1.5em;
    font-size: 0.875em;
  }

  &.-level-1 {
    height: 1.5em;
    font-size: 1em;
  }

  &.-level-2 {
    height: 1.75em;
    font-size: 1.125em;
    font-weight: 600;
  }

  &.-level-3 {
    height: 2em;
    box-shadow: 0 0 20px rgba(black, 0.5);
    z-index: 1;
    font-size: 1.25em;
  }

  > div {
    flex-basis: 0;
    flex-grow: 1;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    text-align: center;
  }

  .trade__side {
    flex-grow: 0;
    flex-basis: 1em;
    font-size: 1em;
    position: absolute;
    font-weight: 600;
  }

  .trade__pair {
    text-align: left !important;
    margin-left: 0.5em;
    flex-grow: 0.66 !important;
  }

  .icon-currency,
  .icon-quote,
  .icon-base {
    line-height: 0;
  }

  .trade__exchange {
    background-repeat: no-repeat;
    flex-grow: 0.5;
    white-space: normal;
    word-break: inherit;
    font-size: 75%;
    line-height: 1;
    text-align: center;
    margin-right: 0;
    margin-left: auto;
    padding-left: 1.5em;
  }

  .trade__price {
    flex-grow: 0.5;

    &:after {
      content: attr(slippage);
      font-size: 80%;
      position: relative;
      top: -2px;
      left: 2px;
      opacity: 0.75;
      font-weight: 400;
    }
  }

  .trade__amount {
    text-align: right;
    position: relative;
    flex-grow: 0.4;

    > span {
      max-width: 100%;
      overflow: hidden;
      text-overflow: ellipsis;
      transition: transform 0.1s ease-in-out;
      display: block;
      padding: 0 1px;

      &.trade__amount__quote {
        position: absolute;
        margin: auto;
      }

      &.trade__amount__base {
        transform: translateX(25%);
        opacity: 0;
        text-align: left;
      }
    }

    &:hover {
      > span.trade__amount__base {
        transform: none;
        opacity: 1;
      }

      > span.trade__amount__quote {
        transform: translateX(-25%);
        opacity: 0;
      }
    }
  }

  .trade__time {
    text-align: right;
    flex-basis: 1.5em;
    flex-grow: 0;
    font-size: 75%;
    font-weight: 400;
    overflow: visible;
  }
}

#app[data-prefered-sizing-currency='base'] .trade .trade__amount {
  .trade__amount__quote {
    transform: translateX(-25%);
    opacity: 0;
  }

  .trade__amount__base {
    transform: none;
    opacity: 1;
  }

  &:hover {
    > span.trade__amount__base {
      transform: translateX(25%);
      opacity: 0;
    }

    > span.trade__amount__quote {
      transform: none;
      opacity: 1;
    }
  }
}

#app.-light .trade.-level-3 {
  box-shadow: 0 0 20px rgba(white, 0.5);
}
</style>
