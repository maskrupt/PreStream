<template>
  <!-- OBS / embed-only mode -->
  <div
    v-if="embedMode"
    class="w-screen h-screen bg-black relative overflow-hidden"
  >
    <iframe
      v-if="iframeSrc"
      class="w-full h-full border-0"
      :src="iframeSrc"
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
      allowfullscreen
    ></iframe>

    <!-- Kiosk green countdown -->
    <div
      v-if="totalSeconds > 0"
      class="pointer-events-none absolute inset-0 flex items-center justify-center z-10"
    >
      <div
        class="font-black uppercase tracking-[0.25em] text-5xl sm:text-6xl md:text-7xl text-emerald-400"
        :style="timerStyle"
      >
        {{ formattedRemaining }}
      </div>
    </div>
  </div>

  <!-- Playlist generator UI mode -->
  <div
    v-else
    class="min-h-screen flex items-center justify-center bg-slate-950 text-slate-50 px-4 py-10"
  >
    <div
      class="w-full max-w-6xl rounded-2xl border border-slate-800/80 bg-gradient-to-br from-slate-900 to-slate-950 shadow-2xl shadow-black/70 p-6 sm:p-8"
    >
      <!-- Header -->
      <div class="flex flex-col sm:flex-row sm:items-center sm:justify-between gap-4">
        <div>
          <h1 class="text-xl sm:text-2xl font-semibold tracking-tight">
            PreStream - Your Pre-Live Power Tool
          </h1>
          <p class="text-xs sm:text-sm text-slate-400 mt-1">
            Build a playlist, auto-fetch YouTube durations and generate an OBS Browser Source URL
            with green countdown overlay.
          </p>
        </div>
        <div
          class="inline-flex items-center gap-2 rounded-full border border-emerald-400/70 bg-emerald-500/10 px-3 py-1 text-xs font-medium text-emerald-300"
        >
          <span class="inline-block h-1.5 w-1.5 rounded-full bg-emerald-300 animate-pulse" />
          <span>Made by Maskrupt</span>
        </div>
      </div>

      <!-- Main grid -->
      <div class="mt-6 grid gap-6 md:grid-cols-[1.8fr,1.2fr]">
        <!-- Left: controls -->
        <div class="space-y-4">
          <!-- Single video -->
          <div>
            <label class="block text-xs font-medium text-slate-300 mb-1">
              Single video (optional quick mode)
            </label>
            <input
              v-model="singleInput"
              type="text"
              class="w-full rounded-xl border border-slate-700/80 bg-slate-900/80 px-3 py-2 text-sm outline-none focus:border-emerald-500 focus:ring-1 focus:ring-emerald-500/60"
              placeholder="https://youtu.be/dQw4w9WgXcQ or dQw4w9WgXcQ"
            />
            <div class="mt-1.5 flex flex-wrap items-center gap-2 text-[11px] text-slate-500">
              <span>Accepts:</span>
              <code class="rounded-full bg-slate-900/80 px-2 py-0.5">watch?v=</code>
              <code class="rounded-full bg-slate-900/80 px-2 py-0.5">youtu.be/</code>
              <code class="rounded-full bg-slate-900/80 px-2 py-0.5">shorts/</code>
              <code class="rounded-full bg-slate-900/80 px-2 py-0.5">pure ID</code>
              <span>• Used if playlist is empty.</span>
            </div>
          </div>

          <!-- Playlist table -->
          <div>
            <label class="block text-xs font-medium text-slate-300 mb-1">
              Playlist
            </label>
            <p class="text-[11px] text-slate-500 mb-1">
              Paste YouTube URLs or IDs. Duration auto-fills when you leave the field. You can tweak
              durations manually afterwards.
            </p>

            <div class="overflow-hidden rounded-xl border border-slate-800/90 bg-slate-950/70">
              <table class="w-full border-collapse text-xs">
                <thead class="bg-slate-950/90">
                  <tr class="border-b border-slate-800/90 text-slate-400">
                    <th class="px-3 py-2 text-left w-6">#</th>
                    <th class="px-3 py-2 text-left">URL or Video ID</th>
                    <th class="px-3 py-2 text-left w-24">Duration</th>
                    <th class="px-3 py-2 text-center w-8"></th>
                  </tr>
                </thead>
                <tbody>
                  <tr
                    v-for="(item, idx) in playlist"
                    :key="idx"
                    class="border-b border-slate-900/70 last:border-b-0 odd:bg-slate-950/40"
                  >
                    <td class="px-3 py-1.5 align-top text-slate-500">
                      {{ idx + 1 }}
                    </td>
                    <td class="px-3 py-1.5 align-top">
                      <input
                        v-model="item.raw"
                        type="text"
                        class="w-full rounded-lg border border-slate-700/80 bg-slate-950/80 px-2 py-1 text-[11px] outline-none focus:border-emerald-500 focus:ring-1 focus:ring-emerald-500/60"
                        placeholder="YouTube URL or ID"
                        @change="onVideoFieldBlur(item, idx)"
                      />
                    </td>
                    <td class="px-3 py-1.5 align-top">
                      <input
                        v-model="item.duration"
                        type="text"
                        class="w-full rounded-lg border border-slate-700/80 bg-slate-950/80 px-2 py-1 text-[11px] outline-none focus:border-emerald-500 focus:ring-1 focus:ring-emerald-500/60"
                        placeholder="mm:ss"
                        @input="recalcSummary"
                      />
                    </td>
                    <td class="px-3 py-1.5 align-top text-center">
                      <button
                        class="text-xs text-red-400 hover:text-red-300"
                        type="button"
                        @click="removeRow(idx)"
                      >
                        ✕
                      </button>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>

            <div class="mt-2 flex items-center justify-between text-[11px] text-slate-400">
              <button
                type="button"
                class="inline-flex items-center gap-1 rounded-full border border-slate-700/80 bg-slate-900/80 px-3 py-1 text-[11px] hover:border-emerald-500 hover:text-emerald-300 transition"
                @click="addRow"
              >
                ➕ Add video
              </button>
              <div class="flex flex-wrap gap-2">
                <span>{{ playlist.length }} video(s)</span>
                <span>•</span>
                <span>With duration: {{ videosWithDuration }}</span>
                <span>•</span>
                <span>Total: {{ formattedTotal }}</span>
              </div>
            </div>
          </div>

          <!-- Mode & actions -->
          <div class="flex flex-col gap-3">
            <div>
              <label class="block text-xs font-medium text-slate-300 mb-1">
                Query mode
              </label>
              <select
                v-model="mode"
                class="w-full rounded-xl border border-slate-700/80 bg-slate-900/80 px-3 py-2 text-xs outline-none focus:border-emerald-500 focus:ring-1 focus:ring-emerald-500/60"
              >
                <option value="videoId">Use videoId + playlist IDs</option>
                <option value="videoUrl">Use videoUrl (first video only)</option>
              </select>
            </div>

            <div class="flex flex-wrap gap-2">
              <button
                type="button"
                class="inline-flex items-center gap-2 rounded-full bg-emerald-500 px-4 py-2 text-xs font-semibold text-slate-950 shadow-lg shadow-emerald-500/40 hover:bg-emerald-400 transition"
                @click="generateUrl"
              >
                ⚙️ Generate OBS URL
              </button>
              <button
                type="button"
                class="inline-flex items-center gap-2 rounded-full border border-slate-700/80 bg-slate-900/80 px-4 py-2 text-xs hover:border-emerald-500 hover:text-emerald-300 transition"
                @click="copyUrl"
              >
                📋 Copy OBS URL
              </button>
            </div>

            <p
              class="text-[11px] min-h-[1.2rem]"
              :class="status.type === 'err' ? 'text-rose-400' : 'text-emerald-400'"
            >
              {{ status.message }}
            </p>
          </div>

          <!-- Outputs -->
          <div class="grid gap-3 sm:grid-cols-2">
            <div class="rounded-xl border border-slate-800/80 bg-slate-950/80 p-3">
              <div
                class="text-[10px] font-semibold uppercase tracking-[0.16em] text-slate-500 mb-1"
              >
                Total playlist duration
              </div>
              <div class="font-mono text-lg">{{ formattedTotal }}</div>
            </div>

            <div class="rounded-xl border border-slate-800/80 bg-slate-950/80 p-3">
              <div
                class="text-[10px] font-semibold uppercase tracking-[0.16em] text-slate-500 mb-1"
              >
                OBS Browser Source URL
              </div>
              <div class="font-mono text-[11px] break-all text-slate-300">
                {{ obsUrl || "—" }}
              </div>
            </div>
          </div>
        </div>

        <!-- Right: preview -->
        <div class="space-y-2">
          <p class="text-[11px] uppercase tracking-[0.16em] text-slate-500">
            Preview (first video)
          </p>
          <div
            class="aspect-video w-full overflow-hidden rounded-2xl border border-slate-800/90 bg-black/80"
          >
            <iframe
              v-if="previewVideoId"
              class="h-full w-full border-0"
              :src="previewSrc"
              allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
              allowfullscreen
            ></iframe>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, onMounted, reactive, ref } from "vue";

