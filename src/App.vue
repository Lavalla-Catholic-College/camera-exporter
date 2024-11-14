<template>
  <div class="container">
    <h1>Camera Footage Exporter</h1>
    <div class="row" v-if="!token">
      <div class="col">
        <div class="form-group">
          <label for="username" class="form-label">NVR</label>
          <input type="text" class="form-control" id="nvr" name="nvr" v-model="nvr">
        </div>
        <div class="form-group">
          <label for="username" class="form-label">Username</label>
          <input type="text" class="form-control" id="username" name="username" v-model="username">
        </div>
        <div class="form-group">
          <label for="password" class="form-label">Password</label>
          <input type="password" class="form-control" id="password" name="password" v-model="password">
        </div>
        <div class="mt-3">
          <button class="btn btn-primary" v-on:click="login()">Login</button>
        </div>
      </div>
    </div>
    <div v-if="token">
      <div class="row">
        <div class="col-6">
          <div class="form-group">
            <label for="cameras" class="form-label">Camera</label>
            <select class="form-control" name="camera" id="camera" v-model="selectedCamera" size="10">
              <option v-for="camera in cameras" :key="camera" :value="camera">{{ camera.name }}</option>
            </select>
          </div>
        </div>
        <div class="col-6">
          <div class="form-group">
            <label for="thumbnail" class="form-label">Thumbnail</label>
            <img class="img-fluid img-thumbnail camera-thumbnail" :src="cameraThumbnail">
          </div>
        </div>
      </div>
      <div class="row mt-4">
        <div class="col-6">
          <div class="form-group">
            <label for="footageDate" class="form-label">Date</label>
            <input class="form-control" type="datetime-local" v-model="footageDate">
          </div>
          <div class="form-group mt-4">
            <label for="duration" class="form-label">Duration (in minutes)</label>
            <input class="form-control" type="number" min="1" v-model="duration">
          </div>
        </div>
        <div class="col-6">
          <div class="form-group">
            <label for="footageDate" class="form-label">Shortcuts</label>
            <div class="form-control">
              <button v-on:click="modifyDate(key)" class="btn btn-primary" v-for="shortcut, key in dateShortcuts"
                :key="shortcut">{{ key }}</button>
            </div>
          </div>
        </div>
      </div>
      <div class="row">
        <div class="col">
          <button class="btn btn-primary" v-on:click="download">Download Footage</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style></style>

<script>




export default {
  name: 'App',
  beforeMount() {
    dayjs.extend(window.dayjs_plugin_objectSupport)
  },
  data() {
    return {
      nvr: '10.12.10.100:7001', // The URL of the NVR. 
      username: 'admin', // The username used to log into the NVR
      password: '', // The password used to log into the NVR
      token: null, // The token used to communicate with the NVR
      cameras: [], // The list of cameras
      selectedCamera: [], // The cameras we've selected
      thumbnail: '', // The thumbnail from the selected camera
      loggedIn: false, // If we're logged in or not. Assume not by default
      duration: 60, // How many minutes of footage to get
      footageDateObject: new dayjs().subtract(1, 'hour'), // Used for dates and times. Default to one hour ago.
      dateShortcuts: { // Shortcut buttons to adjust times and such
        'Before School': {
          'time': { hour: 8, minute: 0 },
          'duration': 60
        },
        'Homeroom': {
          'time': { hour: 9, minute: 0 },
          'duration': 30
        },
        'Period 1': {
          'time': { hour: 9, minute: 15 },
          'duration': 60
        },
        'Period 2': {
          'time': { hour: 10, minute: 10 },
          'duration': 60
        },
        'Recess': {
          'time': { hour: 11, minute: 10 },
          'duration': 40
        },
        'Period 3': {
          'time': { hour: 11, minute: 30 },
          'duration': 70
        },
        'Period 4': {
          'time': { hour: 12, minute: 30 },
          'duration': 70
        },
        'Lunch': {
          'time': { hour: 13, minute: 25 },
          'duration': 60
        },
        'Period 5': {
          'time': { hour: 14, minute: 10 },
          'duration': 70
        },
        'After School': {
          'time': { hour: 15, minute: 10 },
          'duration': 60
        }
      }
    }
  },
  computed: {
    cameraThumbnail() {
      return 'https://' + this.nvr + '/rest/v2/devices/' + this.selectedCamera?.id + '/image?size=320x240'
    },
    footageDate() {
      return this.footageDateObject.format('YYYY-MM-DDTHH:mm:ss')
    }
  },
  methods: {

    // Modifies the footageDateObject. Used to give shortcut buttons like "start of day", "1 hour ago" etc.
    modifyDate(dateShortcut) {
      const ds = this.dateShortcuts[dateShortcut]
      this.duration = ds['duration']
      this.footageDateObject = new dayjs(ds['time'])
    },

    async login() {
      // TODO: Make this robust. If you try and log in when you're already logged in, it'll
      // just throw an error and you'll clear the token. The API tells you your session has expired
      // if you try and log in again, even if it's not actually expired, so perhaps call the API to logout then clear the token?
      try {
        const fetchObject = await fetch('https://' + this.nvr + '/rest/v2/login/sessions', {
          method: 'POST',
          body: JSON.stringify({
            'username': this.username,
            'password': this.password,
            'setCookie': true
          }),
          mode: 'cors',
          headers: {
            'Content-Type': 'application/json'
          }
        })

        // Get the JSON response
        const response = await fetchObject.json()

        // If the response has a token, set it, otherwise, clear it if it's already set
        if (response?.['token']) {
          this.token = response['token']
          this.getCameras()
        } else {
          alert('Unable to log in. Is your username and password correct?')
          this.token = null
        }

      } catch (error) {
        alert('Error logging in. Check the web console for more information')
        console.error(error)
        this.token = null
      }
    },
    async getCameras() {
      // TODO: Set an alert or something
      if (!this.token) {
        return false
      }

      try {
        const fetchObject = await fetch('https://' + this.nvr + '/rest/v2/devices?_orderBy=name&deviceType=Camera', {
          mode: 'cors',
          credentials: 'include',
        })

        // Now we loop through all the cameras
        const cameraList = await fetchObject.json()

        this.cameras = cameraList
      } catch (ex) {
        console.error(ex)
      }


    },

    // Downloads footage from a camera
    async download() {
      if (!this.selectedCamera) {
        alert('You must select a camera!')
        return false
      }

      const response = await fetch(`https://${this.nvr}/hls/${this.selectedCamera.id}.mkv?pos=${this.footageDate}&duration=${this.duration}`, {
        mode: 'no-cors',
        credentials: 'include',
        headers: {
          'Content-Disposition': `attachment; filename="${this.selectedCamera.name}"`,
          'Access-Control-Allow-Origin': true
        }
      })
      const blob = await response.blob();
      const objectUrl = URL.createObjectURL(blob);

      const link = document.createElement('a');
      link.href = objectUrl;
      link.download = this.selectedCamera.name; // the default filename when the user saves the file
      link.click();

    }

  }
}
</script>

<style></style>
