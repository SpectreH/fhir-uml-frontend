<template>
  <v-container class="fill-height">
    <v-responsive class="align-center fill-height mx-auto">
      <v-row style="margin: 5px;">
        <v-col cols="2">
          <v-btn
            variant="outlined"
            class="w-100 pa-1"
            color="blue"
            @click="onGenerate"
          >
            Generate
          </v-btn>

          <v-select
            v-model="mode"
            label="Mode"
            :items="['FHIR -> UML', 'UML -> FHIR']"
            variant="outlined"
            class="pa-2 mt-5"
            density="compact"
          />

          <v-select
            v-model="view"
            label="View"
            :items="['Snapshot', 'Differential']"
            variant="outlined"
            class="pa-2"
            density="compact"
          />

          <v-text-field
            v-if="['PNG','SVG'].includes(exportAs)"
            v-model="attachmentFilename"
            label="Attachment filename"
            variant="outlined"
            class="pa-2"
            density="compact"
          />

          <v-select
            v-model="exportAs"
            label="Export as"
            :items="['PNG', 'SVG', 'Text file']"
            variant="outlined"
            class="pa-2"
            density="compact"
          />

          <v-checkbox
            v-model="hideRemovedObjects"
            label="Hide removed objects"
          />
          <v-checkbox
            v-model="showConstraints"
            label="Show constraints"
          />
          <v-checkbox
            v-model="showBindings"
            label="Show bindings"
          />
          <v-checkbox
            v-model="reduceSliceClasses"
            label="Reduce slice classes"
          />
        </v-col>

        <v-row>
          <v-col cols="6">
            <v-textarea
              v-model="leftText"
              :label="labels.first"
              variant="outlined"
              no-resize
              rows="40"
            />
          </v-col>

          <v-col cols="6">
            <v-textarea
              v-model="rightText"
              :label="labels.second"
              variant="outlined"
              no-resize
              rows="40"
              readonly
            />
          </v-col>

          <v-col cols="12">
            <v-textarea
              :model-value="requestPreview"
              label="API request"
              variant="outlined"
              no-resize
              rows="10"
              auto-grow
              readonly
            />
          </v-col>
        </v-row>
      </v-row>
    </v-responsive>
  </v-container>
</template>

<script setup>
import { ref, computed } from 'vue'

const baseURL = 'http://193.34.69.248:8080'
const mode = ref('FHIR -> UML')
const view = ref('Snapshot')
const exportAs = ref('PNG')
const hideRemovedObjects = ref(false)
const showConstraints = ref(false)
const showBindings = ref(false)
const reduceSliceClasses = ref(false)
const leftText = ref('')
const rightText = ref('')
// Filename defaults to "patient-def.png" but user can change it
const attachmentFilename = ref('patient-def.png')

const labels = computed(() => {
  if (!mode.value.includes('->')) {
    return { first: mode.value, second: '' }
  }
  const [left, right] = mode.value.split('->').map(str => str.trim())
  return { first: left, second: right }
})

// Dynamically build the textual preview of the POST request
const requestPreview = computed(() => {
  const endpoint = mode.value === 'FHIR -> UML' ? '/api/fhir2uml' : '/api/uml2fhir'
  const fullUrl = baseURL + endpoint
  const acceptHeader = `application/json; view=${view.value.toLowerCase()}`
  let contentType
  if (exportAs.value === 'PNG') {
    contentType = 'image/png'
  } else if (exportAs.value === 'SVG') {
    contentType = 'image/svg+xml'
  } else {
    contentType = 'text/plain'
  }

  // Conditionally add Content-Disposition if PNG or SVG
  const lines = [
    `POST ${fullUrl}`,
    `Accept: ${acceptHeader}`,
    `Content-Type: ${contentType}`
  ]

  if (['PNG','SVG'].includes(exportAs.value)) {
    lines.push(`Content-Disposition: attachment; filename="${attachmentFilename.value}"`)
  }

  // Include custom X- headers
  lines.push(
    `X-Hide-Removed-Objects: ${hideRemovedObjects.value ? 'true' : 'false'}`,
    `X-Show-Constraints: ${showConstraints.value ? 'true' : 'false'}`,
    `X-Show-Bindings: ${showBindings.value ? 'true' : 'false'}`,
    `X-Reduce-Slice-Classes: ${reduceSliceClasses.value ? 'true' : 'false'}`
  )

  return lines.join('\n')
})

function onGenerate() {
  const endpoint = mode.value === 'FHIR -> UML' ? '/api/fhir2uml' : '/api/uml2fhir'
  const url = baseURL + endpoint
  const acceptHeader = `application/json; view=${view.value.toLowerCase()}`

  let contentType
  if (exportAs.value === 'PNG') {
    contentType = 'image/png'
  } else if (exportAs.value === 'SVG') {
    contentType = 'image/svg+xml'
  } else {
    contentType = 'text/plain'
  }

  // Build headers object
  const headers = {
    Accept: acceptHeader,
    'Content-Type': contentType,
    'X-Hide-Removed-Objects': hideRemovedObjects.value ? 'true' : 'false',
    'X-Show-Constraints': showConstraints.value ? 'true' : 'false',
    'X-Show-Bindings': showBindings.value ? 'true' : 'false',
    'X-Reduce-Slice-Classes': reduceSliceClasses.value ? 'true' : 'false'
  }

  // Conditionally add Content-Disposition
  if (['PNG','SVG'].includes(exportAs.value)) {
    headers['Content-Disposition'] = `attachment; filename="${attachmentFilename.value}"`
  }

  fetch(url, {
    method: 'POST',
    headers,
    body: leftText.value
  })
    .then(async response => {
      // Check Content-Type in the response
      const contentType = response.headers.get('Content-Type') || ''
      if (contentType.includes('text')) {
        // If it's text, read as text
        const text = await response.text()
        // Put it in the second textarea
        rightText.value = text
      } else if (contentType.includes('image')) {
        // If it's an image, download it
        const blob = await response.blob()
        const downloadUrl = URL.createObjectURL(blob)

        // Create a hidden link element and simulate a click
        const link = document.createElement('a')
        link.href = downloadUrl
        link.download = attachmentFilename.value  // e.g. "patient-def.png"
        document.body.appendChild(link)
        link.click()
        link.remove()
        URL.revokeObjectURL(downloadUrl)

        // Clear the second textarea if you want
        rightText.value = 'Image downloaded'
      } else {
        // Handle other content types or unknown content
        const text = await response.text()
        rightText.value = `Unknown content-type:\n${contentType}\n\n${text}`
      }
    })
    .catch(err => {
      rightText.value = `Error: ${err}`
    })
}
</script>
