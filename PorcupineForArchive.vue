<template>
    <div>
        <h1>Détection de Wakeword avec Porcupine</h1>
        <p v-if="state.isLoaded">Status: composant initialisé correctement</p>
        <p v-else>Status: composant pas initialisé</p>
        <div v-if="state.error">{{ state.error }}</div>
        <button class="mt-5" @click="toggleListening">
            {{ state.isListening ? 'Arrêter' : 'Démarrer' }} l'écoute
        </button>
        <div>{{ state.keywordDetection }}</div>
    </div>
</template>

<script setup>
import { BuiltInKeyword } from '@picovoice/porcupine-web';
import { usePorcupine } from '@picovoice/porcupine-vue';
import { attilaKeywordModel } from '@/public/porcupine/attila_model';
import { SystemId, Status } from '@/types/systemstatus';
import { useStatusStore } from '@/stores/systemStatusStore';

// Init stores and keys
const config = useRuntimeConfig();
const statusStore = useStatusStore();

// Déclarer les variables au niveau supérieur
const { state, init, start, stop, release } = usePorcupine();

onMounted(async () => {
    // Initialiser le composant et le status
    console.log('Composant monté. Appel de initializePorcupine');
    statusStore.setStatus({ id: SystemId.PORCUPINE, status: Status.UP })
    await initializePorcupine();
});

const initializePorcupine = async () => {
    try {
        console.log('Début de initializePorcupine');

        // Built in wakeword loading
        const keyword = {
            builtin: BuiltInKeyword.Jarvis,
            sensitivity: 0.7,
        };

        // load custom keyword
        const keyword_fr = {
            label: config.public.companionName,
            base64: attilaKeywordModel,
            sensitivity: 0.7,
            forceWrite: true,
        }

        await init(
            config.app.picoKey, // API to get from PICOVOICE.AI
            [keyword_fr], // init with custom keyword here
            {
                forceWrite: true,
                publicPath: "/porcupine/porcupine_params_fr.pv", // FR porcupine model with params
            }
        );
        if (state.isLoaded) {
            console.log('Initialisation réussie', state);
            statusStore.setStatus({ id: SystemId.PORCUPINE, status: Status.UP })
        } else {
            console.error('Erreur lors de l\'initialisation', state);
            alert("Erreur lors de l'initialisation de Porcupine: " + state.error);
            statusStore.setStatus({ id: SystemId.PORCUPINE, status: Status.DOWN })
        }

    } catch (error) {
        console.error("Erreur d'initialisation détaillée :", error);
        alert("Erreur d'initialisation: " + error);
        statusStore.setStatus({ id: SystemId.PORCUPINE, status: Status.DOWN })
    }
};

const toggleListening = async () => {
    try {
        if (state.isListening) {
            await stop();

        } else {
            await start();
        }
    } catch (error) {
        statusStore.setStatus({ id: SystemId.PORCUPINE, status: Status.DOWN })
        console.error('Erreur lors du toggle:', error);
    }
};

// Surveiller les détections de mots-clés
watch(
    () => state.keywordDetection,
    (detection) => {
        if (detection) {
            console.log('Mot-clé détecté:', detection);

        }
    }
);

// Libérer les ressources lors du démontage
onBeforeUnmount(() => {
    if (state.isLoaded) {
        release();
    }
    statusStore.setStatus({ id: SystemId.PORCUPINE, status: Status.DOWN })

});
</script>

<style scoped></style>
