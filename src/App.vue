<template>
  <div id="app" v-cloak v-if="!fetching">
        <!-- MODDAL -->
    <transition name="quickfade">
      <div class="overlay formodal" v-if="showModal" v-cloak></div>
    </transition>

    <transition name="slidedown">
    <div v-on:click.self="showModal = false" class="modalContainer" v-if="showModal" v-cloak>

      <form v-if="modalContent == 'form'" v-on:submit.prevent="sendMeasurement" class="modal">
        <div class="formfield">
          <span class="temperature" v-on:click="useCelcius = !useCelcius">
            {{ currentLocation.name }}: 
            <span v-if="useCelcius">{{ inputTemperature }}°C<span class="switch">°F</span></span>
            <span v-else>{{ inputTemperature }}°F<span class="switch">°C</span></span>
          </span>
          <div class="rangewrap">
            <span class="increment" v-on:click="addTemperature(-1)">-</span>
            <input class="range" v-model.number="inputTemperature" type="range" :min="minmaxTemperature[0]" :max="minmaxTemperature[1]" step="1"/>
            <span class="increment" v-on:click="addTemperature(1)">+</span>
          </div>
        </div>
        <div class="submitContainer">
          <p class="warning brief" v-if="locationShouldWarn">
            Your closest location is {{ locations[closetsLocationIndex].name }}. Are you sure you mean to submit to {{ currentLocation.name }}?
          </p>
          <div :disabled="submitDisabled" class="modalButton" v-on:click="sendMeasurement">Submit</div>
        </div>
      </form>
      <div v-else-if="modalContent == 'details'" class="modal">
          <h3 class="center">{{ currentLocation.name }} measurements</h3>
          <transition-group
            v-if="currentMeasurements.length > 0"
            tag="ul"
            name="fade-in-list"
            class="measurementsList"
          >
            <li class="listItem" :key="measurement._id" v-for="measurement in currentMeasurements">
              <span>
                <span>{{ measurement.temperature | round }}{{ currentFormat }}</span>
                <span class="light small align">{{ measurement.timestamp | xTimeAgo }}</span>
              </span>
            </li>
        </transition-group>
        <div v-else class="warning center">No measurements!</div>
      </div>
    </div>
    </transition>

    <!-- TOP CITY MENU -->
    <transition name="slidedown" appear>
      <div class="topmenu" v-cloak>
        <div class="cities">
          <div :class="{ citybutton: true, selected: loc.id == mapSelected }" v-on:click="mapSelected = loc.id" v-for="loc in locations">
            <img class="cityicon" :src="loc.imgUrl" alt="">
            <div class="cityname">{{ loc.name }}</div>
          </div>
        </div>
        <transition name="slidedown">
          <div v-if="flashMessage.active" :class="flashMessage.className + ' flash'">{{ flashMessage.msg }}</div>
        </transition>
      </div>
    </transition>

    <!-- BOTTOM ÌNFORMATION BOX -->
    <transition name="slideup" appear>
      <div class="infocontainer" v-show="!animating" v-cloak>

        <div class="dataContainer">
          
          <div class="row1">
            <h2>{{ currentLocation.name }}</h2>

            <span class="time">{{ currentTime.hours | zeropad }}:{{ currentTime.minutes | zeropad }}:{{ currentTime.seconds | zeropad }}
            </span>
          </div>
          <div class="row2">
            <div>
              <div class="temperature" v-on:click="useCelcius = !useCelcius">
                <span v-if="useCelcius">
                  <span>{{ currentTemperature | round }}°C</span>
                  <span class="switch">°F</span>
                </span>
                <span v-else>
                  <span>{{ currentTemperature | round }}°F</span>
                  <span class="switch">°C</span>
                </span>
              </div>

              <div class="hilo">
                <div>HI: {{ currentHILO.hi | round }}{{ currentFormat }}</div>
                <div>LO: {{ currentHILO.lo | round }}{{ currentFormat }}</div>
              </div>
            </div>
          </div>

        </div>
        <div class="buttons">
          <div class="modalButton todetails" v-on:click="openModal('details')">Details</div>
          <div class="modalButton toform" v-on:click="openModal('form')">Submit</div>
        </div>
      </div>
    </transition>

    <!-- BACKGROUND MAP COMPONENT -->
    <Map id="map" v-on:started="animating = true" v-on:finished="animating = false" :locations="locations" :selected="mapSelected"></Map>
  </div>
</template>

<script>

import Map from './Map.vue'
import io from 'socket.io-client'

