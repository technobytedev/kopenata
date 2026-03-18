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
          :center="[10.6762, 122.9513]"
          :use-global-leaflet="false"
          @click="onMapClick"
          style="height:100%;width:100%"
        >
          <LTileLayer
            url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
            attribution="&copy; OpenStreetMap contributors"
          />
          <!-- Attendee pins -->
          <template v-for="pin in activePins" :key="pin.id">
            <LMarker
              :lat-lng="[pin.lat, pin.lng]"
              :icon="makeEmojiIcon(pin.emoji, pin.going)"
            >
              <LTooltip>
                <div class="tooltip-inner">
                  <strong>{{ pin.display_name }}</strong>
                  <span>{{ pin.going ? '✅ Going' : '❌ Not going' }}</span>
                </div>
              </LTooltip>
            </LMarker>
          </template>
          <!-- Venue pin -->
            <LMarker
            v-if="activeSchedule?.lat && activeSchedule?.lng"
            :lat-lng="[activeSchedule.lat, activeSchedule.lng]"
            :icon="makeVenueIcon()"
            >
            <LTooltip :options="{ permanent: true, direction: 'top' }">
                <div class="tooltip-inner">
                <strong>{{ activeSchedule.venue_name || 'Meet here!' }}</strong>
                <span>{{ formatDate(activeSchedule.scheduled_at) }}</span>
                </div>
            </LTooltip>
            </LMarker>
        </LMap>
      </ClientOnly>

      <!-- Map hint -->
<div v-if="settingVenue" class="map-hint venue-hint">
  📍 Click anywhere on the map to set the coffee venue!
</div>
<div v-else-if="activeSchedule && user && !myPin" class="map-hint">
  ☕ Click the map to drop your pin!
</div>

      <!-- Active schedule badge -->
      <div v-if="activeSchedule" class="schedule-badge">
        <strong>{{ activeSchedule.title }}</strong>
        <span>{{ formatDate(activeSchedule.scheduled_at) }}</span>
        <span class="attendee-count">{{ activePins.filter(p=>p.going).length }} going</span>
      </div>
    </div>

    <!-- Schedules panel -->
    <Transition name="slide">
      <div v-if="showSchedules" class="panel">
        <div class="panel-header">
          <h2>☕ Meetup Schedules</h2>
          <button class="btn-ghost" @click="showSchedules = false">✕</button>
        </div>

        <!-- Create new -->
<div v-if="user" class="create-form">
  <input v-model="newTitle" placeholder="Meetup name..." class="input" />
  <input v-model="newDesc" placeholder="Notes (optional)" class="input" />
  <input v-model="newDate" type="datetime-local" class="input" />
  <input v-model="newVenueName" placeholder="Venue name (e.g. Bo's Coffee)" class="input" />

  <!-- Venue map picker -->
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

        <!-- Schedule list -->
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
            </div>
            <span class="schedule-count">
              {{ attendeeMap[s.id]?.filter(a=>a.going).length || 0 }} ☕
            </span>
          </div>
          <p v-if="!schedules.length" class="empty">No meetups yet. Create one!</p>
        </div>
      </div>
    </Transition>

    <!-- Pin drop modal -->
    <Transition name="fade">
      <div v-if="pendingPin" class="modal-bg" @click.self="pendingPin = null">
        <div class="modal">
          <h3>Drop your pin ✨</h3>
          <p class="modal-sub">for <em>{{ activeSchedule?.title }}</em></p>

          <div class="emoji-grid">
            <button
              v-for="e in EMOJIS"
              :key="e"
              class="emoji-btn"
              :class="{ selected: chosenEmoji === e }"
              @click="chosenEmoji = e"
            >{{ e}}</button>
          </div>

          <div class="going-toggle">
            <button :class="['toggle-btn', going ? 'active' : '']" @click="going = true">✅ Going</button>
            <button :class="['toggle-btn', !going ? 'active-no' : '']" @click="going = false">❌ Can't make it</button>
          </div>

          <div class="modal-actions">
            <button class="btn-outline" @click="pendingPin = null">Cancel</button>
            <button class="btn-primary" @click="dropPin">Drop Pin!</button>
          </div>
        </div>
      </div>
    </Transition>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { divIcon } from 'leaflet'

const supabase = useSupabaseClient()
const user = useSupabaseUser()

// ── State ──────────────────────────────────────────────
const schedules = ref([])
const attendeeMap = ref({})       // { scheduleId: [attendees] }
const activeSchedule = ref(null)
const showSchedules = ref(false)

const newVenueName = ref('')
const newVenueLat = ref(null)
const newVenueLng = ref(null)

const settingVenue = ref(false)

const newTitle = ref('')
const newDesc = ref('')
const newDate = ref('')

