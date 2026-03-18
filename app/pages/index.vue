<template>
  <div class="app">
    <!-- Header -->
    <header class="topbar">
      <h1 class="logo">KOPENATA ☕</h1>
      <div class="header-right">
        <button v-if="!user" class="btn-outline" @click="navigateTo('/login')">Sign in</button>
        <div v-else class="user-pill">
          <span>{{ user.user_metadata?.name?.split(' ')[0] }}</span>
          <button class="btn-ghost" @click="signOut">out</button>
        </div>
        <button class="btn-primary" @click="showSchedules = true">📅 Meetups</button>
      </div>
    </header>

    <!-- Map -->
    <div class="map-wrap">
      <ClientOnly>
        <LMap
          ref="mapRef"
          :zoom="15"
          :center="mapCenter"
          :use-global-leaflet="false"
          style="height:100%;width:100%"
        >
          <LTileLayer
            url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
            attribution="&copy; OpenStreetMap contributors"
          />

          <!-- Moving user pins (going, not arrived) -->
          <template v-for="pin in movingPins" :key="pin.id">
            <LMarker
              :lat-lng="[pin.last_seen_lat ?? pin.lat, pin.last_seen_lng ?? pin.lng]"
              :icon="makeEmojiIcon(pin.emoji, true, false)"
            >
              <LTooltip>
                <div class="tooltip-inner">
                  <strong>{{ pin.display_name }}</strong>
                  <span>🚶 On the way...</span>
                </div>
              </LTooltip>
            </LMarker>
          </template>

          <!-- Not going pins -->
          <template v-for="pin in notGoingPins" :key="pin.id">
            <LMarker
              :lat-lng="[pin.last_seen_lat ?? pin.lat, pin.last_seen_lng ?? pin.lng]"
              :icon="makeEmojiIcon(pin.emoji, false, false)"
            >
              <LTooltip>
                <div class="tooltip-inner">
                  <strong>{{ pin.display_name }}</strong>
                  <span>❌ Not going</span>
                </div>
              </LTooltip>
            </LMarker>
          </template>

          <!-- Venue pin with arrived avatars attached -->
          <LMarker
            v-if="activeSchedule?.lat && activeSchedule?.lng"
            :lat-lng="[activeSchedule.lat, activeSchedule.lng]"
            :icon="makeVenueIcon()"
            @click="onVenueClick"
          >
          </LMarker>

        </LMap>
      </ClientOnly>

      <!-- Venue hint -->
      <div v-if="settingVenue" class="map-hint venue-hint" @click="showSchedules = true">
        📍 Click anywhere on the map to set the coffee venue!
      </div>

      <!-- No schedule selected -->
      <div v-else-if="!activeSchedule" class="map-hint">
        📅 Open Meetups to join a schedule!
      </div>
    </div>

    <!-- Schedules panel -->
    <Transition name="slide">
      <div v-if="showSchedules" class="panel">
        <div class="panel-header">
          <h2>☕ Meetup Schedules</h2>
          <button class="btn-ghost" @click="showSchedules = false">✕</button>
        </div>

        <div v-if="user" class="create-form">
          <input v-model="newTitle" placeholder="Meetup name..." class="input" />
          <input v-model="newDesc" placeholder="Notes (optional)" class="input" />
          <input v-model="newDate" type="datetime-local" class="input" />
          <input v-model="newVenueName" placeholder="Venue name (e.g. Bo's Coffee)" class="input" />
          <button
            class="btn-venue"
            :class="{ active: settingVenue }"
            @click="settingVenue = !settingVenue; showSchedules = false"
          >
            {{ newVenueLat ? '✅ Venue set! (click to change)' : '📍 Click map to set venue' }}
          </button>
          <button class="btn-primary full" @click="createSchedule" :disabled="!newTitle || !newDate">
            + Add Schedule
          </button>
        </div>

        <div class="schedule-list">
          <div
            v-for="s in schedules"
            :key="s.id"
            class="schedule-item"
            :class="{ active: activeSchedule?.id === s.id }"
            @click="selectSchedule(s)"
          >
            <div class="schedule-item-main">
              <strong>{{ s.title }}</strong>
              <span class="schedule-date">{{ formatDate(s.scheduled_at) }}</span>
              <span class="schedule-venue" v-if="s.venue_name">📍 {{ s.venue_name }}</span>
            </div>
            <span class="schedule-count">
              {{ attendeeMap[s.id]?.filter(a => a.going).length || 0 }} ☕
            </span>
          </div>
          <p v-if="!schedules.length" class="empty">No meetups yet. Create one!</p>
        </div>
      </div>
    </Transition>

    <!-- RSVP Modal (shown when clicking venue banner) -->
    <Transition name="fade">
      <div v-if="showRsvpModal" class="modal-bg" @click.self="showRsvpModal = false">
        <div class="modal">
          <!-- Schedule info -->
          <div class="modal-venue-info">
            <div class="modal-venue-emoji">☕</div>
            <div>
              <h3>{{ activeSchedule?.title }}</h3>
              <p class="modal-sub">{{ formatDate(activeSchedule?.scheduled_at) }}</p>
              <p class="modal-sub" v-if="activeSchedule?.venue_name">📍 {{ activeSchedule.venue_name }}</p>
            </div>
          </div>

          <!-- Arrived avatars -->
          <div v-if="arrivedPins.length" class="arrived-row">
            <span class="arrived-label">Already there:</span>
            <span v-for="pin in arrivedPins" :key="pin.id" class="arrived-emoji" :title="pin.display_name">
              {{ pin.emoji }}
            </span>
          </div>

          <!-- Going count -->
          <div class="rsvp-stats">
            <span>🚶 On the way: {{ movingPins.length }}</span>
            <span>🎉 Arrived: {{ arrivedPins.length }}</span>
          </div>

          <!-- If not RSVPed yet -->
          <div v-if="!myPin" class="emoji-section">
            <p class="emoji-label">Pick your avatar:</p>
            <div class="emoji-grid">
              <button
                v-for="e in EMOJIS"
                :key="e"
                class="emoji-btn"
                :class="{ selected: chosenEmoji === e }"
                @click="chosenEmoji = e"
              >{{ e }}</button>
            </div>
          </div>

          <!-- Action buttons -->
          <div v-if="!myPin" class="modal-actions">
            <button class="btn-notgoing" @click="rsvp(false)">❌ Can't make it</button>
            <button class="btn-going" @click="rsvp(true)">✅ I'm Going!</button>
          </div>

          <!-- Already RSVPed -->
          <div v-else class="already-rsvp">
            <div class="my-pin-display">{{ myPin.emoji }}</div>
            <p>You're <strong>{{ myPin.going ? 'going ✅' : 'not going ❌' }}</strong></p>
            <p v-if="myPin.going && !myPin.arrived" class="tracking-note">
              📡 Your location updates every 30s
            </p>
            <p v-if="myPin.arrived" class="arrived-note">
              🎉 You've arrived!
            </p>
            <button class="btn-outline small" @click="cancelRsvp">Change my mind</button>
          </div>
        </div>
      </div>
    </Transition>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'
