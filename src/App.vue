<!-- TODO Make spotting troublesome days easier by adding concurrent svms per day view, in a dedicated row -->
<!-- TODO Add loading indicator -->

<template>
  <div id="app">
    <section class="hero is-info">
      <div class="hero-body">
        <div class="container">
          <h1 class="title">
            Upcoming ILT Classes
          </h1>
          <h2 class="subtitle">
            and the associated Skytap load
          </h2>
        </div>
      </div>
    </section>

    <section style="margin: 2em 0;">
<!--
      <div class="right-container">
        <ul class="gantt-messages">
          <li class="gantt-message" v-for="message in messages">{{ message }}</li>
        </ul>
      </div>
-->
      <!-- @task-updated and @link-updated are defined for sample only, as the grid is read-only -->
      <gantt
        :tasks="upcomingIltClasses"
        @task-updated="logTaskUpdate"
        @link-updated="logLinkUpdate"
        @task-selected="selectIltClass"
      ></gantt>
    </section>

    <!-- Modal -->
    <div class="modal" :class="{ 'is-active': showModal }">
      <div class="modal-background"></div>
      <div class="modal-card">
        <header class="modal-card-head">
          <p class="modal-card-title">Full Details</p>
          <button class="delete" aria-label="close" @click="showModal = false"></button>
        </header>
        <section class="modal-card-body">
          <ul v-if="selectedIltClass">
            <li><strong>{{ selectedIltClass.text}}</strong></li>
            <li><strong>Start Date: </strong>{{ selectedIltClass.start_date | niceDate }}</li>
            <li><strong>End Date: </strong>{{ selectedIltClass.end_date | niceDate }}</li>
            <li><strong>Concurrent SVMs: </strong>{{ selectedIltClass.svms }}</li>
            <li><strong>Storage: </strong>{{ selectedIltClass.storage }} GB</li>
            <li><a :href="'http://login.salesforce.com/' + selectedIltClass.salesforce_id" target="_blank"><strong>Open in Salesforce</strong></a></li>
            <li><a :href="'https://cloud.skytap.com/configurations/' + selectedIltClass.skytap_environment_id" target="_blank"><strong>Open in Skytap</strong></a></li>
          </ul>
        </section>
        <footer class="modal-card-foot">
          <button class="button is-info" @click="showModal = false">OK</button>
        </footer>
      </div>
    </div>
  </div>
</template>

<script>
import Gantt from './components/Gantt.vue'
import axios from 'axios'
import _ from 'lodash'
import moment from 'moment'

// TODO Get this IP address from ENV in Dockerfile
const TRAINING_OPS_SERVER_IP = 'localhost'  // '10.42.100.179'
const HTTP_REST_API = axios.create({
  baseURL: `http://${TRAINING_OPS_SERVER_IP}:5050/api/`,
  headers: {
    'Access-Control-Allow-Origin': '*' // for CORS to work properly
  }
})

function logAxiosError (error) {
  if (error.response) {
    // The request was made and the server responded with a status code that
    // falls out of the range of 2xx
    console.log(error.response.data)
    console.log(error.response.status)
    console.log(error.response.headers)
  } else if (error.request) {
    // The request was made but no response was received. `error.request` is an
    // instance of XMLHttpRequest in the browser and an instance of
    // http.ClientRequest in node.js.
    console.log(error.request)
  } else {
    // Something happened in setting up the request that triggered an error
    console.log('Error', error.message)
  }
  console.log(error.config)
}

function salesforcesDate2IsoDate (salesforcesDate) {
  // Convert "2017-09-20" (from Salesforce) to "20-09-2017" (for dhtmlxGantt)
  return moment(salesforcesDate, "YYYY-MM-DD").format("DD-MM-YYYY")
}

