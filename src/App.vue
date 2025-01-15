<script setup lang="ts">
import {
    AarcFundKitModal,
    type FKConfig,
    SourceConnectorName,
    ThemeName,
    type TransactionErrorData,
    type TransactionSuccessData,
} from "@aarc-dev/fundkit-web-sdk"
import { ref } from "vue"
import ReownConnector from "./components/AarcReownConnector.vue"

const urlParams = new URLSearchParams(window.location.search)
const iframeUrl = urlParams.get("iframeUrl") || import.meta.env.VITE_IFRAME_URL
const apiKey = urlParams.get("apiKey") || import.meta.env.VITE_API_KEY

const config: FKConfig = {
    appName: "Dapp Name",
    module: {
        exchange: {
            enabled: true,
        },
        onRamp: {
            enabled: true,
        },
        bridgeAndSwap: {
            enabled: true,
            fetchOnlyDestinationBalance: false,
            routeType: "Value",
            connectors: [SourceConnectorName.ETHEREUM],
        },
    },
    destination: {
        walletAddress: "0xa48EdE22744A055bfa5EA4be441742299c684efB"
    },
    appearance: {
        themeColor: "#A5E547",
        textColor: "#2D2D2D",
        backgroundColor: "#FAFAFA",
        dark: {
            themeColor: "#A5E547", // #2D2D2D
            textColor: "#FFF", // #FFF
            backgroundColor: "#2D2D2D", // #2D2D2D
            highlightColor: "#08091B", // #FFF
            borderColor: "#424242",
        },
        theme: ThemeName.DARK,
        roundness: 24,
    },
    apiKeys: {
        aarcSDK: apiKey,
    },
    events: {
        onTransactionSuccess: (data: TransactionSuccessData) => {
            console.log("client onTransactionSuccess", data)
        },
        onTransactionError: (data: TransactionErrorData) => {
            console.log("client onTransactionError", data)
        },
        onWidgetClose: () => {
            console.log("client onWidgetClose")
        },
        onWidgetOpen: () => {
            console.log("client onWidgetOpen")
        },
    },
    origin: window.location.origin,
}

const aarcModal = ref(
    new AarcFundKitModal(
        config,
        
    )
)

const address = ref(config.destination.walletAddress)

const handleSubmit = () => {
    console.log("Modal opened with address:", address.value)
    aarcModal.value.openModal()
}
</script>

<template>
    <ReownConnector :client="aarcModal" :debug-log="true" />
    <a>
        <img
            src="https://cdn-icons-png.flaticon.com/512/4943/4943421.png"
            alt="Deposit"
            class="clickable-image"
            @click="handleSubmit"
            style="width: 100px; height: 100px; cursor: pointer"
        />
    </a>
</template>

<style scoped>
.logo {
    height: 6em;
    padding: 1.5em;
    will-change: filter;
    transition: filter 300ms;
}
.logo:hover {
    filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
    filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
