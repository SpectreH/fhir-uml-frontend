<template>
  <v-container class="fill-height">
    <v-responsive class="align-center fill-height mx-auto" style="padding: 5px;">
      <v-row v-if="errorMessage">
        <v-col cols="12">
          <div class="error-box">
            {{ errorMessage }}
          </div>
        </v-col>
      </v-row>

      <v-row>
        <v-col cols="2">
          <v-btn
            variant="outlined"
            class="w-100 pa-1"
            color="blue"
            @click="onGenerate"
            :disabled="isLoading"
          >
            <!-- If you want an inline loader, you could do: -->
            <template #prepend v-if="isLoading">
              <v-progress-circular
                size="20"
                indeterminate
                class="me-2"
              />
            </template>
            {{ isLoading ? 'Please wait...' : 'Generate' }}
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

          <v-select
            v-model="exportAs"
            label="Export as"
            :items="['PNG', 'SVG', 'Text file']"
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

        <!-- Column 2 (5 cols) for the "left" textarea -->
        <v-col cols="5">
          <!-- Left textarea (unchanged) -->
          <v-textarea
            v-model="leftText"
            :label="labels.first"
            variant="outlined"
            no-resize
            rows="40"
          />
        </v-col>

        <v-col cols="5">
          <!-- If it's an image response, show the image wrapper -->
          <template v-if="isImageResponse">
            <div class="image-wrapper" @click="openImageInNewTab">
              <img
                :src="imageUrl"
                alt="Server response image"
              />
              <div class="hover-label">
                Show full size
              </div>
            </div>
          </template>

          <!-- Otherwise, show rightText in a textarea -->
          <template v-else>
            <v-textarea
              v-model="rightText"
              :label="labels.second"
              variant="outlined"
              no-resize
              rows="40"
              readonly
            />
          </template>
        </v-col>
      </v-row>

      <v-row>
        <v-col cols="12">
          <!-- Request preview (unchanged) -->
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
const attachmentFilename = ref('patient-def.png')

const isImageResponse = ref(false)
const imageUrl = ref(null)

const labels = computed(() => {
  if (!mode.value.includes('->')) {
    return { first: mode.value, second: '' }
  }
  const [left, right] = mode.value.split('->').map(str => str.trim())
  return { first: left, second: right }
})

function openImageInNewTab() {
  if (imageUrl.value) {
    window.open(imageUrl.value, '_blank')
  }
}

// Loading and errors
const isLoading = ref(false)
const errorMessage = ref('')

// Clear error after 5s
function showErrorTemporary(msg) {
  errorMessage.value = msg
  setTimeout(() => {
    errorMessage.value = ''
  }, 5000)
}

// Request preview logic
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

  const lines = [
    `POST ${fullUrl}`,
    `Accept: ${acceptHeader}`,
    `Content-Type: ${contentType}`
  ]

  if (['PNG','SVG'].includes(exportAs.value)) {
    lines.push(`Content-Disposition: attachment; filename="${attachmentFilename.value}"`)
  }

  lines.push(
    `X-Hide-Removed-Objects: ${hideRemovedObjects.value ? 'true' : 'false'}`,
    `X-Show-Constraints: ${showConstraints.value ? 'true' : 'false'}`,
    `X-Show-Bindings: ${showBindings.value ? 'true' : 'false'}`,
    `X-Reduce-Slice-Classes: ${reduceSliceClasses.value ? 'true' : 'false'}`
  )

  return lines.join('\n')
})

async function onGenerate() {
  isLoading.value = true
  errorMessage.value = ''

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

  const headers = {
    Accept: acceptHeader,
    'Content-Type': contentType,
    'X-Hide-Removed-Objects': hideRemovedObjects.value ? 'true' : 'false',
    'X-Show-Constraints': showConstraints.value ? 'true' : 'false',
    'X-Show-Bindings': showBindings.value ? 'true' : 'false',
    'X-Reduce-Slice-Classes': reduceSliceClasses.value ? 'true' : 'false'
  }

  if (['PNG','SVG'].includes(exportAs.value)) {
    headers['Content-Disposition'] = `attachment; filename="${attachmentFilename.value}"`
  }

  // 20s timeout
  let didTimeout = false
  const timeoutId = setTimeout(() => {
    didTimeout = true
    isLoading.value = false
    showErrorTemporary('Request timed out after 20 seconds.')
  }, 20000)

  try {
    const response = await fetch(url, {
      method: 'POST',
      headers,
      body: leftText.value
    })

    // If the timeout already triggered, skip parsing
    if (didTimeout) return

    clearTimeout(timeoutId)

    if (!response.ok) {
      // e.g., 4xx or 5xx
      const errorText = await response.text()
      throw new Error(`HTTP ${response.status}: ${errorText}`)
    }

    const respContentType = response.headers.get('Content-Type') || ''

    if (respContentType.includes('text')) {
      const text = await response.text()
      rightText.value = text
      isImageResponse.value = false
      imageUrl.value = null
    } else if (respContentType.includes('image')) {
      const blob = await response.blob()
      imageUrl.value = URL.createObjectURL(blob)
      isImageResponse.value = true
      rightText.value = ''
    } else {
      const text = await response.text()
      rightText.value = `Unknown content-type:\n${respContentType}\n\n${text}`
      isImageResponse.value = false
      imageUrl.value = null
    }
  } catch (error) {
    if (!didTimeout) {
      clearTimeout(timeoutId)
      showErrorTemporary(`Error: ${error}`)
    }
  } finally {
    isLoading.value = false
  }
}
</script>

<style scoped>
/* Example of how you might style the error box */
.error-box {
  background-color: #fdd;
  color: #900;
  padding: 8px;
  border: 1px solid #f99;
  margin-top: 8px;
  border-radius: 4px;
}

/* Existing image-wrapper styles, etc. remain the same */

.image-wrapper {
  position: relative;
  display: inline-block;
  max-width: 100%;
  cursor: pointer;
}

.image-wrapper img {
  display: block;
  max-width: 100%;
  max-height: 700px;
  object-fit: contain;
  border: 1px solid #ccc;
  padding: 4px;
  transition: box-shadow 0.2s ease;
}

.image-wrapper:hover img {
  box-shadow: 0 0 8px rgba(0, 0, 0, 0.3);
}

.hover-label {
  display: none;
  position: absolute;
  bottom: 8px;
  left: 8px;
  background-color: rgba(0,0,0,0.6);
  color: #fff;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 0.8em;
}

.image-wrapper:hover .hover-label {
  display: block;
}
</style>
