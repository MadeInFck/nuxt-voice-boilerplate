<!-- components/SpeechRecognition.vue -->
<template>
    <div class="max-w-2xl mx-auto p-4">
        <!-- Contrôles principaux -->
        <div class="mb-6 space-y-4">
            <!-- Bouton principal -->
            <button @click="toggleRecognition" :class="[
                'w-full py-3 px-6 rounded-lg font-medium transition-all duration-300',
                isListening
                    ? 'bg-red-500 hover:bg-red-600 text-white'
                    : 'bg-blue-500 hover:bg-blue-600 text-white'
            ]">
                <div class="flex items-center justify-center space-x-2">
                    <span v-if="isListening" class="animate-pulse">
                        <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6" fill="none" viewBox="0 0 24 24"
                            stroke="currentColor">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                                d="M19 11a7 7 0 01-7 7m0 0a7 7 0 01-7-7m7 7v4m0 0H8m4 0h4m-4-8a3 3 0 01-3-3V5a3 3 0 116 0v6a3 3 0 01-3 3z" />
                        </svg>
                    </span>
                    <span>{{ isListening ? 'Arrêter l\'écoute' : 'Démarrer l\'écoute' }}</span>
                </div>
            </button>

            <!-- Sélection de la langue -->
            <select v-model="selectedLanguage"
                class="w-full p-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500">
                <option value="fr-FR">Français</option>
                <option value="en-US">English (US)</option>
                <option value="es-ES">Español</option>
            </select>
        </div>

        <!-- Status d'écoute -->
        <div v-if="isListening" class="flex items-center space-x-2 text-gray-600 mb-4">
            <div class="w-2 h-2 bg-red-500 rounded-full animate-pulse"></div>
            <span>Écoute en cours...</span>
        </div>

        <!-- Résultat temporaire -->
        <div v-if="interimResult" class="bg-gray-50 p-4 rounded-lg mb-4 italic text-gray-600">
            {{ interimResult }}
        </div>

        <!-- Liste des transcriptions -->
        <div v-if="transcripts.length" class="space-y-4">
            <h3 class="text-lg font-semibold text-gray-800">Transcriptions</h3>

            <div class="space-y-2">
                <div v-for="(transcript, index) in transcripts" :key="index" :class="[
                    'p-4 rounded-lg transition-all duration-300',
                    transcript.confidence > 0.8
                        ? 'bg-green-50 border border-green-100'
                        : 'bg-gray-50 border border-gray-100'
                ]">
                    <div class="flex justify-between items-start">
                        <p class="text-gray-800">{{ transcript.text }}</p>
                        <span class="text-xs text-gray-500">{{ transcript.time }}</span>
                    </div>
                    <div class="mt-1 h-1 w-full bg-gray-200 rounded">
                        <div class="h-1 bg-green-500 rounded" :style="{ width: `${transcript.confidence * 100}%` }">
                        </div>
                    </div>
                </div>
            </div>

            <!-- Bouton pour effacer -->
            <button @click="clearTranscripts"
                class="text-sm text-red-500 hover:text-red-600 transition-colors duration-300">
                Effacer l'historique
            </button>
        </div>

        <!-- Message d'erreur -->
        <div v-if="error" class="mt-4 p-4 bg-red-50 border border-red-200 rounded-lg text-red-600">
            {{ error }}
        </div>
    </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { useStatusStore } from '@/stores/systemStatusStore'
import { Status, SystemId } from "@/types/systemstatus";

// Declare store for status to update status of the Web Speech Recognition module
const statusStore = useStatusStore();

const isListening = ref(false)
const interimResult = ref('')
const transcripts = ref([])
const error = ref('')
const recognition = ref(null)
const selectedLanguage = ref('fr-FR')

const initializeSpeechRecognition = () => {
    try {
        if (!('webkitSpeechRecognition' in window) && !('SpeechRecognition' in window)) {
            throw new Error('La reconnaissance vocale n\'est pas supportée par votre navigateur.')
        }

        const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition
        recognition.value = new SpeechRecognition()

        recognition.value.continuous = true
        recognition.value.interimResults = true
        recognition.value.lang = selectedLanguage.value

        recognition.value.onstart = () => {
            isListening.value = true
            error.value = ''
        }

        recognition.value.onend = () => {
            isListening.value = false
        }

        recognition.value.onerror = (event) => {
            error.value = `Erreur: ${event.error}`
            isListening.value = false
        }

        recognition.value.onresult = (event) => {
            for (let i = event.resultIndex; i < event.results.length; i++) {
                const result = event.results[i]

                if (result.isFinal) {
                    transcripts.value.push({
                        text: result[0].transcript,
                        confidence: result[0].confidence,
                        time: new Date().toLocaleTimeString()
                    })
                    interimResult.value = ''
                } else {
                    interimResult.value = result[0].transcript
                }
            }
        }
        statusStore.setStatus({ id: SystemId.STT, status: Status.UP })
    } catch (err) {
        statusStore.setStatus({ id: SystemId.STT, status: Status.DOWN })
        error.value = err.message
    }
}

const toggleRecognition = () => {
    if (!recognition.value) {
        initializeSpeechRecognition()
    }

    if (isListening.value) {
        recognition.value.stop()
    } else {
        try {
            recognition.value.start()
        } catch (err) {
            error.value = 'Erreur lors du démarrage de la reconnaissance vocale'
        }
    }
}

const clearTranscripts = () => {
    transcripts.value = []
    interimResult.value = ''
}

watch(selectedLanguage, (newLang) => {
    if (recognition.value) {
        recognition.value.lang = newLang
        if (isListening.value) {
            recognition.value.stop()
            recognition.value.start()
        }
    }
})

onMounted(() => {
    initializeSpeechRecognition()
})

onUnmounted(() => {
    if (recognition.value && isListening.value) {
        recognition.value.stop()
    }
})
</script>
