<!-- TODO Make spotting troublesome days easier by adding concurrent svms per day view, as colored daily events -->
<!-- TODO Add loading indicator -->
<!-- TODO Simplify format of data returned by API. No need to group by data center if calendar view is used -->

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

    <section class="section" v-show="errorMessage">
     <div class="container">
        <article class="message is-danger">
          <div class="message-header">
            <p>Error</p>
            <button class="delete" aria-label="delete"  @click="errorMessage = ''"></button>
          </div>
          <div class="message-body">
            <p>An unexpected error occurred. Please check the <strong>Browser Console</strong> in the <strong>Developer Tools</strong> for more details.</p>
            <p>The exact error message is: <strong>{{ errorMessage }}</strong></p>
          </div>
        </article>
      </div>
    </section>

    <section class="section">
      <div class="container">
        <div class="columns">
          <div class="column is-three-quarters">
            <a class="button is-primary" :class="{ 'is-loading': loading }" @click="fetchUpcomingIltClasses">Fetch upcoming ILT classes from Salesforce</a>
            <full-calendar :events="events" locale="en" @eventClick="handleEventClick" @changeMonth="handleChangeMonth" @dayClick="handleDayClick" @moreClick="handleMoreClick">
              <template slot="fc-event-card" scope="data">
                  <p><i class="fa">sadfsd</i> {{ data.event.title }} test</p>
              </template>
            </full-calendar>
          </div>
          <div class="column">
            <h3 class="title">Class Details</h3>
            <div v-if="selectedIltClass">
              <p><strong>Name: </strong>{{ selectedIltClass.title}}</p>
              <p><strong>Start Date: </strong>{{ selectedIltClass.start }}</p>
              <p><strong>End Date: </strong>{{ selectedIltClass.end }}</p>
              <p><strong>Concurrent SVMs: </strong>{{ selectedIltClass.extra.svms }}</p>
              <p><strong>Storage: </strong>{{ selectedIltClass.extra.storage }} GB</p>
              <p><a :href="'http://login.salesforce.com/' + selectedIltClass.extra.salesforce_id" target="_blank"><strong>[open in Salesforce]</strong></a></p>
              <p><a :href="'https://cloud.skytap.com/configurations/' + selectedIltClass.extra.skytap_environment_id" target="_blank"><strong>[open in Skytap]</strong></a></p>
            </div>
            <div v-else>
              <p>Select a class to display more details...</p>
            </div>
          </div>
        </div>
      </div>
    </section>

  </div>
</template>

<script>
import axios from 'axios'
import _ from 'lodash'
import moment from 'moment'

// TODO Get this IP address from ENV in Dockerfile?
const TRAINING_OPS_SERVER_IP = 'localhost'  // '10.42.100.179' or 'localhost' for local testing
const HTTP_REST_API = axios.create({
  baseURL: `http://${TRAINING_OPS_SERVER_IP}:5050/api/`,
  headers: {
    'Access-Control-Allow-Origin': '*' // for CORS to work properly
  }
})

function logAxiosError (error) {
  if (error.response) {
    console.log('The request was made and the server responded with a status code that falls out of the range of 2xx')
    console.log(error.response.data)
    console.log(error.response.status)
    console.log(error.response.headers)
  } else if (error.request) {
    console.log('The request was made but no response was received')
    // `error.request` is an instance of XMLHttpRequest in the browser and an
    // instance of http.ClientRequest in node.js
    console.log(error.request)
  } else {
    console.log('Something happened in setting up the request that triggered an error')
    console.log('Error', error.message)
  }
  console.log(error.config)
}

function salesforcesDate2IsoDate (salesforcesDate) {
  // Convert "2017-09-20" (from Salesforce) to "20-09-2017" (for dhtmlxGantt)
  return moment(salesforcesDate, "YYYY-MM-DD").format("DD-MM-YYYY")
}

export default {
  name: 'app',
  components: {
    'full-calendar': require('vue-fullcalendar')
  },
  data: () => ({
    rawData: {},
    selectedIltClass: null,
    loading: false,
    errorMessage: '',
  }),
  computed: {
    events: function () {
      let upcomingIltClasses = []

      // For each region...
      _.forEach(this.rawData, function (iltClasses, dataCenter) {

        // ..add ILT classes as events in the calendar
        _.forEach(iltClasses, function (iltClass) {

          upcomingIltClasses.push({
            title   : iltClass.Full_Name__c,
            start   : iltClass.Start_Date__c,
            end     : iltClass.End_Date__c,
            cssClass: [dataCenter],
            extra: {
              region: dataCenter,
              svms: iltClass.svms,
              storage: iltClass.storage,
              salesforce_id: iltClass.Id,
              skytap_environment_id: iltClass.Environment_ID__c
            }
          })
        })
      })

      return upcomingIltClasses
    }
  },
  methods: {
    fetchUpcomingIltClasses () {
      // Fetch upcoming ILT classes from REST API and display them in calendar
      console.log('Fetching upcoming ILT classes from Salesforce...')
      this.loading = true
      HTTP_REST_API.get('/upcoming-ilt-classes')
        .then(response => {
          this.loading = false
          this.rawData = response.data
        })
        .catch(e => {
          this.loading = false
          this.errorMessage = e.message
          logAxiosError(e)
        })
    },
    // Calendar events
    handleChangeMonth (start, end, current) {
      console.log('changeMonth', start, end, current)
    },
    handleEventClick (event, jsEvent, pos) {
      console.log('eventClick', event, jsEvent, pos)
      this.selectedIltClass = event
    },
    handleDayClick (day, jsEvent) {
      console.log('dayClick', day, jsEvent)
    },
    handleMoreClick (day, events, jsEvent) {
      console.log('moreCLick', day, events, jsEvent)
    }
  },
  created () {
    fetchUpcomingIltClasses()
  }
}
</script>

<style>
@import "~bulma/css/bulma.css";

/* Calendar */
.full-calendar-body .dates .dates-events .events-week .events-day .event-box p.APAC {
  color: #ffffff;
  background-color: #FFBC42;
}

.full-calendar-body .dates .dates-events .events-week .events-day .event-box p.EMEA {
  color: #ffffff;
  background-color: #0496FF;
}

.full-calendar-body .dates .dates-events .events-week .events-day .event-box p.US-East {
  color: #ffffff;
  background-color: #d81159;
}

.full-calendar-body .dates .dates-events .events-week .events-day .event-box p.US-West {
  color: #ffffff;
  background-color: #04e762;
}
</style>