export default {
  components: { Map },
  name: 'app',
  data () {
    return {
      fetching: true,
      animating: true,
      showModal: false,
      modalContent: 'form',
      // index of selected location for map so
      // that map can rotate before ui changes
      mapSelected: 0,
      // index of selected location for ui
      selected: 0,
      useCelcius: true,
      userlocation: false,
      sending: false,
      disconnected: false,
      flashMessage: {
        active: false,
        className: "",
        msg: ""
      },
      inputTemperature: 0,
      locations: [],
      measurements: [],
      unixtime: 0,
    }
  },
  watch: {
    useCelcius: function(usingCelcius) {
      // when we change our degree type change the current measurement input value
      if (usingCelcius) {
        this.inputTemperature = Math.round(this.fahrenheitToCelcius(this.inputTemperature))
      } else {
        this.inputTemperature = Math.round(this.celciusToFahrenheit(this.inputTemperature))
      }
    },
    animating: function(value) {
      if (value === false) {
        // Rotation animation has stopped
        // time to let the ui know the new city
        this.selected = this.mapSelected
      }
    }
  },
  created: function() {
    // fetch inital measurement and location data
    this.fetchAndSetData()

    // initialize socket for server communication
    this.initSocket()

    // ask user for gps location
    if (navigator && navigator.geolocation) {
      navigator.geolocation.getCurrentPosition((position) => {
        const { latitude, longitude } = position.coords
        this.userlocation = [longitude, latitude]
      })
    }
    // update time in one second intervals
    setInterval(function(){
      this.unixtime = Date.now()
    }.bind(this), 1000)
  },
  computed: {
    currentFormat: function() {
      return this.useCelcius ? '°C' : '°F'
    },
    currentTime: function() {
      // get the time of the location we have selected
      const time = this.unixtime || 0
      const offset = this.currentLocation.utcOffsetHours || 0
      // shift a new date by the offset amount, so that the real
      // time will be in .getUTC, this seems backwards but it works..
      const locationtime = new Date(time + offset*60*60*1000)
      return {
        hours: locationtime.getUTCHours(),
        minutes: locationtime.getUTCMinutes(),
        seconds: locationtime.getUTCSeconds()
      }
    },
    currentLocation: function() {
      if (this.locations[this.selected] === void 0) {
        // if we do not have a location return a dummy
        return { id: 0, name: 'undefined' }
      }
      return this.locations[this.selected]
    },
    submitDisabled: function() {
      // should the submit button be disabled
      return this.sending || this.disconnected
    },
    minmaxTemperature: function() {
      // returns the max degrees of current setting
      if (this.useCelcius) {
        return [-100, 100]
      } else {
        return [this.celciusToFahrenheit(-100), this.celciusToFahrenheit(100)]
      }
    },
    currentMeasurements: function() {
      // the measurements of the selected city
      let currentM =
        // deep clone
        JSON.parse(JSON.stringify(this.measurements))
        // show only measurements from selected id
        .filter(row => row.locationId === this.selected)
        .sort((a, b) => a.timestamp < b.timestamp)

      // if we use fahrenheits convert first
      if (!this.useCelcius) {
        const convert = this.celciusToFahrenheit
        currentM = currentM.map(m => {
          m.temperature =
            convert(m.temperature)
          return m
        })
      }

      return currentM
    },
    currentHILO: function() {
      // current locations highest and lowest temperature value
      if (this.currentMeasurements.length < 1) return { hi: NaN, lo: NaN }
      
      const byTemp =
        this.currentMeasurements.slice()
        .sort((a, b) => a.temperature > b.temperature)

      const hi = byTemp[byTemp.length - 1].temperature
      const lo = byTemp[0].temperature
      return { hi, lo }
    },
    currentTemperature: function() {
      // the current locations latest temperature
      const measurements = this.currentMeasurements
      if (measurements.length === 0) return NaN
      return measurements[0].temperature
    },
    locationShouldWarn: function() {
      // if we have user location and user has selected
      // other than their own location, we will return true
      const closest = this.closetsLocationIndex
      return  closest !== -1 && closest !== this.selected
    },
    closetsLocationIndex: function() {
      const userloc = this.userlocation
      const locs = this.locations.slice()

      const getClosestLocation = (from, to) => {
        const distance = (x, y) => {
          return Math.sqrt(Math.pow(x[0]-y[0], 2) + Math.pow(x[1]-y[1], 2))
        }
        const locationsByDistance =
          to.sort((a, b) => { 
            return (
              distance(from, a.coordinates)
            > distance(from, b.coordinates)
            )
          })
        return locationsByDistance[0].id
      }

      if (userloc !== false && locs.length > 0) {
        const closest =
          getClosestLocation(userloc, locs)
        return closest
      }
      return -1
    }
  },
  filters: {
    xTimeAgo: function(unix) {
      const minuteDifference = Math.round((Date.now() - unix) / 1000 / 60)
      if (minuteDifference > 59) {
        const hourDifference = Math.round(minuteDifference / 60)
        return hourDifference + ' hours ago'
      }
      return minuteDifference + ' minutes ago'
    },
    round: function(value) {
      return Math.round(value)
    },
    zeropad: function(t) {
      if (t / 10 < 1) t = "0"+t
      return t
    }
  },
  methods: {
    openModal: function(content) {
      if (content) this.modalContent = content
      this.showModal = true
    },
    addTemperature: function(value) {
      // add values to the temperature input
      // not allowed to go beyond the minmax range
      const max = (a, b) => a > b ? a : b
      const min = (a, b) => a < b ? a : b
      const temprange = this.minmaxTemperature
      if (value > 0) {
        this.inputTemperature = min(temprange[1], this.inputTemperature + value)
      } else {
        this.inputTemperature = max(temprange[0], this.inputTemperature - 1)
      }
    },
    sendMeasurement: function() {
      if (this.submitDisabled) return
      // Since we already have a socket open let's use that
      // instead of a post request
      this.sending = true
      const input = this.inputTemperature
      this.socket.emit('add', {
        id: this.selected,
        temperature: this.useCelcius ? input : this.fahrenheitToCelcius(input)
      }, (res) => {
        this.showModal = false
        this.sending = false
        if (res.success){
          this.setFlashMessage("Thank you for your contribution!", "success")
        } else {
          this.setFlashMessage("There was an error sending data!", "error")
        }
      })
    },
    resetFlash: function() {
      this.flashMessage = {
        active: false,
        msg: "",
        className: ""
      }
    },
    setFlashMessage: function(msg, className, duration) {
      this.flashMessage = {
        active: true,
        className,
        msg
      }
      setTimeout(this.resetFlash, duration || 3000)
    },
    celciusToFahrenheit: function(celcius) {
      return (celcius * 9 / 5) + 32
    },
    fahrenheitToCelcius: function(fahrenheit) {
      return (fahrenheit - 32) * 5 / 9
    },
    receiveMeasurement: function(measurement) {
      console.log("received measurement", measurement)
      this.measurements.push(measurement)
    },
    fetchAndSetData: function() {
      return fetch('/data').then(res => res.json())
      .then((data) => {
        data.locations = data.locations.map(l => {
          l.imgUrl = require('./assets/img/'+l.slug+'.png')
          return l
        })
        this.locations = data.locations
        this.measurements = data.measurements
        this.fetching = false
        return new Promise((r) => r())
      })
      .catch((err) => {
        this.disconnected = true
        this.flashMessage = {
          active: true,
          className: 'error',
          msg: 'No connection to database. Try again later.'
        }
        this.fetching = false
      })
    },
    initSocket: function() {
      this.socket = io()
      // receive live data from the server
      this.socket.on('update', this.receiveMeasurement)

      const connectionError = (reason) => {
        this.disconnected = true
        this.flashMessage = {
          active: true,
          className: "error",
          msg: "Connection error. Try to refresh or come back later."
        }
      }

      // when socket has problems show a connection error
      this.socket.on('disconnect', connectionError)
      this.socket.on('connect_error', connectionError)
      this.socket.on('connect_timeout', connectionError)

      // when socket reconnects fetch new data and then
      // remove any lingering flash messages
      this.socket.on('reconnect', function() {
        this.fetchAndSetData()
        .then(() => {
          // after we fetched updated data remove disconnection
          this.disconnected = false
          this.resetFlash()
        })
      }.bind(this))
    }
  }
}
</script>

