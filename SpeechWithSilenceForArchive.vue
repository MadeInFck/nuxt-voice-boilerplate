<template>
  <div class="max-w-2xl mx-auto p-4">
    <div class="mb-6 space-y-4">
      <!-- Bouton principal -->
      <button @click="toggleRecognition" :class="[
        'w-full py-3 px-6 rounded-lg font-medium transition-all duration-300',
        isListening
          ? 'bg-red-500 hover:bg-red-600 text-white'
          : 'bg-blue-500 hover:bg-blue-600 text-white'
      ]">
        {{ isListening ? 'Arrêter l\'écoute' : 'Démarrer l\'écoute' }}
      </button>

      <!-- Visualisation du volume -->
      <div class="space-y-2">
        <div class="flex justify-between text-sm">
          <span>Volume: {{ volume.toFixed(2) }}</span>
          <span>Seuil: {{ volumeThreshold }}</span>
        </div>
        <div class="h-2 bg-gray-200 rounded-full overflow-hidden">
          <div class="h-full bg-green-500 transition-all duration-100"
            :style="{ width: `${Math.min(volume * 100, 100)}%` }"></div>
        </div>
      </div>

      <!-- Indicateur d'état -->
      <div class="flex items-center space-x-2">
        <div :class="[
          'w-3 h-3 rounded-full',
          isSpeaking ? 'bg-green-500' : 'bg-red-500'
        ]"></div>
        <span>{{ isSpeaking ? 'Parole détectée' : 'Silence' }}</span>
      </div>

      <!-- Contrôles -->
      <div class="space-y-4 p-4 bg-gray-50 rounded-lg">
        <div>
          <label class="block text-sm font-medium text-gray-700">
            Seuil de volume ({{ volumeThreshold }})
          </label>
          <input v-model="volumeThreshold" type="range" min="0" max="1" step="0.01" class="w-full">
        </div>
        <div>
          <label class="block text-sm font-medium text-gray-700">
            Durée du silence ({{ silenceDuration }}ms)
          </label>
          <input v-model="silenceDuration" type="range" min="100" max="2000" step="100" class="w-full">
        </div>
      </div>
    </div>

    <!-- Transcriptions -->
    <div class="h-64 overflow-y-auto border rounded-lg bg-gray-50 p-2">
      <div class="space-y-2">
        <div v-for="(text, index) in transcripts" :key="index" class="p-3 bg-white rounded shadow text-black">
          {{ text }}
        </div>
      </div>
    </div>

    <div v-if="error" class="mt-4 p-4 bg-red-50 text-red-600 rounded-lg">
      {{ error }}
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { useStatusStore } from '@/stores/systemStatusStore';
import { SystemId, Status } from '@/types/systemstatus'

const statusStore = useStatusStore()

const isListening = ref(false)
const isSpeaking = ref(false)
const volume = ref(0)
const error = ref('')
const transcripts = ref([])

// Paramètres configurables
const volumeThreshold = ref(0.1)
const silenceDuration = ref(1000)

const recognition = ref(null)
let audioStream = null
let audioContext = null
let analyser = null
let dataArray = null
let animationFrame = null
let silenceTimeout = null

const analyzeAudio = () => {
  analyser.getByteFrequencyData(dataArray)
  let values = 0
  let length = dataArray.length

  // Calculer les indices correspondant aux fréquences de la parole
  const sampleRate = audioContext.sampleRate
  const binSize = sampleRate / analyser.fftSize

  const lowerFrequency = 300 // Hz
  const upperFrequency = 3400 // Hz

  const lowerIndex = Math.floor(lowerFrequency / binSize)
  const upperIndex = Math.floor(upperFrequency / binSize)

  for (let i = lowerIndex; i <= upperIndex; i++) {
    values += dataArray[i]
  }

  const adjustedLength = upperIndex - lowerIndex + 1
  volume.value = values / adjustedLength / 255

  if (volume.value > volumeThreshold.value) {
    if (silenceTimeout) {
      clearTimeout(silenceTimeout)
      silenceTimeout = null
    }
    isSpeaking.value = true
  } else {
    if (!silenceTimeout && isSpeaking.value) {
      silenceTimeout = setTimeout(() => {
        isSpeaking.value = false
      }, silenceDuration.value)
    }
  }

  animationFrame = requestAnimationFrame(analyzeAudio)
}

