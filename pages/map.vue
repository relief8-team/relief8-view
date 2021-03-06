<template>
  <div class="container">
    <div class="appbar">
      <div class="appbar-title">Survivors in Need of Relief</div>
    </div>
    <GmapMap
      ref="map"
      :center="{lat:26.4042366, lng: -80.1206704}"
      :zoom="14"
      :options="{mapTypeControl: false, gestureHandling: 'greedy'}"
      map-type-id="terrain"
      style="width: 100vw; height: 50vh"
    />
    <div class="resources">
      <h1 class="title">Important resources</h1>

      <div class="resource">
        <nuxt-link to="/hurricane-prep">
          <img src="/img/res1.png"/>
        </nuxt-link>
        <p class="resource-text">Hurricane Prep Checklist</p>
      </div>
      <div class="resource" v-on:click="displayResourceListings($event, 'hospitals')">
        <img src="/img/res2.png"/>
        <p class="resource-text">Medical Facilities</p>
      </div>
      <div class="resource" v-on:click="displayResourceListings($event, 'bloodBanks')">
        <img src="/img/res3.png"/>
        <p class="resource-text">Blood banks</p>
      </div>
      <div class="resource" v-on:click="displayResourceListings($event, 'shelters')">
        <img src="/img/res4.png"/>
        <p class="resource-text">Shelters</p>
      </div>
      <div class="resource" v-on:click="displayResourceListings($event, 'gasStations')">
        <img src="/img/res4.png"/>
        <p class="resource-text">Gas stations</p>
      </div>
      <div class="resource" v-on:click="displayResourceListings($event, 'pharmacies')">
        <img src="/img/res5.png"/>
        <p class="resource-text">Pharmacies</p>
      </div>
    </div>
  </div>
</template>

<script>
  import { gmapApi } from 'vue2-google-maps'

  export default {
    computed: {
      google: gmapApi,
    },
    mounted() {
      this.$geolocation.getCurrentPosition().then(async result => {
        // Get user current geolocation
        let location = {
          lat: result.coords.latitude,
          lng: result.coords.longitude,
        }

        // Save location
        this.userLocation = location

        // View unloaded?
        if (!this.$refs.map) {
          return
        }

        // Get Google map
        this.map = await this.$refs.map.$mapPromise

        // add a click event handler to the map object
        this.google.maps.event.addListener(this.map, 'click', event => {
          if (this.lastInfoWindow) {
            this.lastInfoWindow.close()
          }
        })

        // Load center of map
        this.map.setCenter(location)

        // Set map zoom
        this.map.setZoom(13)

        // Marker for yourself
        this.displayMyself(location)

        // Fetch with Axios
        try {
          const result = await this.$axios.$post('/map', {
            latitude: location.lat,
            longitude: location.lng,
          })

          this.displaySurvivors(result.survivors)

          // Save result for later
          this.result = result
        } catch (err) {
          alert(err + '\n\n' + err)
        }
      })
    },

    data: function () {
      return {
        result: {},
        markers: [],
      }
    },

    methods: {
      displayMyself(location) {
        var marker = new google.maps.Marker({
          position: location,
          map: this.map,
          title: 'me',
        })
      },

      clearMarkers() {
        for (var i = 0; i < this.markers.length; i++) {
          this.markers[i].setMap(null)
        }
        this.markers = []
      },

      displayResourceListings(element, resourceType) {
        // Get resources
        let resources = this.result[resourceType]

        // Not loaded yet?
        if (!resources) {
          this.$toast.show('Please wait...', {duration: 2000})
          return
        }

        // Clear previous selected
        if (this.previouslySelectedResource) {
          this.previouslySelectedResource.className = 'resource'
        }
        element.currentTarget.className = 'resource selected'

        this.previouslySelectedResource = element.currentTarget

        // Clear existing tooltips
        this.clearMarkers()

        for (var resource of resources) {
          var distanceFromMe = this.calcDistanceMiles(
            this.userLocation.lat,
            this.userLocation.lng,
            resource.location.lat,
            resource.location.lng,
          )

          var tooltipHtml =
            `
          <div class="tooltip">
            <h2 class="tooltipHeading">${resource.name}</h2>
              <p class="tooltipRole">${resourceType}</p>` +
            (resource.rating ? `<p>${resource.rating} stars</p>` : '') +
            `
              <br />
              <p class="tooltipAddress">${resource.vicinity}</p>
              <br />
              <p class="tooltipDistance">${Math.round(distanceFromMe * 100) /
            100} miles away</p>
            </div>`

          let infowindow = new google.maps.InfoWindow({
            content: tooltipHtml,
            maxWidth: 200,
          })

          let marker = new google.maps.Marker({
            position: resource.location,
            map: this.map,
            title: resource.name,
            icon: `/img/${resourceType}-icon.svg`,
          })
          this.markers.push(marker)
          marker.addListener('click', () => {
            if (this.lastInfoWindow) {
              this.lastInfoWindow.close()
            }

            infowindow.open(this.map, marker)
            this.lastInfoWindow = infowindow
          })
        }

        if (this.userLocation) {
          // Load center of map
          this.map.panTo(this.userLocation)

          // Set map zoom
          this.map.setZoom(11)
        }
      },

      calcDistanceMiles(lat1, lon1, lat2, lon2) {
        var R = 6371 // km
        var dLat = this.toRad(lat2 - lat1)
        var dLon = this.toRad(lon2 - lon1)
        var lat1 = this.toRad(lat1)
        var lat2 = this.toRad(lat2)

        var a =
          Math.sin(dLat / 2) * Math.sin(dLat / 2) +
          Math.sin(dLon / 2) *
          Math.sin(dLon / 2) *
          Math.cos(lat1) *
          Math.cos(lat2)
        var c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a))
        var d = R * c
        // Convert to miles
        return d * 0.621371
      },

      // Converts numeric degrees to radians
      toRad(Value) {
        return (Value * Math.PI) / 180
      },

      displaySurvivors(survivors) {
        for (var survivor of survivors) {
          var location = {
            lat: survivor.location.coordinates[1],
            lng: survivor.location.coordinates[0],
          }

          var distanceFromMe = this.calcDistanceMiles(
            this.userLocation.lat,
            this.userLocation.lng,
            location.lat,
            location.lng,
          )

          var tooltipHtml =
            `
          <div class="tooltip">
            <h2 class="tooltipHeading">${survivor.firstName} ${
              survivor.lastName
            }</h2>
              <p class="tooltipRole">Survivor</p>
              <br />
              <p class="tooltipAddress">${survivor.address}</p>
              <br />
              <p>Age: ${new Date().getFullYear() -
            survivor.dateOfBirth.year} </p>
              <p>Blood type: ${survivor.bloodType}</p>
              <br />
              <a href="tel:` +
            survivor.phoneNumber +
            `" class="call-button">CALL</a>
              <p class="tooltipDistance">${Math.round(distanceFromMe * 100) /
            100} miles away</p>
            </div>`

          let infowindow = new google.maps.InfoWindow({
            content: tooltipHtml,
            maxWidth: 200,
          })

          let marker = new google.maps.Marker({
            position: location,
            map: this.map,
            title: 'Angel',
            icon: '/img/angel.svg',
          })

          this.markers.push(marker)

          marker.addListener('click', () => {
            if (this.lastInfoWindow) {
              this.lastInfoWindow.close()
            }
            infowindow.open(this.map, marker)
            this.lastInfoWindow = infowindow
          })
        }
      },
    },
  }