import { LMap, LTileLayer, LMarker, LTooltip } from '@vue-leaflet/vue-leaflet'
import { divIcon } from 'leaflet'
import 'leaflet/dist/leaflet.css'

const supabase = useSupabaseClient()
const user = useSupabaseUser()

// ── State ──────────────────────────────────────────────
const schedules = ref([])
const attendeeMap = ref({})
const activeSchedule = ref(null)
const showSchedules = ref(false)
const showRsvpModal = ref(false)
const mapCenter = ref([10.6762, 122.9513])

const newTitle = ref('')
const newDesc = ref('')
const newDate = ref('')
const newVenueName = ref('')
const newVenueLat = ref(null)
const newVenueLng = ref(null)
const settingVenue = ref(false)

const chosenEmoji = ref('😀')
const locationWatcher = ref(null)
const locationInterval = ref(null)

const EMOJIS = ['😀','😎','🥳','😊','🤩','😋','🥸','😇','🤓','😏','🫡','🤠','😤','🧐','😍','🥰','😆','😜','🤪','🫶','💪','🙋','🧑','👦','👧','🧒','👩','👨','🧔','👴','👵']

// ── Computed ───────────────────────────────────────────
const activePins = computed(() =>
  activeSchedule.value ? (attendeeMap.value[activeSchedule.value.id] || []) : []
)
const myPin = computed(() =>
  user.value ? activePins.value.find(p => p.user_id === user.value.id) : null
)
const movingPins = computed(() =>
  activePins.value.filter(p => p.going && !p.arrived && (p.last_seen_lat || p.lat))
)
const notGoingPins = computed(() =>
  activePins.value.filter(p => !p.going && (p.last_seen_lat || p.lat))
)
const arrivedPins = computed(() =>
  activePins.value.filter(p => p.going && p.arrived)
)

