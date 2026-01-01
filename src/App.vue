<script setup>
import { ref, onMounted, onUnmounted } from "vue"

const isListening = ref(false)
const transcript = ref("")
const statusMessage = ref("待機中")
let recognition = null
let isManualStop = false
let restartTimeout = null
let soundTimeout = null
let lockTimeout = null

// ターゲットワード（表記揺れ対策）
const targetWords = [
    "ぶいあーるちゃっとで",
    "ブイアールチャットで",
    "無理やりチャットで",
    "無理矢理チャットで",
    "VRチャットで",
    "VRちゃっとで",
    "VRChatで",
    "VRchatで",
    "VRChatA",
    "VRchatA",
    "VRChata",
    "VRchata",
]

// 音声ファイル候補 (publicフォルダに配置)
const audioOptions = [
    { label: "日本語機械音声", path: "/ja_translate.mp3" },
    { label: "英語機械音声", path: "/en_translate.mp3" },
    { label: "ゆっくりボイス", path: "/yukkuri.mp3" },
    { label: "すぅぅぅぅ", path: "/suuu.mp3" },
]
const selectedAudio = ref(audioOptions[0].path)
let audio = new Audio(selectedAudio.value)

onMounted(() => {
    const SpeechRecognition =
        window.SpeechRecognition || window.webkitSpeechRecognition
    if (!SpeechRecognition) {
        statusMessage.value = "このブラウザはWeb Speech APIに対応していません。"
        return
    }

    recognition = new SpeechRecognition()
    recognition.lang = "ja-JP"
    recognition.interimResults = true
    recognition.continuous = true

    recognition.onstart = () => {
        isListening.value = true
        statusMessage.value = "監視中..."
        isManualStop = false
    }

    recognition.onend = () => {
        isListening.value = false
        if (!isManualStop) {
            console.log("再起動します...")
            startRecognition()
        } else {
            statusMessage.value = lockTimeout
                ? "音声を再生します"
                : "停止しました"
        }
    }

    recognition.onresult = (event) => {
        let interimTranscript = ""
        let finalTranscript = ""

        for (let i = event.resultIndex; i < event.results.length; ++i) {
            if (event.results[i].isFinal) {
                finalTranscript += event.results[i][0].transcript
            } else {
                interimTranscript += event.results[i][0].transcript
            }
        }

        const sanitizedTranscript = (
            finalTranscript || interimTranscript
        ).replace(/ /g, "")
        transcript.value = sanitizedTranscript
        checkWords(transcript.value)
    }

    recognition.onerror = (event) => {
        console.error("Speech recognition error", event.error)
        switch (event.error) {
            case "not-allowed":
                statusMessage.value = "マイクの使用が許可されていません。"
                isManualStop = true
                recognition.stop()
                break
            case "network":
                statusMessage.value =
                    "ネットワークエラー。接続を確認して再試行します。"
                isManualStop = true
                clearTimeout(restartTimeout)
                restartTimeout = setTimeout(() => {
                    isManualStop = false
                    startRecognition()
                }, 3000)
                break
            case "no-speech":
                statusMessage.value = "音声が検出されませんでした。"
                break
            case "aborted":
                statusMessage.value = "認識が中断されました。"
                break
            default:
                statusMessage.value = `エラーが発生しました: ${event.error}`
        }
    }
})

onUnmounted(() => {
    if (recognition) {
        isManualStop = true
        recognition.stop()
    }
    clearTimeout(restartTimeout)
    clearTimeout(soundTimeout)
    clearTimeout(lockTimeout)
})

const startRecognition = () => {
    if (!recognition) return
    try {
        recognition.start()
    } catch (e) {
        console.error("Already started", e)
    }
}

const stopRecognition = () => {
    if (!recognition) return
    isManualStop = true
    recognition.stop()
}

const setAudio = (path) => {
    if (selectedAudio.value === path) return
    selectedAudio.value = path
    audio.pause()
    audio.currentTime = 0
    audio.src = path
    audio.load()
}

const checkWords = (text) => {
    const normalizedText = text.replace(/\s+/g, "")
    for (const word of targetWords) {
        if (normalizedText.includes(word.replace(/\s+/g, ""))) {
            triggerAction(word)
            break
        }
    }
}

const triggerAction = (word) => {
    transcript.value = ""

    isManualStop = true
    recognition?.stop()

    clearTimeout(soundTimeout)
    soundTimeout = setTimeout(() => {
        playSound()
    }, 500)

    clearTimeout(lockTimeout)
    lockTimeout = setTimeout(() => {
        isManualStop = false
        statusMessage.value = "監視中..."
        startRecognition()
        lockTimeout = null
    }, 1500)
}

const playSound = () => {
    audio.currentTime = 0
    audio.play().catch((e) => console.error("Audio play failed", e))
}
</script>

<template>
    <div class="container">
        <h1>VRChatでびゅー<br />代行サービス</h1>

        <div class="controls">
            <button
                v-if="!isListening"
                @click="startRecognition"
                class="btn start"
            >
                開始
            </button>
            <button v-else @click="stopRecognition" class="btn stop">
                停止
            </button>
        </div>

        <div class="audio-select">
            <h3>でびゅー音を選択</h3>
            <div class="audio-buttons">
                <button
                    v-for="option in audioOptions"
                    :key="option.path"
                    @click="setAudio(option.path)"
                    :class="[
                        'btn',
                        'audio-btn',
                        { active: selectedAudio === option.path },
                    ]"
                >
                    {{ option.label }}
                </button>
            </div>
        </div>

        <div class="status" :class="{ active: isListening, play: lockTimeout }">
            ステータス: {{ statusMessage }}
            <span v-if="isListening" class="indicator">●</span>
        </div>

        <div class="transcript-box">
            <h3>認識中のテキスト:</h3>
            <p>{{ transcript }}</p>
        </div>
    </div>
</template>

<style scoped>
.container {
    max-width: 600px;
    margin: 0 auto;
    text-align: center;
    font-family: sans-serif;
    padding: 2rem;
}

.btn {
    font-size: 1.2rem;
    padding: 10px 20px;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    margin: 10px;
}

.start {
    background-color: #42b983;
    color: white;
}

.stop {
    background-color: #e74c3c;
    color: white;
}

.audio-select {
    margin: 10px 0 20px;
}

.audio-buttons {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 8px;
}

.audio-btn {
    background-color: #f4f4f4;
    color: #333;
    border: 1px solid #dcdcdc;
}

.audio-btn.active {
    border: 2px solid #42b983;
    color: #42b983;
    background-color: #e8f7f0;
}

.status {
    margin: 20px 0;
    font-weight: bold;
}

.status.active {
    color: #42b983;
}

.status.play {
    color: #e2d137;
}

.indicator {
    color: red;
    animation: blink 1s infinite;
}

@keyframes blink {
    0% {
        opacity: 1;
    }
    50% {
        opacity: 0;
    }
    100% {
        opacity: 1;
    }
}

.transcript-box {
    border: 1px solid #ccc;
    padding: 10px;
    min-height: 100px;
    border-radius: 5px;
    background-color: #f9f9f9;
    color: #333;
}

@keyframes pop {
    0% {
        transform: scale(0.8);
        opacity: 0;
    }
    100% {
        transform: scale(1);
        opacity: 1;
    }
}
</style>
