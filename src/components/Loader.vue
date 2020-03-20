<template lang="pug">
.loader
  .loader__import
    section.loader__url
      input.loader__url-input(
        v-for="input in urlInputs"
        ref="entries"
        type="text"
        placeholder="Enter calendar URL"
        @input="onUrlInput"
      )

      input.loader__url-submit(
        type="submit"
        value="Load URLs"
        @click="onUrlSubmit"
      )

    section.loader__dropzone
      .loader__dropzone-inner
        span Click here or drag and drop some files [.ics]
        input.loader__dropzone-input(
          ref="fileInput"
          type="file"
          accept=".ics"
          @change="onFileInput"
          multiple
        )

  .loader__files(v-if="calendars.length > 0")
    files
</template>

<script>
import axios from 'axios'
import * as moment from 'moment'
import { mapGetters } from 'vuex'
import { icsToJson } from '@/utils/ics'
import { b64ToString } from '@/utils/string'
import { getEventSummary } from '@/utils/events'
import Files from '@/components/Files'

export default {
  components: { Files },

  computed: {
    ...mapGetters(['calendars'])
  },

  data () {
    return {
      urlInputs: 1
    }
  },

  methods: {
    onUrlInput () {
      const { entries } = this.$refs
      const emptyEntries = entries.filter(it => !it.value)

      if (emptyEntries.length === 0) {
        this.urlInputs++
      }
    },

    async onUrlSubmit () {
      const proxyUrl = 'https://cors-anywhere.herokuapp.com/'
      const { entries } = this.$refs

      const requests = entries
        .filter(({ value }) => value)
        .map(({ value }) => axios.get(`${proxyUrl}${value}`))

      try {
        const responses = await Promise.all(requests)
        const icsTexts = responses.map(({ data }) => data)
        const calendars = icsTexts.map(icsToJson)
        await this.$store.dispatch('addCalendars', calendars)
        this.$store.dispatch('selectCalendars', calendars)
        this.$emit('onLoad')
      } catch (error) {
        this.$emit('onError', error)
      }
    },

    async onFileInput () {
      const { fileInput } = this.$refs
      const { files } = fileInput

      try {
        this.validateFiles(files)
        const fileReaders = [...files].map((file) => this.readFromFile(file))
        const icsTexts = await Promise.all(fileReaders)

        const calendars = icsTexts.map(icsToJson)
        await this.$store.dispatch('addCalendars', calendars)
        this.$store.dispatch('selectCalendars', calendars)
        this.$emit('onLoad')
      } catch (error) {
        this.$emit('onError', error)
      }
    },

    validateFiles (files) {
      const filesAreValid = [...files].every(({ name }) => name.includes('.ics'))
      if (!filesAreValid) {
        throw new Error('Only ICS files are supported')
      } else {
        return true
      }
    },

    async readFromFile (file) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader()
        reader.onload = event => {
          const { target } = event
          const { result } = target
          // Decode base64
          resolve(b64ToString(result.split(',').pop()))
        }
        reader.onerror = error => reject(error)
        reader.readAsDataURL(file)
      })
    },

    getIcsJSONFromTexts (texts) {
      return texts.map(icsToJson)
    },

    getEventGroupsFromIcsTexts (icsTexts) {
      return icsTexts.reduce((eventGroups, icsText) => {
        const icsJSON = icsToJson(icsText)
        const { events } = icsJSON
        const group = this.computeEventGroups(events)
        eventGroups = { ...eventGroups, ...group }
        return eventGroups
      }, {})
    },

    computeEventGroups (events) {
      return events.reduce((list, event) => {
        const { startDate, endDate } = event
        const { value: summary } = getEventSummary(event)

        if (startDate && endDate) {
          if (!list[summary]) {
            list[summary] = {
              hours: 0,
              events: []
            }
          }

          const start = moment(startDate.value.trim().toUpperCase())
          const end = moment(endDate.value.trim().toUpperCase())

          // Take minutes in count
          const diffHours = end.diff(start, 'minutes') / 60
          list[summary].total += diffHours
          list[summary].events.push(event)
        }

        return list
      }, {})
    }
  }
}
</script>

<style lang="scss">
.loader {
  display: flex;
  flex: 1 1 auto;

  &__import {
    display: flex;
    flex-direction: column;
    width: 100%;
    flex: 1 1 auto;
  }

  &__url {
    width: 100%;
    padding: 0.5em 0.5em 0 0.5em;

    &-input {
      width: 100%;
      font-size: 1em;
      padding: 0.5em 0;
      border: none;
      border-top: solid 1px black;
    }

    &-submit {
      background-color: transparent;
      border: none;
      border-top: solid 1px black;
      border-bottom: solid 1px black;
      font-size: 1em;
      padding: 0.5em 0;
      width: 100%;
      text-align: left;
      cursor: pointer;
    }
  }

  &__dropzone {
    display: block;
    width: 100%;
    height: 100%;
    padding: 0.5em;

    &-inner {
      width: 100%;
      height: 100%;
      border: dashed 1px;
      position: relative;
      display: flex;
      align-items: center;
      justify-content: center;
    }

    &-input {
      cursor: pointer;
      opacity: 0;
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
    }
  }

  &__files {
    width: 30%;
    flex: 0 0 auto;
    padding: 0.5em;
  }
}
</style>