function extractVideoId(input) {
  const value = (input || "").trim();
  if (/^[a-zA-Z0-9_-]{6,15}$/.test(value) && !value.startsWith("http")) {
    return value;
  }
  try {
    const url = new URL(value);
    if (url.hostname.includes("youtu.be")) {
      return url.pathname.replace("/", "");
    }
    if (url.searchParams.get("v")) {
      return url.searchParams.get("v");
    }
    if (url.pathname.includes("/shorts/")) {
      const parts = url.pathname.split("/");
      const idx = parts.indexOf("shorts");
      if (idx !== -1 && parts[idx + 1]) {
        return parts[idx + 1];
      }
    }
  } catch {}
  return "";
}

function parseDurationToSeconds(value) {
  const v = (value || "").trim();
  if (!v) return 0;
  if (/^\d+$/.test(v)) {
    return parseInt(v, 10);
  }
  const parts = v.split(":").map((p) => parseInt(p, 10));
  if (parts.some((n) => Number.isNaN(n))) return 0;
  if (parts.length === 2) {
    const [m, s] = parts;
    return m * 60 + s;
  }
  if (parts.length === 3) {
    const [h, m, s] = parts;
    return h * 3600 + m * 60 + s;
  }
  return 0;
}

function formatSeconds(totalSeconds) {
  const s = Math.max(0, Math.floor(totalSeconds));
  const hours = Math.floor(s / 3600);
  const minutes = Math.floor((s % 3600) / 60);
  const seconds = s % 60;
  if (hours > 0) {
    return (
      String(hours).padStart(2, "0") +
      ":" +
      String(minutes).padStart(2, "0") +
      ":" +
      String(seconds).padStart(2, "0")
    );
  }
  return String(minutes).padStart(2, "0") + ":" + String(seconds).padStart(2, "0");
}