// ── Helpers ────────────────────────────────────────────
function formatDate(dt) {
  if (!dt) return ''
  return new Date(dt).toLocaleString('en-PH', {
    month: 'short', day: 'numeric',
    hour: '2-digit', minute: '2-digit'
  })
}

function getDistance(lat1, lng1, lat2, lng2) {
  const R = 6371000
  const dLat = (lat2 - lat1) * Math.PI / 180
  const dLng = (lng2 - lng1) * Math.PI / 180
  const a = Math.sin(dLat/2)**2 +
    Math.cos(lat1 * Math.PI/180) * Math.cos(lat2 * Math.PI/180) * Math.sin(dLng/2)**2
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a))
}

function makeEmojiIcon(emoji, going, arrived) {
  if (arrived) return divIcon({ html: '', className: '', iconSize: [0,0] }) // hidden, shown on banner
  const opacity = going ? '1' : '0.4'
  const anim = going ? 'animation:walk 0.6s ease-in-out infinite alternate' : ''
  return divIcon({
    html: `<div class="pin-emoji" style="opacity:${opacity};${anim}">${emoji}</div>`,
    className: '',
    iconSize: [40, 40],
    iconAnchor: [20, 40],
    tooltipAnchor: [0, -40],
  })
}

function makeVenueIcon() {
  const arrived = arrivedPins.value
  const avatarRow = arrived.map(p =>
    `<span class="banner-avatar" title="${p.display_name}">${p.emoji}</span>`
  ).join('')

  return divIcon({
    html: `
      <div class="venue-banner" onclick="">
        <div class="venue-banner-top">
          <span class="venue-pin-icon">📍</span>
          <div class="venue-banner-info">
            <strong>${activeSchedule.value?.title || 'Meet here!'}</strong>
            <span>${activeSchedule.value?.venue_name || ''}</span>
          </div>
        </div>
        ${arrived.length ? `<div class="venue-avatars">${avatarRow}</div>` : ''}
        <div class="venue-tap-hint">tap to RSVP</div>
      </div>
    `,
    className: '',
    iconSize: [180, 'auto'],
    iconAnchor: [90, 80],
  })
}

// ── Map click for venue setting ────────────────────────
function onMapClick(e) {
  if (settingVenue.value) {
    newVenueLat.value = e.latlng.lat
    newVenueLng.value = e.latlng.lng
    settingVenue.value = false
    showSchedules.value = true
  }
}

function onVenueClick() {
  if (!user.value) { navigateTo('/login'); return }
  showRsvpModal.value = true
}