const pendingPin = ref(null)      // { lat, lng }
const chosenEmoji = ref('☕')
const going = ref(true)

const EMOJIS = ['😀','😎','🥳','😊','🤩','😋','🥸','😇','🤓','😏','🫡','🤠','😤','🧐','😍','🥰','😆','😜','🤪','🫶','💪','🙋','🧑','👦','👧','🧒','👩','👨','🧔','👴','👵']

// ── Computed ───────────────────────────────────────────
const activePins = computed(() =>
  activeSchedule.value ? (attendeeMap.value[activeSchedule.value.id] || []) : []
)

const myPin = computed(() =>
  user.value
    ? activePins.value.find(p => p.user_id === user.value.id)
    : null
)

// ── Helpers ────────────────────────────────────────────
function formatDate(dt) {
  return new Date(dt).toLocaleString('en-PH', {
    month: 'short', day: 'numeric',
    hour: '2-digit', minute: '2-digit'
  })
}

function makeEmojiIcon(emoji, isGoing) {
  const opacity = isGoing ? '1' : '0.45'
  return divIcon({
    html: `<div class="pin-emoji" style="opacity:${opacity};animation:walk 0.6s ease-in-out infinite alternate">${emoji}</div>`,
    className: '',
    iconSize: [40, 40],
    iconAnchor: [20, 40],
    tooltipAnchor: [0, -40],
  })
}

// ── Data loading ───────────────────────────────────────
async function loadSchedules() {
  const { data } = await supabase
    .from('schedules')
    .select('*')
    .order('scheduled_at', { ascending: true })
  schedules.value = data || []
  for (const s of schedules.value) {
    await loadAttendees(s.id)
  }
  // Auto-select nearest upcoming
  if (!activeSchedule.value && schedules.value.length) {
    const upcoming = schedules.value.find(s => new Date(s.scheduled_at) > new Date())
    activeSchedule.value = upcoming || schedules.value[0]
  }
}

async function loadAttendees(scheduleId) {
  const { data } = await supabase
    .from('attendees')
    .select('*')
    .eq('schedule_id', scheduleId)
  attendeeMap.value[scheduleId] = data || []
}

function makeVenueIcon() {
  return divIcon({
    html: `<div class="venue-pin">📍<div class="venue-label">${activeSchedule.value?.venue_name || 'Meet here!'}</div></div>`,
    className: '',
    iconSize: [60, 60],
    iconAnchor: [20, 50],
  })
}