</script>

<style>
  @font-face {
    font-family: "Raleway SemiBold";
    src: url("~static/fonts/raleway-font/Raleway-SemiBold.ttf");
  }

  /* Always set the map height explicitly to define the size of the div
  * element that contains the map. */

  #map {
    height: 100%;
  }

  .appbar {
    background: #ec2024;
    height: 53px;
  }

  .appbar-title {
    color: #fff;
    font-size: 21px;
    text-align: center;
    padding: 13px 0 0 0;
  }

  .title {
    color: #fff;
    font-size: 20px;
  }

  .tooltip {
    color: #232a5e;
    font-size: 12px;
  }

  .resource.selected {
    opacity: 0.5;
  }

  .tooltipHeading {
    font-weight: 500;
    font-size: 12px;
  }

  .call-button {
    color: #ec2024;
    float: right;
    display: block;
    margin-top: 5px;
    text-decoration: none;
    font-weight: 500;
  }

  .tooltip p {
    font-size: 12px;
    margin-top: 5px;
  }

  .tooltip br {
    height: 5px;
  }

  .tooltipRole {
    color: #ec2024;
    font-family: Helvetica Neue;
    text-transform: capitalize;
    font-weight: 500;
    font-family: 12px;
  }

  .resources {
    background: #f0f0f0;
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    grid-gap: 20px 0;
    padding: 0 15px;
  }

  .title {
    grid-column: 1 / -1;
  }

  .resources h1 {
    color: #232a5e;
    font-size: 27px;
    font-family: 'Raleway SemiBold';
    padding: 40px 0 5px;
    text-align: center;
  }

  /* Optional: Makes the sample page fill the window. */

  .resource-group {
    margin: 0 0 30px 0;
  }

  .resource {
    cursor: pointer;
    color: #231f20;
    font-size: 12px;
    display: grid;
    justify-items: center;
  }

  .tooltipDistance {
    font-weight: bold;
  }

  .resource-text {
    color: #231f20;
    text-align: center;
  }

  html,
  body {
    height: 100%;
    margin: 0;
    padding: 0;
  }
</style>
