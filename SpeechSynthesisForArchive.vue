<!-- components/SimpleTextToSpeech.vue -->
<template>
    <div class="max-w-2xl mx-auto p-4 space-y-6">
        <!-- Textarea -->
        <div class="space-y-2">
            <label class="block text-sm font-medium text-gray-700">
                Texte à prononcer
            </label>
            <textarea v-model="text" rows="4"
                class="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500 resize-none"
                placeholder="Entrez votre texte ici..."></textarea>
        </div>

        <!-- Contrôles basiques -->
        <div class="flex gap-4">
            <button @click="speak" :disabled="!text || isSpeaking" class="flex-1 bg-blue-500 text-white py-2 px-4 rounded-lg hover:bg-blue-600 
               disabled:opacity-50 disabled:cursor-not-allowed transition-all duration-300">
                {{ isSpeaking ? 'En cours...' : 'Lire' }}
            </button>

            <button @click="stop" :disabled="!isSpeaking" class="px-4 py-2 bg-red-500 text-white rounded-lg hover:bg-red-600 
               disabled:opacity-50 disabled:cursor-not-allowed transition-all duration-300">
                Stop
            </button>
        </div>

        <!-- Sélection de la voix -->
        <div class="space-y-2">
            <label class="block text-sm font-medium text-gray-700">
                Voix
            </label>
            <select v-model="selectedVoice" class="w-full p-2 border border-gray-300 rounded-lg">
                <option v-for="voice in voices" :key="voice.name" :value="voice">
                    {{ voice.name }} ({{ voice.lang }})
                </option>
            </select>
        </div>

        <!-- Message d'erreur -->
        <div v-if="error" class="p-4 bg-red-50 border border-red-200 rounded-lg text-red-600">
            {{ error }}
        </div>
    </div>
</template>

<script setup lang="ts">
import { useStatusStore } from '@/stores/systemStatusStore';
import { SystemId, Status } from '@/types/systemstatus';

const statusStore = useStatusStore()

const text = ref('')
const voices = ref([])
const selectedVoice = ref(null)
const isSpeaking = ref(false)
const error = ref('')

// Initialisation des voix disponibles
const loadVoices = () => {
    try {
        voices.value = window.speechSynthesis.getVoices()
        statusStore.setStatus({ id: SystemId.TTS, status: Status.UP })

        // Sélectionner par défaut une voix française si disponible
        selectedVoice.value = voices.value.find(voice => voice.lang.includes('fr')) || voices.value[0]
    } catch (err) {
        statusStore.setStatus({ id: SystemId.TTS, status: Status.DOWN })
        error.value = "Erreur lors du chargement des voix"
        console.error(err)
    }
}

// Fonction pour parler
const speak = () => {
    try {
        if (!text.value.trim()) return

        // Créer un nouvel utterance
        const utterance = new SpeechSynthesisUtterance(text.value)

        // Configurer la voix si sélectionnée
        if (selectedVoice.value) {
            utterance.voice = selectedVoice.value
        }

        // Événements
        utterance.onstart = () => {
            isSpeaking.value = true
            error.value = ''
        }

        utterance.onend = () => {
            isSpeaking.value = false
        }

        utterance.onerror = (event) => {
            statusStore.setStatus({ id: SystemId.TTS, status: Status.DOWN })
            error.value = `Erreur lors de la synthèse vocale: ${event.error}`
            isSpeaking.value = false
        }

        // Lancer la synthèse
        window.speechSynthesis.speak(utterance)

    } catch (err) {
        statusStore.setStatus({ id: SystemId.TTS, status: Status.DOWN })
        error.value = "Erreur lors de la synthèse vocale"
        console.error(err)
    }
}

// Fonction pour arrêter
const stop = () => {
    window.speechSynthesis.cancel()
    isSpeaking.value = false
}

// Initialisation au montage du composant
onMounted(() => {
    if (typeof window !== 'undefined' && 'speechSynthesis' in window) {
        // set system status to UP
        statusStore.setStatus({ id: SystemId.TTS, status: Status.UP })
        // Chargement initial des voix
        loadVoices()
        // Les voix peuvent être chargées de manière asynchrone
        window.speechSynthesis.onvoiceschanged = loadVoices
    } else {
        // set system status to DOWN
        statusStore.setStatus({ id: SystemId.TTS, status: Status.DOWN })
        error.value = "La synthèse vocale n'est pas supportée par votre navigateur"
    }
})

onUnmounted(() => {

})
</script>