function raw2Gantt (data) {
  // Transform raw data from REST API to Gantt-compatible format.
  //
  // The output of the REST API is a dictionary whose keys are region names ('APAC', 'EMEA', 'US-East' or 'US-West') and values are class details.
  //
  // {
  //     "EMEA": [
  //         {
  //             "Id": "a9u390000004X5uAAE",
  //             "Name": "Orange Cameroun",
  //             "Full_Name__c": "ILT = 6.3 DI Advanced - FR @ Orange Cameroun, QU-10325182985 [Sep 19 - Sep 19, 2017]",
  //             "Start_Date__c": "2017-09-19",
  //             "End_Date__c": "2017-09-19",
  //             "Environment_ID__c": "22798724",
  //             "Attendee_Count__c": 6.0,
  //             "Region__c": "EMEA",
  //             "svms": 156,
  //             "storage": 910.0
  //         },
  //         {
  //             "Id": "a9u390000004XBOAA2",
  //             "Name": "Orange Cameroun",
  //             "Full_Name__c": "ILT = 6.4 DI Administration @ Orange Cameroun, QU-10325182985 [Sep 19 - Sep 19, 2017]",
  //             "Start_Date__c": "2017-09-19",
  //             "End_Date__c": "2017-09-19",
  //             "Environment_ID__c": "22798988",
  //             "Attendee_Count__c": 6.0,
  //             "Region__c": "EMEA",
  //             "svms": 182,
  //             "storage": 780.0
  //         }
  //     ],
  //     "US-East": [
  //         {
  //             "Id": "a9u390000004XBOAA3",
  //             "Name": "Tyson",
  //             "Full_Name__c": "ILT = 6.3 Big Data Basics @ Tyson, QU-10325182986 [Sep 19 - Sep 19, 2017]",
  //             "Start_Date__c": "2017-09-19",
  //             "End_Date__c": "2017-09-19",
  //             "Environment_ID__c": "22798990",
  //             "Attendee_Count__c": 4.0,
  //             "Region__c": "US-East",
  //             "svms": 168,
  //             "storage": 590.0
  //         }
  //     ]
  // }

  // Initialize result with empty data
  let result = {
    data: [],
    links: []
  }

  // Initialize result with all possible regions
  _.forEach(data, function (iltClasses, dataCenter) {

    result.data.push({
      id: dataCenter,
      text: dataCenter,
      open: true  // so the list of classes in this region is displayed on startup
    })

    // Add the ILT classes and link them to their parent (created above)
    _.forEach(iltClasses, function (iltClass) {

      // Expected format of a class:
      //   {
      //     id: 8,
      //     text: "6.3 Big Data Advanced - Spark & 6.3 Big Data Advanced - MapReduce & 6.3 Big Data Basics Ed 2 @ TD Bank",
      //     start_date: "20-09-2017",
      //     duration: 2,
      //     open: true,
      //     parent: "US-East",
      //     svms: 390,
      //     storage: 1520,
      //     salesforce_id: "a9u390000004PLV",
      //     skytap_environment_id: "22779816"
      //   }
      result.data.push({
        id: iltClass.Id,
        text: iltClass.Full_Name__c,
        start_date: salesforcesDate2IsoDate(iltClass.Start_Date__c),
        duration: iltClass.Total_Duration_Days__c,
        open: true,
        parent: dataCenter,
        svms: iltClass.svms,
        storage: iltClass.storage,
        salesforce_id: iltClass.Id,
        skytap_environment_id: iltClass.Environment_ID__c
      })
    })
  })

  return result
}

export default {
  name: 'app',
  components: {
    Gantt
  },
  data: () => ({
    upcomingIltClasses: {
      data: [],
      links: []
    },
    selectedIltClass: null,
    messages: [],
    loading: false,
    showModal: false
  }),
  filters: {
    toPercent (val) {
      if(!val) return '0'
      return Math.round((+val) * 100)
    },

    niceDate (aDate) {
      let day = ('0' + aDate.getDate()).slice(-2)
      let month = ('0' + (aDate.getMonth()+1)).slice(-2)
      let year = aDate.getFullYear();
      return `${year}-${month}-${day}`
    }
  },
  methods: {
    selectIltClass (iltClass) {
      this.selectedIltClass = iltClass
      this.showModal = true
    },

    addMessage (message) {
      this.messages.unshift(message)
      if(this.messages.length > 40) {
        this.messages.pop()
      }
    },

    logTaskUpdate (id, mode, iltClass) {
      let text = (iltClass && iltClass.text ? ` (${iltClass.text})`: '')
      let message = `Class ${mode}: ${id} ${text}`
      this.addMessage(message)
    },

    logLinkUpdate (id, mode, link) {
      let message = `Link ${mode}: ${id}`
      if(link){
        message += ` ( source: ${link.source}, target: ${link.target} )`
      }
      this.addMessage(message)
    }
  },
  created () {
    // Fetch results from REST API and display as Gantt chart
    this.loading = true
    HTTP_REST_API.get('/upcoming-ilt-classes')
      .then(response => {
        this.loading = false
        this.upcomingIltClasses = raw2Gantt(response.data)
      })
      .catch(e => {
        this.loading = false
        logAxiosError(e)
      })
  }
}
</script>

<style>
@import "~bulma/css/bulma.css";

/*
html, body {
  height: 100%;
  margin: 0;
  padding: 0;
  font-family: BlinkMacSystemFont,-apple-system,"Segoe UI","Roboto","Oxygen","Ubuntu","Cantarell","Fira Sans","Droid Sans","Helvetica Neue","Helvetica","Arial",sans-serif;
}
*/

.gantt-messages {
  list-style-type: none;
  height: 50%;
  margin: 0;
  overflow-x: hidden;
  overflow-y: auto;
  padding-left: 5px;
}

.gantt-messages > .gantt-message {
  background-color: #f4f4f4;
  box-shadow:inset 5px 0 #d69000;
  font-size: 14px;
  margin: 5px 0;
  padding: 8px 0 8px 10px;
}
</style>
