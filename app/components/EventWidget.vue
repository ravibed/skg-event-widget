<script setup>
// event widget - loads events and shows them as cards
// updated: added search filter + retry button

import { ref, computed, onMounted } from 'vue'

// props for embedding in other pages
// dataUrl - where the json comes from
// title, subtitle - can be customized if widget is reused
const props = defineProps({
  dataUrl: {
    type: String,
    default: '/events.json'
  },
  title: {
    type: String,
    default: 'Upcoming Events'
  },
  subtitle: {
    type: String,
    default: 'Entdecken Sie spannende Veranstaltungen in Ihrer Region'
  }
})

const events = ref([])
const isLoading = ref(true)
const errorMsg = ref('')
const searchText = ref('')

// full german month names
const monthNames = [
  'Januar', 'Februar', 'März', 'April', 'Mai', 'Juni',
  'Juli', 'August', 'September', 'Oktober', 'November', 'Dezember'
]

// short months for the date badge
const monthShort = [
  'Jan', 'Feb', 'Mär', 'Apr', 'Mai', 'Jun',
  'Jul', 'Aug', 'Sep', 'Okt', 'Nov', 'Dez'
]

// full weekday names in german
const weekdays = [
  'Sonntag', 'Montag', 'Dienstag', 'Mittwoch',
  'Donnerstag', 'Freitag', 'Samstag'
]

function formatDate(dateStr) {
  if (!dateStr) return ''
  const d = new Date(dateStr)
  const day = d.getDate()
  const month = monthNames[d.getMonth()]
  const year = d.getFullYear()
  const hh = String(d.getHours()).padStart(2, '0')
  const mm = String(d.getMinutes()).padStart(2, '0')
  return `${day}. ${month} ${year}, ${hh}:${mm} Uhr`
}

function getDay(dateStr) {
  if (!dateStr) return ''
  return new Date(dateStr).getDate()
}

function getMonthShort(dateStr) {
  if (!dateStr) return ''
  return monthShort[new Date(dateStr).getMonth()]
}

function getWeekday(dateStr) {
  if (!dateStr) return ''
  return weekdays[new Date(dateStr).getDay()]
}

function getTime(dateStr) {
  if (!dateStr) return ''
  const d = new Date(dateStr)
  const hh = String(d.getHours()).padStart(2, '0')
  const mm = String(d.getMinutes()).padStart(2, '0')
  return `${hh}:${mm}`
}

// filter by name or description - case insensitive
const filteredEvents = computed(() => {
  const q = searchText.value.trim().toLowerCase()
  if (!q) return events.value

  return events.value.filter(ev => {
    const name = (ev.name || '').toLowerCase()
    const desc = (ev.description || '').toLowerCase()
    return name.includes(q) || desc.includes(q)
  })
})

function clearSearch() {
  searchText.value = ''
}

async function loadEvents() {
  isLoading.value = true
  errorMsg.value = ''

  try {
    await new Promise(r => setTimeout(r, 800))

    const res = await fetch(props.dataUrl)
    if (!res.ok) {
      throw new Error('Request failed with status ' + res.status)
    }

    // check content type - had a weird bug where server returned html
    const ct = res.headers.get('content-type') || ''
    if (!ct.includes('application/json')) {
      throw new Error('Response is not JSON')
    }

    const json = await res.json()

    if (json && json.itemListElement) {
      events.value = json.itemListElement
    } else {
      events.value = []
    }
  } catch (e) {
    console.log('something went wrong loading events', e)
    errorMsg.value = 'Events could not be loaded. Please try again later.'
  }

  isLoading.value = false
}

