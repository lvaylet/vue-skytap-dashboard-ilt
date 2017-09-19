<template>
  <div class="container">
    <div class="right-container">
      <div class="gantt-selected-info">
        <div v-if="selectedIltClass">
          <h2>{{ selectedIltClass.text }}</h2>
          <ul>
            <li><strong>Start Date: </strong>{{ selectedIltClass.start_date | niceDate }}</li>
            <li><strong>End Date: </strong>{{ selectedIltClass.end_date | niceDate }}</li>
            <li><strong>Concurrent SVMs: </strong>{{ selectedIltClass.svms }}</li>
            <li><strong>Storage: </strong>{{ selectedIltClass.storage }} GB</li>
            <li><a :href="'http://login.salesforce.com/' + selectedIltClass.salesforce_id" target="_blank"><strong>Open in Salesforce</strong></a></li>
            <li><a :href="'https://cloud.skytap.com/configurations/' + selectedIltClass.skytap_environment_id" target="_blank"><strong>Open in Skytap</strong></a></li>
          </ul>
        </div>
        <div v-else class="select-task-prompt">
          <h2>Click any class...</h2>
        </div>
      </div>
      <ul class="gantt-messages">
        <li class="gantt-message" v-for="message in messages">{{ message }}</li>
      </ul>
    </div>
    <!-- @task-updated and @link-updated are defined for sample only, as the grid is read-only -->
    <gantt
      class="left-container"
      :tasks="upcomingIltClasses"
      @task-updated="logTaskUpdate"
      @link-updated="logLinkUpdate"
      @task-selected="selectIltClass"
    ></gantt>
  </div>
</template>

<script>
import Gantt from './components/Gantt.vue'
import axios from 'axios'

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

function raw2Gantt (data) {
  // Transform raw data from REST API to Gantt-compatible format.
  //
  // The output is a dictionary whose keys are region names ('APAC', 'EMEA', 'US-East' or 'US-West') and values are class details.
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
  //
  // The expected format for the Gantt chart is:
  //
  // [
  //   ...
  //   {
  //     id: 7,
  //     text: "US-East",
  //     open: true
  //   },
  //   {
  //     id: 8,
  //     text: "6.3 Big Data Advanced - Spark & 6.3 Big Data Advanced - MapReduce & 6.3 Big Data Basics Ed 2 @ TD Bank",
  //     start_date: "20-09-2017",
  //     duration: 2,
  //     open: true,
  //     parent: 7,
  //     svms: 390,
  //     storage: 1520,
  //     salesforce_id: "a9u390000004PLV",
  //     skytap_environment_id: "22779816"
  //   },
  //   ...
  // ]
  return {
    data: [
      {
        id: 1,
        text: "APAC",
        open: true
      },
      {
        id: 2,
        text: "6.3 DI Basics @ Duravit",
        start_date: "03-09-2017",
        duration:2,
        open: true,
        parent: 1,
        svms: 156,
        storage: 910,
        salesforce_id: "a9u390000004Cjj",
        skytap_environment_id: "22401248"
      },
      {
        id: 3,
        text: "ILT Class #2",
        start_date: "02-09-2017",
        duration:7,
        open: true,
        parent: 1,
        svms: 240,
        storage: 580
      }
    ]
  }
}

export default {
  name: 'app',
  components: {
    Gantt
  },
  data () {
    return {
      upcomingIltClasses: {
        data: [
          {
            id: 1,
            text: "APAC",
            open: true
          },
          {
            id: 2,
            text: "6.3 DI Basics @ Duravit",
            start_date: "03-09-2017",
            duration:2,
            open: true,
            parent: 1,
            svms: 156,
            storage: 910,
            salesforce_id: "a9u390000004Cjj",
            skytap_environment_id: "22401248"
          },
          {
            id: 3,
            text: "ILT Class #2",
            start_date: "02-09-2017",
            duration:7,
            open: true,
            parent: 1,
            svms: 240,
            storage: 580
          },
          {
            id: 4,
            text: "EMEA",
            open: true
          },
          {
            id: 5,
            text: "ILT Class #1",
            start_date: "17-09-2017",
            duration:3,
            open: true,
            parent: 4,
            svms: 520,
            storage: 980
          },
          {
            id: 6,
            text: "ILT Class #2",
            start_date: "05-09-2017",
            duration:4,
            open: true,
            parent: 4,
            svms: 860,
            storage: 1240
          },
          {
            id: 7,
            text: "US-East",
            open: true
          },
          {
            id: 8,
            text: "6.3 Big Data Advanced - Spark & 6.3 Big Data Advanced - MapReduce & 6.3 Big Data Basics Ed 2 @ TD Bank",
            start_date: "20-09-2017",
            duration: 2,
            open: true,
            parent: 7,
            svms: 390,
            storage: 1520,
            salesforce_id: "a9u390000004PLV",
            skytap_environment_id: "22779816"
          },
          {
            id: 9,
            text: "ILT Class #2",
            start_date: "10-09-2017",
            duration:4,
            open: true,
            parent: 7,
            svms: 860,
            storage: 1240
          },
          {
            id: 10,
            text: "US-West",
            open: true
          },
          {
            id: 11,
            text: "ILT Class #1",
            start_date: "01-09-2017",
            duration:2,
            open: true,
            parent: 10,
            svms: 520,
            storage: 980
          },
          {
            id: 12,
            text: "ILT Class #2",
            start_date: "03-09-2017",
            duration:2,
            open: true,
            parent: 10,
            svms: 860,
            storage: 1240
          }
        ]
      },
      selectedIltClass: null,
      messages: [],
      loading: false
    }
  },
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
  html, body {
    height: 100%;
    margin: 0;
    padding: 0;
    font-family: BlinkMacSystemFont,-apple-system,"Segoe UI","Roboto","Oxygen","Ubuntu","Cantarell","Fira Sans","Droid Sans","Helvetica Neue","Helvetica","Arial",sans-serif;
  }

  .container {
    height: 100%;
    width: 100%;
  }

  .left-container {
    overflow: hidden;
    position: relative;
    height: 100%;
  }

  .right-container {
    border-right: 1px solid #cecece;
    float: right;
    height: 100%;
    width: 340px;
    box-shadow: 0 0 5px 2px #aaa;
    position: relative;
    z-index:2;
  }

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
    /*font-family: Geneva, Arial, Helvetica, sans-serif;*/
    font-size: 14px;
    margin: 5px 0;
    padding: 8px 0 8px 10px;
  }

  .gantt-selected-info {
    /*border-bottom: 1px solid #cecece;*/
    box-sizing: border-box;
    /*font-family: Geneva, Arial, Helvetica, sans-serif;*/
    height: 50%;
    line-height: 28px;
    padding: 10px;
  }

  .gantt-selected-info h2 {
    border-bottom: 1px solid #cecece;
  }

  .select-task-prompt h2{
    color: #d9d9d9;
  }

  .gantt-selected-info ul {
    list-style: none;
    padding-left: 0;
  }
</style>