<style>
  * {
    box-sizing: border-box;
    -webkit-tap-highlight-color: rgba(255, 255, 255, 0);
  }
  [v-cloak]{display: none}
  html, body {
    font-family: -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,Oxygen-Sans,Ubuntu,Cantarell,"Helvetica Neue",sans-serif;
    font-size: 17px;
    word-spacing: 1px;
    color:#455;
    padding: 0;
    margin: 0;
  }
  .light {
    color:#9aa;
  }
  .small {
    font-size: 80%
  }
  .center {
    text-align: center
  }

  #app{
    position: relative;
    height: 100vh;
    width: 100vw;
    display: flex;
    flex-direction: column;
    background-color: #fff;
  }
  .citybutton {
    user-select: none;
    -moz-user-select: none;
    cursor: pointer;
    display: inline-flex;
    flex-direction: column;
    background-color: white;
    position: relative;
    margin: 10px;
    padding: 10px;
    min-width: 50px;
    transition: top 0.2s cubic-bezier(.25,.8,.25,1);
    box-shadow: 0 1px 3px rgba(0,0,0,0.12), 0 1px 2px rgba(0,0,0,0.24);
    border: 2px solid #bcc;
    top: 0px;
  }

  .citybutton:hover, .citybutton.selected {
    box-shadow: 0 8px 15px rgba(0,0,0,0.2), 0 5px 5px rgba(0,0,0,0.2);
  }
  .citybutton.selected{
    cursor: default;
    top: 15px;
  }
  .citybutton > *{
    pointer-events: none;
  }
  .cityicon {
    width: 50px;
    height: auto;
    display: block;
    margin: auto;
    padding: 5px;
  }
  .temperature {
    user-select: none;
    -moz-user-select: none;
    cursor: pointer;
    text-align: center;
    font-size: 28px;
    display: block
  }
  .temperature > span {
    display: inline-flex;
    align-items: center;
  }
  #map{
    position: absolute;
    top: 0;
    left: 0;
    z-index: 0;
  }
  .overlay {
    position: absolute;
    pointer-events: none;
    top: 0;
    left: 0;
    height: 100vh;
    width: 100vw;
  }
  .topmenu {
    text-align: center;
    z-index: 2;
    font-size: 15px;
  }
  .switch {
    font-size: 50%;
    padding-left: 5px;
    opacity: 0.5;
    cursor: pointer;
  }

  .infocontainer {
    z-index: 2;
    background-color:white;
    border: 2px solid #bcc;
    align-self: center;
    margin-top: auto;
    margin-bottom: 10px;
    padding: 7px 10px;
    min-width: 300px;
  }
  .infocontainer .dataContainer {
    display: flex;
    justify-content: space-between;
    padding: 5px 15px;
  }
  .infocontainer .row1 {
  }
  .infocontainer .row1  {
    text-align: center;
    flex-grow: 1;
    display: flex;
    flex-direction: column;
  }
  .infocontainer .temperature {
    text-align: left;
  }
  .infocontainer .row1 h2{
    margin:0;
  }
  .infocontainer .row2{
    flex-grow: 1;
    text-align: center;
  }
  .infocontainer .row2 > div {
    text-align: left;
    display: inline-block;
  }
  .infocontainer .buttons{
    display: flex;
  }
  .infocontainer .buttons div {
  }
  .infocontainer .buttons .todetails {
    margin-right: 10px;
  }
  .time {
    margin-top: auto;
    margin-bottom: 10px;
    font-size: 20px;
  }

  .flex{display: flex;}
  .submitContainer {
    display: flex;
    flex-direction: column;
  }
  .rangewrap{
    display: flex;
    margin: 10px -10px 10px -10px;
  }
  .range{
    flex-grow: 1;
    align-self: center;
  }
  .increment{
    user-select: none;
    -moz-user-select: none;
    font-size: 24px;
    padding: 10px;
    cursor: pointer;
  }
  .modalButton {
    user-select: none;
    -moz-user-select: none;
    cursor: pointer;
    padding: 7px 10px;
    text-align: center;
    margin:10px 0;
    display: inline-block;
    flex-grow: 1;
    transition: all .1s ease-out;
    position: relative;
    top: 0;
    box-shadow:
      3px 3px #bcc, /* bottom right */
      -2px -2px #bcc, /* top left */
      3px -2px #bcc, /* top right */
      -2px 3px #bcc /* bottom left */
  }
  .modalButton[disabled] {
    opacity: 0.5;
  }
  .modalButton:active {
    top:2px;
    box-shadow:
      1px 1px #bcc, /* bottom right */
      -2px -2px #bcc, /* top left */
      1px -2px #bcc, /* top right */
      -2px 1px #bcc /* bottom left */
  }
  .modalContainer {
    z-index: 10;
    position: absolute;
    top: 0;
    left: 0;
    height: 100vh;
    width: 100vw;
  }
  .overlay.formodal {
    z-index: 9;
    background-color: rgba(0,0,0,0.5);
  }
  .block { display: block }
  .modal {
    user-select: none;
    -moz-user-select: none;
    max-height: 70vh;
    background: white;
    border: 2px solid #bcc;
    padding: 20px 40px;
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    max-width: 95%;
    overflow-y: scroll;
  }
  .formfield {padding: 5px 0;}
  .warning {
    color: orange;
  }
  .brief{
    max-width: 300px;
  }
  .flash {
    text-align: center;
    position: relative;
    top: 15px;
    z-index: 3;
    width: 95%;
    max-width: 500px;
    margin: auto;
    color: white;
    font-weight: bold;
    padding: 10px 20px;
    box-shadow: 0 1px 3px rgba(0,0,0,0.12), 0 1px 2px rgba(0,0,0,0.24);
  }
  .flash.success {
    background: rgb(120, 200, 100);
  }
  .flash.error {
    background: rgb(230, 120, 100);
  }

  .slidedown-enter-active, .slidedown-leave-active {
    transform: translateY(0%);
    opacity: 1;
    transition: all .3s cubic-bezier(0.42, 0, 0.58, 1);
  }
  .slidedown-enter, .slidedown-leave-to{
    transform: translateY(-100%);
    opacity: 0;
    transition: all .3s cubic-bezier(0.42, 0, 0.58, 1);
  }

  .fade-enter-active, .fade-leave-active {
    opacity: 1;
    transition: all 0.2s cubic-bezier(0.42, 0, 0.58, 1);
  }
  .fade-enter, .fade-leave-to{
    opacity: 0;
    transition: all 0.2s cubic-bezier(0.42, 0, 0.58, 1);
  }

  .slideup-enter-active, .slideup-leave-active {
    transform: translateY(0%);
    opacity: 1;
    transition: all .3s cubic-bezier(0.42, 0, 0.58, 1);
  }
  .slideup-enter, .slideup-leave-to{
    transform: translateY(100%);
    opacity: 0;
    transition: all .3s cubic-bezier(0.42, 0, 0.58, 1);
  }
  .fade-in-list-enter, .fade-in-list-leave-to{
    opacity: 0;
    transform: translateX(-30px);
  }
  .fade-in-list-leave-active {
    position: absolute;
  }
  .measurementsList{
    list-style: none;
    padding-left: 0;
  }
  .listItem {
    transition: all 0.4s;
  }
  .listItem > span {
    display: flex;
    justify-content: space-evenly;
    align-items: center;
  }


  /* CUSTOM RANGE SLIDER STYLES */
  input[type=range] {
    -webkit-appearance: none; /* Hides the slider so that custom slider can be made */
    width: 100%; /* Specific width is required for Firefox. */
    background: transparent; /* Otherwise white in Chrome */
  }
  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; }
  input[type=range]:focus { outline: none; }
  input[type=range]::-moz-focus-outer { border: 0; }
  /* Special styling for WebKit/Blink */
  input[type=range]::-webkit-slider-thumb {
    box-shadow: 1px 1px 1px #111, 0px 0px 1px #0d0d0d;
    -webkit-appearance: none;
    border: 3px solid #bcc;
    height: 30px;
    width: 16px;
    border-radius: 0px;
    background: #fff;
    cursor: pointer;
    margin-top: -14px;
  }
  /* All the same stuff for Firefox */
  input[type=range]::-moz-range-thumb {
    box-shadow: 0 1px 3px rgba(0,0,0,0.12), 0 1px 2px rgba(0,0,0,0.24);
    border: 3px solid #bcc;
    height: 30px;
    width: 16px;
    border-radius: 0px;
    background: #bcc;
    cursor: pointer;
  }
  input[type=range]::-webkit-slider-runnable-track {
    width: 100%;
    height: 8.4px;
    cursor: pointer;
    background: #fff;
    border-radius: 0px;
    border: 0.2px solid #bcc;
  }

  input[type=range]:focus::-webkit-slider-runnable-track {
    background: #367ebd;
  }

  input[type=range]::-moz-range-track {
    width: 100%;
    height: 8.4px;
    cursor: pointer;
    background: #fff;
    border-radius: 0;
    border: 0.2px solid #bcc;
  }

  @media(max-width: 800px) {

    .topmenu .cities {
      display: flex;
    }
    .citybutton {
      width: 20%;
    }
    .cityname { display: none; }
    .cityicon {
      width: 100%;
      padding: 0;
    }
    .infocontainer, .modal {
      width: calc(100% - 20px);
    }
  }

</style>