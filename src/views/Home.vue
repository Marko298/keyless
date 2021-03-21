<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>Keyless 🔑</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true" v-if="service">
      <div class="container">
        <strong>Is Keyless working?</strong>

        <div>
          Monitoring enabled:
          <span style="color: green" v-if="isMonitoring">Yes</span>
          <span style="color: red" v-else>No</span>
        </div>

        <div>
          In region:
          <span style="color: green" v-if="inRegion">Yes</span>
          <span style="color: red" v-else>No</span>
        </div>

        <div>
          Is ranging:
          <span style="color: green" v-if="isRanging">Yes</span>
          <span style="color: red" v-else>No</span>
        </div>

        <div>
          Is near:
          <span style="color: green" v-if="isNear">Yes</span>
          <span style="color: red" v-else>No</span>
        </div>

        <div style="">
          <ion-label>Dry run</ion-label>
          <ion-toggle slot="start" name="Dry run" :checked="dryRun" @onIonChange="dryRun = !dryRun"/>
        </div>

        <div>
          RSSI: {{ beaconState?.rssi }}<br>
          Accuracy: {{ beaconState?.accuracy }}<br>
          Proximity: {{ beaconState?.proximity }}<br>
        </div>

        <div>
          Last request: {{ lastRequest }} <br>
          Latency: {{ latency }}ms
        </div>

        <div style="margin-top: 2rem" v-if="debug">
          <strong>Debug</strong><br>
          {{ debug }}
        </div>
      </div>
    </ion-content>
    <ion-content v-else>
      <div class="container">
        <strong>BLE not found 😢</strong>
      </div>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import { IonContent, IonHeader, IonLabel, IonPage, IonTitle, IonToggle, IonToolbar } from '@ionic/vue';
import { defineComponent } from 'vue';
import { IBeacon, IBeaconPluginResult } from "@ionic-native/ibeacon";
import { filter, tap } from 'rxjs/operators';
import axios from 'axios';
import moment from 'moment';

const REQUEST_INTERVAL = 2000;
const BEACON_ID = 'c29ce823-e67a-4e71-bff2-abaa32e77a98';
const BEACON_NAME = 'door';

const wait = (time: number) => new Promise((r) => setTimeout(r, time))

const filterRegion = filter((data: IBeaconPluginResult) => {
  return data.region.identifier === BEACON_NAME
});

const log = (message: string) => tap((data: IBeaconPluginResult) => {
  console.log(message);
  // console.debug(data);
});

export default defineComponent({
  name:       'Home',
  components: {
    IonContent,
    IonHeader,
    IonPage,
    IonTitle,
    IonToolbar,
    IonToggle,
    IonLabel,
  },
  data() {
    return {
      region:  IBeacon.BeaconRegion(BEACON_NAME, BEACON_ID),
      service: IBeacon.Delegate(),

      debug: '',

      beaconState: {},
      dryRun:      true,

      isMonitoring: false,
      isRanging:    false,
      inRegion:     false,
      isNear:       false,
      latency:      0,
      lastRequest:  '-',

      keyTimer: 0,
    }
  },
  methods:    {
    async prepare() {
      this.service.monitoringDidFailForRegionWithError().pipe(
          log('🚨 Monitoring failed')
      ).subscribe(() => this.isMonitoring = false);

      this.service.didStartMonitoringForRegion().pipe(
          filterRegion,
          log('✅ Started monitoring for iBeacon')
      )
          .subscribe(() => this.isMonitoring = true);

      this.service.didDetermineStateForRegion().pipe(
          filterRegion,
          log('✅ Region state changed')
      ).subscribe(data => this.inRegion = data.state === 'CLRegionStateInside');

      this.service.didRangeBeaconsInRegion().pipe(
          filterRegion,
          log('✅ iBeacons detected')
      ).subscribe(data => this.beaconState = data.beacons[0]);
    },

    async start() {
      await IBeacon.requestAlwaysAuthorization();
      await IBeacon.startMonitoringForRegion(this.region);
      await IBeacon.requestStateForRegion(this.region);
    },

    async startRanging() {
      await IBeacon.startRangingBeaconsInRegion(this.region);

      console.log('✅ Ranging started');

      this.isRanging = true;
    },

    async stopRanging() {
      await IBeacon.stopRangingBeaconsInRegion(this.region);

      console.log('✅ Stopped ranging');

      this.isRanging = false;
    },

    async toggleRanging() {
      if (this.inRegion && !this.isRanging) {
        return this.startRanging();
      }

      if (!this.inRegion && this.isRanging) {
        return this.stopRanging();
      }
    },

    async sendRequest() {
      const url = 'https://admin.nerdzlab.com/api/door-bot/link/W8bCIZIIpnLZ7ktjmyykHc8O5y55iSvTuxWZ0U8GaHRJwbDJkVGhzAjuW8UFGQwnWkPI85iTpLWvaX9k28ImQffMHwsAKEmj9Rrw'
      //const url = 'https://home.mar4ik.com/api/webhook/sbghjklsdfghvibehotew78vu32b985yfvb389r27b4fy1289vcbhjrwgfhkldbxLJZO'

      const t1 = +new Date();

      if (this.dryRun) {
        await wait(200);
      } else {
        await axios.post(url);
      }

      const t2 = +new Date();

      console.log('⏳ Request send');

      this.latency = t2 - t1;
      this.lastRequest = moment().format('hh:mm:ss')
    },

    startRequestTimer() {
      this.keyTimer = setInterval(() => this.sendRequest(), REQUEST_INTERVAL);
    },

    stopRequestTimer() {
      clearInterval(this.keyTimer);
      this.keyTimer = 0;
    },

    toggleKey() {
      if (this.keyTimer) {
        return this.stopRequestTimer();
      }

      this.startRequestTimer()
    }
  },
  mounted() {
    if (!this.service) {
      return;
    }

    this.prepare()
    this.start()
  },
  watch:      {
    inRegion() {
      this.toggleRanging();
    },

    isNear() {
      this.toggleKey();
    },

    beaconState(newValue) {
      this.isNear = newValue?.proximity === 'ProximityNear' ||
                    newValue?.proximity === 'ProximityImmediate'
    }
  },
});
</script>

<style scoped>
.container {
  text-align: center;

  position: absolute;
  left: 0;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
}

.container strong {
  font-size: 1.25rem;
  line-height: 1.5rem;
}

.container div {
  /*text-align: left;*/
  margin-top: 1rem
}
</style>