const initAudio = async () => {
  try {
    audioStream = await navigator.mediaDevices.getUserMedia({
      audio: {
        echoCancellation: true,
        noiseSuppression: true,
        autoGainControl: false,
      },
    });

    audioContext = new AudioContext();
    analyser = audioContext.createAnalyser();
    const source = audioContext.createMediaStreamSource(audioStream);

    analyser.fftSize = 256;
    analyser.smoothingTimeConstant = 0.8;

    // Dans initAudio(), après avoir créé 'source'
    const biquadFilter = audioContext.createBiquadFilter()
    biquadFilter.type = 'bandpass'
    biquadFilter.frequency.setValueAtTime(1700, audioContext.currentTime) // Fréquence centrale
    biquadFilter.Q.setValueAtTime(1, audioContext.currentTime) // Qualité du filtre

    // Connecter le filtre
    source.connect(biquadFilter)
    biquadFilter.connect(analyser)

    // Initialiser dataArray ici
    dataArray = new Uint8Array(analyser.frequencyBinCount);

    // Démarrer l'analyse audio
    animationFrame = requestAnimationFrame(analyzeAudio);

    statusStore.setStatus({ id: SystemId.STT, status: Status.UP })

  } catch (error) {
    console.error("Erreur lors de l'initialisation audio :", error);
    statusStore.setStatus({ id: SystemId.STT, status: Status.DOWN })
  }
};

const initSpeechRecognition = () => {
  if (!('webkitSpeechRecognition' in window) && !('SpeechRecognition' in window)) {
    error.value = 'La reconnaissance vocale n\'est pas supportée'
    return
  }

  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition
  recognition.value = new SpeechRecognition()

  recognition.value.continuous = true
  recognition.value.interimResults = true
  recognition.value.lang = 'fr-FR'

  recognition.value.onstart = () => {
    isListening.value = true
    error.value = ''
  }

  recognition.value.onend = () => {
    if (isListening.value && isSpeaking.value) {
      try {
        recognition.value.start()
      } catch (e) {
        console.error('Erreur lors du redémarrage:', e)
      }
    }
  }

  recognition.value.onerror = (event) => {
    if (event.error !== 'no-speech') {
      error.value = `Erreur: ${event.error}`
    }
  }

  recognition.value.onresult = (event) => {
    for (let i = event.resultIndex; i < event.results.length; i++) {
      if (event.results[i].isFinal) {
        transcripts.value.unshift(event.results[i][0].transcript)
      }
    }
  }
}

const toggleRecognition = async () => {
  try {
    if (!recognition.value) {
      initSpeechRecognition()
    }

    if (isListening.value) {
      stopRecognition()
    } else {
      await startRecognition()
    }
  } catch (err) {
    error.value = err.message
  }
}

const startRecognition = async () => {
  await initAudio()
  recognition.value.start()
  analyzeAudio()
}

const stopRecognition = () => {
  isListening.value = false
  isSpeaking.value = false
  recognition.value?.stop()

  if (animationFrame) {
    cancelAnimationFrame(animationFrame)
  }
  if (silenceTimeout) {
    clearTimeout(silenceTimeout)
  }

  if (audioStream) {
    audioStream.getTracks().forEach(track => track.stop())
    audioStream = null
  }
  if (audioContext) {
    audioContext.close()
    audioContext = null
  }
}

onMounted(() => {

  initSpeechRecognition()
})

onUnmounted(() => {
  stopRecognition()
})
</script>