// ── Actions ────────────────────────────────────────────
async function createSchedule() {
  if (!user.value || !newTitle.value || !newDate.value) return
  const { data, error } = await supabase.from('schedules').insert({
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
    activeSchedule.value = data  // auto-select new schedule
  }
}
async function selectSchedule(s) {
  activeSchedule.value = s
  showSchedules.value = false
  await loadAttendees(s.id)
}

function onMapClick(e) {
  if (!user.value) { navigateTo('/login'); return }

  // Venue-setting mode (when creating schedule)
  if (settingVenue.value) {
    newVenueLat.value = e.latlng.lat
    newVenueLng.value = e.latlng.lng
    settingVenue.value = false
    return
  }

  if (!activeSchedule.value) { showSchedules.value = true; return }
  pendingPin.value = { lat: e.latlng.lat, lng: e.latlng.lng }
}

async function dropPin() {
  if (!pendingPin.value || !activeSchedule.value || !user.value) return
  const name = user.value.user_metadata?.full_name || user.value.email
  const pin = {
    schedule_id: activeSchedule.value.id,
    user_id: user.value.id,   // ← this must match auth.uid()
    display_name: name,
    emoji: chosenEmoji.value,
    going: going.value,
    lat: pendingPin.value.lat,
    lng: pendingPin.value.lng,
  }
  const { error } = await supabase
    .from('attendees')
    .upsert(pin, { onConflict: 'schedule_id,user_id' })
  
  console.log('pin error:', error) // temp debug
  if (!error) {
    await loadAttendees(activeSchedule.value.id)
    pendingPin.value = null
  }
}

async function signOut() {
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
onUnmounted(() => { realtimeChannel?.unsubscribe() })
</script>

<style>
/* Walking emoji animation - global so leaflet can use it */
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
</style>

<style scoped>
.app { display: flex; flex-direction: column; height: 100vh; overflow: hidden; }

/* Header */
.topbar {
  display: flex; align-items: center; justify-content: space-between;
  padding: 0.6rem 1.25rem;
  background: var(--brown);
  color: var(--cream);
  z-index: 1000;
  border-bottom: 3px solid var(--coffee);
}
.logo { font-family: "Berkshire Swash", serif; font-size: 1.4rem; color: var(--cream); letter-spacing: 0.05em; }
.header-right { display: flex; align-items: center; gap: 0.75rem; }
.user-pill { display: flex; align-items: center; gap: 0.5rem; font-size: 0.85rem; }

/* Map */
.map-wrap { position: relative; flex: 1; }
.map-hint {
  position: absolute; bottom: 1.5rem; left: 50%; transform: translateX(-50%);
  background: var(--brown); color: var(--cream);
  padding: 0.5rem 1.25rem; font-size: 0.85rem;
  border: 2px solid var(--steam); z-index: 500;
  animation: pulse 2s ease-in-out infinite;
}
@keyframes pulse { 0%,100%{opacity:1} 50%{opacity:0.7} }

.schedule-badge {
  position: absolute; top: 1rem; left: 50%; transform: translateX(-50%);
  background: var(--card-bg); border: 2px solid var(--coffee);
  padding: 0.5rem 1rem; display: flex; gap: 1rem; align-items: center;
  z-index: 500; font-size: 0.85rem; box-shadow: 4px 4px 0 var(--coffee);
}
.attendee-count { background: var(--coffee); color: var(--cream); padding: 0.1rem 0.5rem; font-size: 0.75rem; }

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
.schedule-count { font-size: 1.2rem; }
.empty { text-align: center; color: var(--coffee); font-style: italic; padding: 2rem; }

/* Modal */
.modal-bg {
  position: fixed; inset: 0; background: rgba(61,31,10,0.5);
  display: grid; place-items: center; z-index: 3000;
}
.modal {
  background: var(--cream); border: 3px solid var(--brown);
  padding: 2rem; width: 340px; box-shadow: 8px 8px 0 var(--brown);
}
.modal h3 { font-family: "Berkshire Swash", serif; font-size: 1.5rem; margin-bottom: 0.25rem; }
.modal-sub { color: var(--coffee); font-style: italic; margin-bottom: 1.25rem; }
.emoji-grid { display: grid; grid-template-columns: repeat(6, 1fr); gap: 0.4rem; margin-bottom: 1.25rem; }
.emoji-btn {
  font-size: 1.5rem; padding: 0.25rem; border: 2px solid transparent;
  background: none; cursor: pointer; border-radius: 4px; transition: all 0.1s;
}
.emoji-btn:hover { border-color: var(--coffee); transform: scale(1.2); }
.emoji-btn.selected { border-color: var(--brown); background: var(--steam); }
.going-toggle { display: flex; gap: 0.5rem; margin-bottom: 1.25rem; }
.toggle-btn {
  flex: 1; padding: 0.5rem; border: 2px solid var(--coffee);
  background: white; cursor: pointer; font-family: "Courier Prime", monospace;
  font-size: 0.85rem; transition: all 0.1s;
}
.toggle-btn.active { background: var(--brown); color: var(--cream); border-color: var(--brown); }
.toggle-btn.active-no { background: var(--coffee); color: var(--cream); }
.modal-actions { display: flex; gap: 0.75rem; }

/* Buttons */
.btn-primary {
  padding: 0.5rem 1.25rem; background: var(--brown); color: var(--cream);
  border: 2px solid var(--brown); cursor: pointer;
  font-family: "Courier Prime", monospace; font-weight: 700;
  transition: all 0.15s; box-shadow: 3px 3px 0 var(--coffee);
}
.btn-primary:hover { transform: translate(-1px,-1px); box-shadow: 4px 4px 0 var(--coffee); }
.btn-primary:disabled { opacity: 0.5; cursor: not-allowed; }
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
.input {
  width: 100%; padding: 0.5rem 0.75rem; border: 2px solid var(--coffee);
  font-family: "Courier Prime", monospace; font-size: 0.9rem;
  background: white; color: var(--brown); outline: none;
}
.input:focus { border-color: var(--brown); }

/* Tooltip */
.tooltip-inner { display: flex; flex-direction: column; gap: 0.2rem; font-family: "Courier Prime", monospace; }

/* Transitions */
.slide-enter-active, .slide-leave-active { transition: transform 0.25s ease; }
.slide-enter-from, .slide-leave-to { transform: translateX(100%); }
.fade-enter-active, .fade-leave-active { transition: opacity 0.2s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }

@media (max-width: 500px) {
  .panel { width: 100%; }
}

.btn-venue {
  width: 100%;
  padding: 0.5rem 0.75rem;
  border: 2px dashed var(--coffee);
  background: white;
  cursor: pointer;
  font-family: "Courier Prime", monospace;
  font-size: 0.85rem;
  color: var(--coffee);
  transition: all 0.15s;
  text-align: center;
}
.btn-venue:hover, .btn-venue.active {
  background: var(--coffee);
  color: var(--cream);
  border-style: solid;
}

.venue-hint {
  background: var(--accent) !important;
  font-weight: 700;
  font-size: 0.95rem;
}
</style>