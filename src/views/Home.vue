<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>Keyless 🔑</ion-title>
      </ion-toolbar>
    </ion-header>

    <!--    <ion- -->

    <ion-content :fullscreen="true">
      <div id="container">
        <strong>Is Keyless working?</strong>
        <div style="margin-top: 1rem">
          Monitoring enabled:
          <span style="color: green" v-if="isMonitoring">Yes</span>
          <span style="color: red" v-else>No</span>
        </div>
        <div style="margin-top: 1rem">
          In region:
          <span style="color: green" v-if="inRegion">Yes</span>
          <span style="color: red" v-else>No</span>
        </div>
        <div style="margin-top: 1rem">
          Is ranging:
          <span style="color: green" v-if="isRanging">Yes</span>
          <span style="color: red" v-else>No</span>
        </div>
        <div style="margin-top: 1rem">
          Is near:
          <span style="color: green" v-if="isNear">Yes</span>
          <span style="color: red" v-else>No</span>
        </div>

        <div style="margin-top: 1rem">
          RSSI: {{ beaconState?.rssi }}<br>
          Accuracy: {{ beaconState?.accuracy }}<br>
          Proximity: {{ beaconState?.proximity }}<br>
        </div>

        <div style="margin-top: 1rem">
          Last request: {{ lastRequest }}
        </div>

        <div style="margin-top: 2rem" v-if="debug">
          <strong>Debug</strong><br>
          {{ debug }}
        </div>
      </div>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import { IonContent, IonHeader, IonPage, IonTitle, IonToolbar } from '@ionic/vue';
import { defineComponent } from 'vue';
import { IBeacon, IBeaconPluginResult } from "@ionic-native/ibeacon";
import { filter, tap } from 'rxjs/operators';

const BEACON_ID = 'c29ce823-e67a-4e71-bff2-abaa32e77a98';
const BEACON_NAME = 'door';

const filterRegion = filter((data: IBeaconPluginResult) => {
  return data.region.identifier === BEACON_NAME
});

const log = (message: string) => tap((data: IBeaconPluginResult) => {
  console.log(message);
  console.debug(data);
});

export default defineComponent({
  name:       'Home',
  components: {
    IonContent,
    IonHeader,
    IonPage,
    IonTitle,
    IonToolbar
  },
  data() {
    return {
      region:  IBeacon.BeaconRegion(BEACON_NAME, BEACON_ID),
      service: IBeacon.Delegate(),

      beaconState: {},

      isMonitoring: false,
      isRanging:    false,
      inRegion:     false,
      isNear:       false,
      lastRequest:  '-',
      debug:        ''
    }
  },
  methods:    {
    async prepare() {
      // this.service.didEnterRegion()
      //     .pipe(filterRegion)
      //     .subscribe(() => console.log('✅ didEnterRegion'))
      //
      // this.service.didExitRegion()
      //     .pipe(filterRegion)
      //     .subscribe(() => console.log('✅ didExitRegion'))

      this.service.monitoringDidFailForRegionWithError()
          .pipe(log('🚨 Monitoring failed'))
          .subscribe(() => this.isMonitoring = false);

      this.service.didStartMonitoringForRegion()
          .pipe(filterRegion)
          .pipe(log('✅ Started monitoring for iBeacon'))
          .subscribe(() => this.isMonitoring = true);

      this.service.didDetermineStateForRegion()
          .pipe(filterRegion)
          .pipe(log('✅ Region state changed'))
          .subscribe(data => this.inRegion = data.state === 'CLRegionStateInside');

      this.service.didRangeBeaconsInRegion()
          .pipe(filterRegion)
          .pipe(log('✅ iBeacons detected'))
          .subscribe(data => this.beaconState = data.beacons[0]);
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
    }
  },
  mounted() {
    this.prepare()
    this.start()
  },
  watch:      {
    inRegion() {
      this.toggleRanging();
    },

    beaconState(newValue) {
      this.isNear = newValue?.proximity === 'ProximityNear' ||
                    newValue?.proximity === 'ProximityImmediate'
    }
  },
});
</script>

<style scoped>
#container {
  text-align: center;

  position: absolute;
  left: 0;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
}

#container strong {
  font-size: 20px;
  line-height: 26px;
}
</style>