onMounted(() => {
  loadEvents()
})
</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-slate-50 via-blue-50 to-indigo-50 py-12 px-4">
    <div class="max-w-7xl mx-auto">

      <!-- header -->
      <div class="mb-8 text-center">
        <div class="inline-block mb-3">
          <span class="text-xs font-bold tracking-widest text-blue-600 uppercase bg-blue-100 px-4 py-1.5 rounded-full">
            Veranstaltungen
          </span>
        </div>
        <h1 class="text-4xl md:text-5xl font-bold text-slate-900 mb-3">
          {{ title }}
        </h1>
        <p class="text-slate-600 text-lg max-w-2xl mx-auto">
          {{ subtitle }}
        </p>
      </div>

      <!-- search bar - only show when events are loaded -->
      <div v-if="!isLoading && !errorMsg" class="max-w-xl mx-auto mb-10">
        <div class="relative">
          <div class="absolute inset-y-0 left-0 pl-4 flex items-center pointer-events-none">
            <svg class="w-5 h-5 text-slate-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"/>
            </svg>
          </div>

          <input
            v-model="searchText"
            type="text"
            placeholder="Search events by name or description…"
            class="w-full pl-12 pr-12 py-3.5 bg-white border border-slate-200 rounded-full shadow-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent text-slate-700 placeholder-slate-400 transition-all"
          />

          <!-- clear button shows up when typing -->
          <button
            v-if="searchText"
            @click="clearSearch"
            class="absolute inset-y-0 right-0 pr-4 flex items-center text-slate-400 hover:text-slate-700"
          >
            <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/>
            </svg>
          </button>
        </div>

        <div v-if="searchText" class="mt-3 text-center text-sm text-slate-500">
          Found {{ filteredEvents.length }} of {{ events.length }} events
        </div>
      </div>

      <!-- loader -->
      <div v-if="isLoading" class="flex flex-col items-center justify-center py-20">
        <div class="w-16 h-16 border-4 border-blue-100 border-t-blue-600 rounded-full animate-spin"></div>
        <p class="mt-4 text-slate-500 font-medium">Loading events…</p>
      </div>

      <!-- error -->
      <div v-else-if="errorMsg" class="max-w-xl mx-auto">
        <div class="bg-red-50 border-l-4 border-red-500 rounded-lg p-5 shadow-sm">
          <div class="flex items-start gap-3">
            <div class="text-red-500 text-2xl">⚠️</div>
            <div class="flex-1">
              <h3 class="font-semibold text-red-900 mb-1">Something went wrong</h3>
              <p class="text-red-700 text-sm mb-4">{{ errorMsg }}</p>
              <button
                @click="loadEvents"
                class="bg-red-600 hover:bg-red-700 text-white text-sm font-semibold px-5 py-2 rounded-lg transition-colors"
              >
                Try again
              </button>
            </div>
          </div>
        </div>
      </div>

      <!-- no search results -->
      <div v-else-if="filteredEvents.length === 0 && searchText" class="text-center py-16">
        <div class="text-6xl mb-4">🔍</div>
        <h3 class="text-xl font-semibold text-slate-700 mb-2">No matching events</h3>
        <p class="text-slate-500 mb-5">Try a different keyword or clear the search.</p>
        <button
          @click="clearSearch"
          class="bg-blue-600 hover:bg-blue-700 text-white text-sm font-semibold px-5 py-2 rounded-lg transition-colors"
        >
          Clear search
        </button>
      </div>

      <!-- cards grid -->
      <div v-else class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        <div
          v-for="(ev, i) in filteredEvents"
          :key="i"
          class="group bg-white rounded-2xl overflow-hidden shadow-md hover:shadow-2xl transition-all duration-300 hover:-translate-y-1 border border-slate-100"
        >
          <!-- top strip -->
          <div class="h-2 bg-gradient-to-r from-blue-500 via-indigo-500 to-purple-500"></div>

          <div class="p-6">
            <!-- date + time -->
            <div class="flex items-start gap-3 mb-4">
              <!-- calendar badge -->
              <div class="bg-gradient-to-br from-blue-500 to-indigo-600 rounded-xl p-3 text-white text-center min-w-[62px] shadow-lg">
                <div class="text-[10px] font-semibold uppercase tracking-wide opacity-90">
                  {{ getMonthShort(ev.startDate) }}
                </div>
                <div class="text-2xl font-bold leading-tight">
                  {{ getDay(ev.startDate) }}
                </div>
              </div>

              <div class="flex-1 min-w-0">
                <div class="text-sm text-slate-700 font-semibold">
                  {{ getWeekday(ev.startDate) }}
                </div>
                <div class="flex items-center gap-1 text-slate-500 text-sm mt-0.5">
                  <svg class="w-4 h-4 text-blue-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z"/>
                  </svg>
                  {{ getTime(ev.startDate) }} Uhr
                </div>
              </div>
            </div>

            <!-- title -->
            <h3 class="text-xl font-bold text-slate-900 mb-2 group-hover:text-blue-600 transition-colors leading-snug">
              {{ ev.name }}
            </h3>

            <!-- description -->
            <p class="text-slate-600 text-sm leading-relaxed mb-5 line-clamp-3">
              {{ ev.description }}
            </p>

            <!-- location -->
            <div class="pt-4 border-t border-slate-100">
              <div class="flex items-start gap-3">
                <div class="bg-blue-50 p-2 rounded-lg flex-shrink-0">
                  <svg class="w-5 h-5 text-blue-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17.657 16.657L13.414 20.9a1.998 1.998 0 01-2.827 0l-4.244-4.243a8 8 0 1111.314 0z"/>
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 11a3 3 0 11-6 0 3 3 0 016 0z"/>
                  </svg>
                </div>
                <div class="flex-1 min-w-0">
                  <div class="font-semibold text-slate-900 text-sm">
                    {{ ev.location?.name }}
                  </div>
                  <div class="text-xs text-slate-500 mt-0.5">
                    {{ ev.location?.address }}
                  </div>
                </div>
              </div>
            </div>

            <!-- full formatted date -->
            <div class="mt-4 text-xs text-slate-400 font-medium">
              {{ formatDate(ev.startDate) }}
            </div>
          </div>
        </div>
      </div>

      <!-- empty -->
      <div v-if="!isLoading && !errorMsg && events.length === 0" class="text-center py-20">
        <div class="text-6xl mb-4">📭</div>
        <p class="text-slate-500 text-lg">No events found.</p>
      </div>

    </div>
  </div>
</template>

<style scoped>
.line-clamp-3 {
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
</style>