// ── Data ───────────────────────────────────────────────
async function loadSchedules() {
  const { data } = await supabase
    .from('schedules').select('*').order('scheduled_at', { ascending: true })
  schedules.value = data || []
  for (const s of schedules.value) await loadAttendees(s.id)
  if (!activeSchedule.value && schedules.value.length) {
    const upcoming = schedules.value.find(s => new Date(s.scheduled_at) > new Date())
    activeSchedule.value = upcoming || schedules.value[0]
    if (activeSchedule.value?.lat) mapCenter.value = [activeSchedule.value.lat, activeSchedule.value.lng]
  }
}

async function loadAttendees(scheduleId) {
  const { data } = await supabase.from('attendees').select('*').eq('schedule_id', scheduleId)
  attendeeMap.value[scheduleId] = data || []
}

async function selectSchedule(s) {
  activeSchedule.value = s
  showSchedules.value = false
  await loadAttendees(s.id)
  if (s.lat && s.lng) mapCenter.value = [s.lat, s.lng]
}

async function createSchedule() {
  if (!user.value || !newTitle.value || !newDate.value) return
  const { data } = await supabase.from('schedules').insert({
    title: newTitle.value,
    description: newDesc.value,
    scheduled_at: new Date(newDate.value).toISOString(),
    created_by: user.value.id,
    lat: newVenueLat.value,
    lng: newVenueLng.value,
    venue_name: newVenueName.value,
  }).select().single()
  if (data) {
    schedules.value.unshift(data)
    attendeeMap.value[data.id] = []
    newTitle.value = ''; newDesc.value = ''; newDate.value = ''
    newVenueName.value = ''; newVenueLat.value = null; newVenueLng.value = null
    showSchedules.value = false
    activeSchedule.value = data
    if (data.lat) mapCenter.value = [data.lat, data.lng]
  }
}

// ── RSVP ──────────────────────────────────────────────
async function rsvp(isGoing) {
  if (!user.value || !activeSchedule.value) return

  // Get current GPS position first
  let lat = activeSchedule.value.lat
  let lng = activeSchedule.value.lng

  if (isGoing && navigator.geolocation) {
    try {
      const pos = await new Promise((res, rej) =>
        navigator.geolocation.getCurrentPosition(res, rej, { timeout: 5000 })
      )
      lat = pos.coords.latitude
      lng = pos.coords.longitude
    } catch (e) { /* use venue as fallback */ }
  }

  const pin = {
    schedule_id: activeSchedule.value.id,
    user_id: user.value.id,
    display_name: user.value.user_metadata?.full_name || user.value.email,
    emoji: chosenEmoji.value,
    going: isGoing,
    lat,
    lng,
    last_seen_lat: isGoing ? lat : null,
    last_seen_lng: isGoing ? lng : null,
    arrived: false,
    updated_at: new Date().toISOString(),
  }

  await supabase.from('attendees').upsert(pin, { onConflict: 'schedule_id,user_id' })
  await loadAttendees(activeSchedule.value.id)

  if (isGoing) startLocationTracking()
}

async function cancelRsvp() {
  if (!user.value || !activeSchedule.value) return
  await supabase.from('attendees')
    .delete()
    .eq('schedule_id', activeSchedule.value.id)
    .eq('user_id', user.value.id)
  await loadAttendees(activeSchedule.value.id)
  stopLocationTracking()
}

// ── Location tracking ──────────────────────────────────
function startLocationTracking() {
  if (!navigator.geolocation) return
  stopLocationTracking()

  async function pushLocation() {
    if (!myPin.value?.going || myPin.value?.arrived) {
      stopLocationTracking(); return
    }
    navigator.geolocation.getCurrentPosition(async (pos) => {
      const lat = pos.coords.latitude
      const lng = pos.coords.longitude

      // Check if arrived (within 50m)
      const dist = getDistance(lat, lng, activeSchedule.value.lat, activeSchedule.value.lng)
      const arrived = dist <= 50

      await supabase.from('attendees').update({
        last_seen_lat: lat,
        last_seen_lng: lng,
        arrived,
        updated_at: new Date().toISOString(),
      })
      .eq('schedule_id', activeSchedule.value.id)
      .eq('user_id', user.value.id)

      await loadAttendees(activeSchedule.value.id)
    })
  }

  pushLocation() // immediate first push
  locationInterval.value = setInterval(pushLocation, 30000) // every 30s
}