let ytApiReadyPromise = null;
function ensureYoutubeApi() {
  if (window.YT && window.YT.Player) return Promise.resolve();
  if (ytApiReadyPromise) return ytApiReadyPromise;
  ytApiReadyPromise = new Promise((resolve) => {
    const prev = window.onYouTubeIframeAPIReady;
    window.onYouTubeIframeAPIReady = function () {
      if (typeof prev === "function") prev();
      resolve();
    };
  });
  return ytApiReadyPromise;
}

function getYoutubeDuration(videoId) {
  return ensureYoutubeApi().then(
    () =>
      new Promise((resolve, reject) => {
        let player;
        let timeout;
        let container;

        function cleanup() {
          clearTimeout(timeout);
          if (player && player.destroy) player.destroy();
          if (container && container.parentNode) {
            container.parentNode.removeChild(container);
          }
        }

        timeout = setTimeout(() => {
          cleanup();
          reject("Timeout getting duration");
        }, 15000);

        container = document.createElement("div");
        Object.assign(container.style, {
          position: "fixed",
          width: "0",
          height: "0",
          opacity: "0",
          pointerEvents: "none",
          zIndex: "-1",
        });
        document.body.appendChild(container);

        player = new YT.Player(container, {
          height: "0",
          width: "0",
          videoId,
          events: {
            onReady: () => {
              try {
                const duration = player.getDuration();
                cleanup();
                resolve(duration);
              } catch (err) {
                cleanup();
                reject(err);
              }
            },
            onError: (e) => {
              console.error("YT onError while probing duration", e);
              cleanup();
              reject("Failed to load YouTube video");
            },
          },
        });
      })
  );
}

const embedMode = ref(false);
const iframeSrc = ref("");
const totalSeconds = ref(0);
const remaining = ref(0);
let countdownHandle = null;

const formattedRemaining = computed(() => formatSeconds(remaining.value));
const timerStyle = {
  textShadow:
    "0 0 10px rgba(22, 163, 74, 1), 0 0 30px rgba(21, 128, 61, 1), 0 0 80px rgba(21, 128, 61, 1)",
  transform: "skewX(-6deg)",
};

function startCountdown(seconds) {
  if (!seconds || seconds <= 0) return;
  totalSeconds.value = seconds;
  remaining.value = seconds;
  if (countdownHandle) clearInterval(countdownHandle);
  countdownHandle = setInterval(() => {
    remaining.value = Math.max(0, remaining.value - 1);
    if (remaining.value <= 0 && countdownHandle) {
      clearInterval(countdownHandle);
      countdownHandle = null;
    }
  }, 1000);
}

