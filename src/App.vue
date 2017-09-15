<template>
  <div class="container">
    <div class="right-container">
      <div class="gantt-selected-info">
        <div v-if="selectedTask">
          <h2>{{ selectedTask.text }}</h2>
          <ul>
            <li><strong>Start Date: </strong>{{ selectedTask.start_date | niceDate }}</li>
            <li><strong>End Date: </strong>{{ selectedTask.end_date | niceDate }}</li>
            <li><strong>Concurrent SVMs: </strong>{{ selectedTask.svms }}</li>
            <li><strong>Storage: </strong>{{ selectedTask.storage }} GB</li>
            <li><a :href="'http://login.salesforce.com/' + selectedTask.salesforce_id" target="_blank"><strong>Open in Salesforce</strong></a></li>
            <li><a :href="'https://cloud.skytap.com/configurations/' + selectedTask.skytap_environment_id" target="_blank"><strong>Open in Skytap</strong></a></li>
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
    <gantt class="left-container" :tasks="tasks" @task-updated="logTaskUpdate" @link-updated="logLinkUpdate" @task-selected="selectTask"></gantt>
  </div>
</template>

<script>
import Gantt from './components/Gantt.vue'

export default {
  name: 'app',
  components: {
    Gantt
  },
  data () {
    return {
      tasks: {
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
      selectedTask: null,
      messages: []
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
    selectTask (task) {
      this.selectedTask = task
    },

    addMessage (message) {
      this.messages.unshift(message)
      if(this.messages.length > 40) {
        this.messages.pop()
      }
    },

    logTaskUpdate (id, mode, task) {
      let text = (task && task.text ? ` (${task.text})`: '')
      let message = `Task ${mode}: ${id} ${text}`
      this.addMessage(message)
    },

    logLinkUpdate (id, mode, link) {
      let message = `Link ${mode}: ${id}`
      if(link){
        message += ` ( source: ${link.source}, target: ${link.target} )`
      }
      this.addMessage(message)
    }
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
