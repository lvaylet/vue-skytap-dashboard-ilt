<template>
  <div ref="gantt"></div>
</template>

<script>
import 'dhtmlx-gantt'

export default {
  name: 'gantt',
  props: {
    tasks: {
      type: Object,
      default () {
        return {
          data: [],
          links: []
        }
      }
    }
  },

  mounted () {

    gantt.attachEvent('onTaskSelected', (id) => {
      let task = gantt.getTask(id)
      this.$emit('task-selected', task)
    })

    gantt.attachEvent('onAfterTaskAdd', (id, task) => {
      this.$emit('task-updated', id, 'inserted', task)
      task.progress = task.progress || 0
      if(gantt.getSelectedId() == id) {
        this.$emit('task-selected', task)
      }
    })

    gantt.attachEvent('onAfterTaskUpdate', (id, task) => {
      this.$emit('task-updated', id, 'updated', task)
    })

    gantt.attachEvent('onAfterTaskDelete', (id) => {
      this.$emit('task-updated', id, 'deleted')
      if(!gantt.getSelectedId()) {
        this.$emit('task-selected', null)
      }
    })

    gantt.attachEvent('onAfterLinkAdd', (id, link) => {
      this.$emit('link-updated', id, 'inserted', link)
    })

    gantt.attachEvent('onAfterLinkUpdate', (id, link) => {
      this.$emit('link-updated', id, 'updated', link)
    })

    gantt.attachEvent('onAfterLinkDelete', (id, link) => {
      this.$emit('link-updated', id, 'deleted')
    })

    // FIXME How to update the fields in gantt.config based on the ones defined
    // in $props.config without overwriting gantt.config entirely?
    //gantt.config.readonly = this.$props.config.readonly

    // Set diagram as read only
    gantt.config.readonly = true

    // force the Gantt chart to automatically change its size to show all tasks
    // without scrolling
    gantt.config.autosize = "xy"

    // Set date format to be the same as Salesforce
    //gantt.config.api_date = '%Y-%m-%d'

    // Display 21 days ahead, even if the ILT classes span less than this duration
    let today = new Date()
    let todayPlus21Days = new Date().setDate(today.getDate() + 21)
    gantt.config.start_date = today
    gantt.config.end_date = todayPlus21Days

    // Configures columns
    // Reference: https://docs.dhtmlx.com/gantt/api__gantt_columns_config.html
    gantt.config.columns = [
      {
        name: "text",
        label: "ILT Classes",
        tree: true,
        width: "*"  // == use the remaining space
      }
    ]

    // Initialize diagram and parse data
    gantt.init(this.$refs.gantt)
    gantt.parse(this.$props.tasks)
  },
  watch: {
    tasks: function (newValue) {
      console.log('Refreshing Gantt chart with new data...')
      gantt.clearAll()
      gantt.parse(newValue)
    }
  }
}
</script>

<style>
	@import "~dhtmlx-gantt/codebase/dhtmlxgantt.css";
</style>