function stopLocationTracking() {
  if (locationInterval.value) { clearInterval(locationInterval.value); locationInterval.value = null }
}

// Resume tracking if already going when page loads
watch(myPin, (pin) => {
  if (pin?.going && !pin?.arrived && !locationInterval.value) startLocationTracking()
}, { immediate: true })

async function signOut() {
  stopLocationTracking()
  await supabase.auth.signOut()
  navigateTo('/login')
}

// ── Realtime ───────────────────────────────────────────
let realtimeChannel
onMounted(async () => {
  await loadSchedules()
  realtimeChannel = supabase
    .channel('public:attendees')
    .on('postgres_changes', { event: '*', schema: 'public', table: 'attendees' }, async (payload) => {
      const sid = payload.new?.schedule_id || payload.old?.schedule_id
      if (sid) await loadAttendees(sid)
    })
    .on('postgres_changes', { event: 'INSERT', schema: 'public', table: 'schedules' }, async () => {
      await loadSchedules()
    })
    .subscribe()
})

onUnmounted(() => {
  realtimeChannel?.unsubscribe()
  stopLocationTracking()
})
</script>

<style>
@keyframes walk {
  from { transform: translateY(0) rotate(-5deg); }
  to   { transform: translateY(-6px) rotate(5deg); }
}
.pin-emoji {
  font-size: 2rem;
  display: block;
  filter: drop-shadow(0 2px 4px rgba(0,0,0,0.3));
  cursor: pointer;
}