function initEmbedModeFromUrl() {
  const params = new URLSearchParams(window.location.search);
  const qsVideoId = params.get("videoId");
  const qsVideoUrl = params.get("videoUrl");
  const qsPlaylist = params.get("playlist");
  const mutedParam = params.get("muted");
  const secondsParam = params.get("seconds");
  const hasEmbedParams = qsVideoId || qsVideoUrl || qsPlaylist;
  if (!hasEmbedParams) return false;

  let playlistIds = [];
  if (qsPlaylist) {
    playlistIds = qsPlaylist
      .split(",")
      .map((x) => x.trim())
      .filter((x) => !!x);
  }
  let mainIdCandidate = qsVideoId || extractVideoId(qsVideoUrl);
  let mainVideoId = extractVideoId(mainIdCandidate);
  if (!mainVideoId && playlistIds.length > 0) {
    mainVideoId = extractVideoId(playlistIds[0]);
  }
  if (!mainVideoId) return false;

  const finalPlaylistParam =
    playlistIds.length > 0 ? playlistIds.join(",") : mainVideoId;
  const isMuted =
    mutedParam === "1" || mutedParam === "true" || mutedParam === "yes";

  iframeSrc.value = `https://www.youtube-nocookie.com/embed/${encodeURIComponent(
    mainVideoId
  )}?autoplay=1&controls=0&rel=0&mute=${isMuted ? 1 : 0}&playsinline=1&loop=1&playlist=${encodeURIComponent(
    finalPlaylistParam
  )}`;

  const secondsTotal = parseInt(secondsParam || "0", 10) || 0;
  if (secondsTotal > 0) {
    startCountdown(secondsTotal);
  }

  embedMode.value = true;
  return true;
}

const singleInput = ref("");
const playlist = reactive([{ raw: "", duration: "" }]);
const mode = ref("videoId");
const obsUrl = ref("");
const status = reactive({ message: "", type: "ok" });

const totalSecondsPlaylist = computed(() =>
  playlist.reduce((acc, item) => acc + parseDurationToSeconds(item.duration), 0)
);
const formattedTotal = computed(() => formatSeconds(totalSecondsPlaylist.value));
const videosWithDuration = computed(
  () => playlist.filter((p) => parseDurationToSeconds(p.duration) > 0).length
);

const previewVideoId = computed(() => {
  const first = playlist.find((p) => extractVideoId(p.raw));
  return first ? extractVideoId(first.raw) : extractVideoId(singleInput.value);
});
const previewSrc = computed(() => {
  if (!previewVideoId.value) return "";
  return `https://www.youtube.com/embed/${encodeURIComponent(
    previewVideoId.value
  )}?autoplay=0&controls=1&rel=0`;
});

function recalcSummary() {
  void totalSecondsPlaylist.value;
}

async function onVideoFieldBlur(item, idx) {
  const id = extractVideoId(item.raw);
  if (!id) return;
  try {
    item.duration = "…";
    const sec = await getYoutubeDuration(id);
    item.duration = formatSeconds(sec);
    recalcSummary();
  } catch (err) {
    console.error("auto duration failed", err);
    if (item.duration === "…") {
      item.duration = "";
    }
  }
}

function addRow() {
  playlist.push({ raw: "", duration: "" });
}
function removeRow(idx) {
  playlist.splice(idx, 1);
}

function buildFullViteUrl(queryString) {
  const { origin, pathname } = window.location;
  const basePath = pathname.replace(/index\.html$/, "");
  const trimmedBase = basePath.endsWith("/") ? basePath.slice(0, -1) : basePath;
  const finalBase = trimmedBase || "/";
  return `${origin}${finalBase}?${queryString}`;
}

function setStatus(message, type = "ok") {
  status.message = message;
  status.type = type;
}

function generateUrl() {
  let ids = playlist
    .map((p) => extractVideoId(p.raw))
    .filter((id) => !!id);

  let usingPlaylist = ids.length > 0;
  let totalSec = 0;

  if (usingPlaylist) {
    totalSec = totalSecondsPlaylist.value;
  } else {
    const singleRaw = singleInput.value || "";
    const singleId = extractVideoId(singleRaw);
    if (!singleRaw.trim() || !singleId) {
      setStatus("Add at least one playlist item or a valid single video.", "err");
      obsUrl.value = "";
      return;
    }
    ids = [singleId];
    totalSec = 300;
  }

  if (totalSec <= 0) {
    setStatus(
      "Playlist durations are 0. Let durations auto-fill or set them manually.",
      "err"
    );
    return;
  }

  const params = new URLSearchParams();
  params.set("seconds", String(totalSec));

  const firstVideoId = ids[0];
  if (firstVideoId) params.set("videoId", firstVideoId);
  if (ids.length > 0) params.set("playlist", ids.join(","));

  params.set("muted", "0");

  const fullUrl = buildFullViteUrl(params.toString());
  obsUrl.value = fullUrl;
  setStatus(
    `Generated OBS URL. Total seconds = ${totalSec} (${formatSeconds(totalSec)}).`,
    "ok"
  );
}

async function copyUrl() {
  if (!obsUrl.value) {
    setStatus("Nothing to copy yet. Generate first.", "err");
    return;
  }
  try {
    await navigator.clipboard.writeText(obsUrl.value);
    setStatus("Copied OBS URL to clipboard ✔", "ok");
  } catch (err) {
    console.error(err);
    setStatus("Failed to copy. You can still select & copy manually.", "err");
  }
}

onMounted(() => {
  const handled = initEmbedModeFromUrl();
  if (!handled) {
    embedMode.value = false;
  }
});
</script>