/* Venue banner */
.venue-banner {
  background: var(--brown, #3d1f0a);
  color: var(--cream, #f5f0e8);
  border: 2px solid var(--steam, #c9a87c);
  padding: 0.5rem 0.75rem;
  cursor: pointer;
  box-shadow: 4px 4px 0 rgba(0,0,0,0.3);
  min-width: 160px;
  font-family: "Courier Prime", monospace;
  transition: transform 0.15s;
}
.venue-banner:hover { transform: scale(1.03); }
.venue-banner-top { display: flex; align-items: center; gap: 0.4rem; }
.venue-pin-icon { font-size: 1.2rem; }
.venue-banner-info { display: flex; flex-direction: column; }
.venue-banner-info strong { font-size: 0.85rem; line-height: 1.2; }
.venue-banner-info span { font-size: 0.7rem; opacity: 0.75; }
.venue-avatars { display: flex; flex-wrap: wrap; gap: 2px; margin-top: 0.35rem; padding-top: 0.35rem; border-top: 1px dashed var(--steam, #c9a87c); }
.banner-avatar { font-size: 1.2rem; animation: walk 0.6s ease-in-out infinite alternate; display: inline-block; }
.venue-tap-hint { font-size: 0.65rem; opacity: 0.6; text-align: center; margin-top: 0.3rem; letter-spacing: 0.05em; }
</style>

<style scoped>
.app { display: flex; flex-direction: column; height: 100vh; overflow: hidden; }

.topbar {
  display: flex; align-items: center; justify-content: space-between;
  padding: 0.6rem 1.25rem;
  background: var(--brown); color: var(--cream);
  z-index: 1000; border-bottom: 3px solid var(--coffee);
}
.logo { font-family: "Berkshire Swash", serif; font-size: 1.4rem; color: var(--cream); }
.header-right { display: flex; align-items: center; gap: 0.75rem; }
.user-pill { display: flex; align-items: center; gap: 0.5rem; font-size: 0.85rem; }

.map-wrap { position: relative; flex: 1; }

.map-hint {
  position: absolute; bottom: 1.5rem; left: 50%; transform: translateX(-50%);
  background: var(--brown); color: var(--cream);
  padding: 0.5rem 1.25rem; font-size: 0.85rem;
  border: 2px solid var(--steam); z-index: 500;
  animation: pulse 2s ease-in-out infinite; white-space: nowrap;
}
.venue-hint { background: var(--accent, #e8533a) !important; font-weight: 700; }
@keyframes pulse { 0%,100%{opacity:1} 50%{opacity:0.7} }

/* Panel */
.panel {
  position: fixed; top: 0; right: 0; bottom: 0; width: 360px;
  background: var(--cream); border-left: 3px solid var(--coffee);
  z-index: 2000; display: flex; flex-direction: column;
  box-shadow: -8px 0 32px rgba(61,31,10,0.15);
}
.panel-header {
  display: flex; align-items: center; justify-content: space-between;
  padding: 1rem 1.25rem; border-bottom: 2px solid var(--coffee);
  background: var(--brown); color: var(--cream);
}
.panel-header h2 { font-family: "Berkshire Swash", serif; font-size: 1.3rem; }
.create-form { padding: 1rem 1.25rem; border-bottom: 2px dashed var(--steam); display: flex; flex-direction: column; gap: 0.5rem; }
.schedule-list { flex: 1; overflow-y: auto; padding: 0.75rem; display: flex; flex-direction: column; gap: 0.5rem; }
.schedule-item {
  display: flex; align-items: center; justify-content: space-between;
  padding: 0.75rem 1rem; border: 2px solid var(--coffee);
  cursor: pointer; transition: all 0.15s; background: white;
}
.schedule-item:hover { background: var(--cream); transform: translateX(-3px); }
.schedule-item.active { background: var(--brown); color: var(--cream); border-color: var(--brown); }
.schedule-item-main { display: flex; flex-direction: column; gap: 0.2rem; }
.schedule-date { font-size: 0.78rem; opacity: 0.7; }
.schedule-venue { font-size: 0.75rem; opacity: 0.8; }
.schedule-count { font-size: 1.2rem; }
.empty { text-align: center; color: var(--coffee); font-style: italic; padding: 2rem; }

/* Modal */
.modal-bg {
  position: fixed; inset: 0; background: rgba(61,31,10,0.55);
  display: grid; place-items: center; z-index: 3000;
}
.modal {
  background: var(--cream); border: 3px solid var(--brown);
  padding: 1.75rem; width: 340px; box-shadow: 8px 8px 0 var(--brown);
  max-height: 90vh; overflow-y: auto;
}
.modal-venue-info { display: flex; align-items: center; gap: 1rem; margin-bottom: 1rem; }
.modal-venue-emoji { font-size: 2.5rem; }
.modal h3 { font-family: "Berkshire Swash", serif; font-size: 1.4rem; }
.modal-sub { color: var(--coffee); font-size: 0.85rem; margin-top: 0.2rem; }

.arrived-row { display: flex; align-items: center; gap: 0.5rem; flex-wrap: wrap; margin-bottom: 0.75rem; padding: 0.5rem; background: white; border: 1px dashed var(--coffee); }
.arrived-label { font-size: 0.75rem; color: var(--coffee); }
.arrived-emoji { font-size: 1.4rem; }

.rsvp-stats { display: flex; gap: 1rem; font-size: 0.82rem; color: var(--coffee); margin-bottom: 1rem; }

.emoji-label { font-size: 0.85rem; color: var(--coffee); margin-bottom: 0.5rem; }
.emoji-grid { display: grid; grid-template-columns: repeat(6, 1fr); gap: 0.4rem; margin-bottom: 1.25rem; }
.emoji-btn {
  font-size: 1.4rem; padding: 0.2rem; border: 2px solid transparent;
  background: none; cursor: pointer; border-radius: 4px; transition: all 0.1s;
}
.emoji-btn:hover { border-color: var(--coffee); transform: scale(1.2); }
.emoji-btn.selected { border-color: var(--brown); background: var(--steam); }

.modal-actions { display: flex; gap: 0.75rem; }
.btn-going {
  flex: 1; padding: 0.65rem; background: var(--brown); color: var(--cream);
  border: 2px solid var(--brown); cursor: pointer;
  font-family: "Courier Prime", monospace; font-weight: 700; font-size: 0.95rem;
  transition: all 0.15s; box-shadow: 3px 3px 0 var(--coffee);
}
.btn-going:hover { transform: translate(-1px,-1px); box-shadow: 4px 4px 0 var(--coffee); }
.btn-notgoing {
  flex: 1; padding: 0.65rem; background: white; color: var(--brown);
  border: 2px solid var(--coffee); cursor: pointer;
  font-family: "Courier Prime", monospace; font-size: 0.9rem; transition: all 0.15s;
}
.btn-notgoing:hover { background: var(--cream); }

.already-rsvp { text-align: center; padding: 0.5rem 0; }
.my-pin-display { font-size: 3rem; margin-bottom: 0.5rem; animation: walk 0.6s ease-in-out infinite alternate; display: inline-block; }
.tracking-note { font-size: 0.8rem; color: var(--coffee); margin-top: 0.5rem; font-style: italic; }
.arrived-note { font-size: 0.9rem; color: var(--brown); font-weight: 700; margin-top: 0.5rem; }
.btn-outline.small { margin-top: 1rem; padding: 0.35rem 1rem; font-size: 0.8rem; }

/* Shared buttons */
.btn-primary {
  padding: 0.5rem 1.25rem; background: var(--brown); color: var(--cream);
  border: 2px solid var(--brown); cursor: pointer;
  font-family: "Courier Prime", monospace; font-weight: 700;
  transition: all 0.15s; box-shadow: 3px 3px 0 var(--coffee);
}
.btn-primary:hover { transform: translate(-1px,-1px); box-shadow: 4px 4px 0 var(--coffee); }
.btn-primary:disabled { opacity: 0.5; cursor: not-allowed; transform: none; }
.btn-primary.full { width: 100%; }
.btn-outline {
  padding: 0.5rem 1.25rem; background: transparent; color: var(--brown);
  border: 2px solid var(--brown); cursor: pointer;
  font-family: "Courier Prime", monospace; font-weight: 700; transition: all 0.15s;
}
.btn-outline:hover { background: var(--brown); color: var(--cream); }
.btn-ghost {
  background: none; border: none; cursor: pointer; color: inherit;
  font-family: "Courier Prime", monospace; text-decoration: underline; font-size: 0.85rem;
}
.btn-venue {
  width: 100%; padding: 0.5rem 0.75rem; border: 2px dashed var(--coffee);
  background: white; cursor: pointer; font-family: "Courier Prime", monospace;
  font-size: 0.85rem; color: var(--coffee); transition: all 0.15s;
}
.btn-venue:hover, .btn-venue.active { background: var(--coffee); color: var(--cream); border-style: solid; }
.input {
  width: 100%; padding: 0.5rem 0.75rem; border: 2px solid var(--coffee);
  font-family: "Courier Prime", monospace; font-size: 0.9rem;
  background: white; color: var(--brown); outline: none;
}
.input:focus { border-color: var(--brown); }

.tooltip-inner { display: flex; flex-direction: column; gap: 0.2rem; font-family: "Courier Prime", monospace; }

.slide-enter-active, .slide-leave-active { transition: transform 0.25s ease; }
.slide-enter-from, .slide-leave-to { transform: translateX(100%); }
.fade-enter-active, .fade-leave-active { transition: opacity 0.2s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }

@media (max-width: 500px) { .panel { width: 100%; } }
</style>
