<script setup lang="ts">
import { computed, defineProps, ref, watch } from "vue"

import {
    createAppKit,
    useAppKit,
    useAppKitAccount,
    useAppKitNetwork,
    useAppKitProvider,
    useDisconnect,
} from "@reown/appkit/vue"

import { EthersAdapter } from "@reown/appkit-adapter-ethers"
import {
    mainnet,
    arbitrum,
    avalanche,
    base,
    linea,
    opBNB,
    optimism,
    polygon,
    scroll,
    type AppKitNetwork,
} from "@reown/appkit/networks"

import { BrowserProvider, type Eip1193Provider, JsonRpcSigner } from "ethers"

import { InitAarcWithEthWalletListener } from "@aarc-dev/wallet-connector-vuejs"

import type { WebClientInterface } from "@aarc-dev/fundkit-web-sdk"

interface Chain {
    id: number
    name: string
    nativeCurrency: {
        name: string
        symbol: string
        decimals: number
    }
    blockExplorers?: {
        default?: {
            url?: string
        }
    }
}

interface ChainDefinition {
    chainIdHex: string
    chainName: string
    nativeCurrency: {
        name: string
        symbol: string
        decimals: number
    }
    rpcUrls: string[]
    blockExplorerUrls: string[]
}

// -- 1. Props: we accept `client` and `debugLog`.
const props = defineProps<{
    client: WebClientInterface
    debugLog?: boolean
}>()

// -- 2. Constants and configuration
const projectId = "YOUR_PROJECT_ID"

const metadata = {
    name: "My Website",
    description: "My Website description",
    url: "https://mywebsite.com", // origin must match your domain & subdomain
    icons: ["https://avatars.mywebsite.com/"],
}

const KNOWN_NETWORKS: [AppKitNetwork, ...AppKitNetwork[]] = [
    mainnet,
    arbitrum,
    base,
    optimism,
    opBNB,
    polygon,
    avalanche,
    linea,
    scroll,
]

const KNOWN_CHAINS: Record<number, ChainDefinition> = KNOWN_NETWORKS.reduce(
    (acc, chain) => {
        acc[Number(chain.id)] = {
            chainName: chain.name,
            chainIdHex: "0x" + chain.id.toString(16),
            nativeCurrency: chain.nativeCurrency,
            rpcUrls: [chain.rpcUrls.default.http[0]],
            blockExplorerUrls: [chain?.blockExplorers?.default.url ?? ""],
        }
        return acc
    },
    {} as Record<number, ChainDefinition>
)

// -- 3. Create the AppKit instance (Vue version).
const modal = createAppKit({
    adapters: [new EthersAdapter()],
    metadata: metadata,
    networks: KNOWN_NETWORKS,
    projectId,
    features: {
        analytics: true,
    },
})

// -- 4. Use composables
const { disconnect } = useDisconnect()
const { open } = useAppKit()
const account = useAppKitAccount()
const appKitNetwork = useAppKitNetwork()
const appKitWalletProvider = useAppKitProvider<Eip1193Provider>("eip155")

// -- 5. Local Refs / Reactive data
const walletClient = ref<AarcEthersEthereumSigner | null>(null)

// -- 6. Switch chain
async function switchChain({ chainId }: { chainId: number }): Promise<void> {
    const network = KNOWN_NETWORKS.find((n) => Number(n.id) === chainId)
    if (!network) {
        throw new Error("Unknown network with chainId: " + chainId)
    }

    // Switch in Reown's AppKit
    await appKitNetwork.value.switchNetwork(network)

    // Switch in the local `modal` instance if needed
    modal.switchNetwork(network)
}

// -- 7. Combined disconnect
async function combinedDisconnect() {
    await disconnect()
}

// -- 8. onClickConnect
async function onClickConnect() {
    if (open) open()
}

// -- 9. Watch or onMounted effect to set walletClient
watch(
    () => appKitWalletProvider.walletProvider,
    async (newProvider) => {
        if (!newProvider) {
            walletClient.value = null
            return
        }
        try {
            const provider = new BrowserProvider(newProvider as Eip1193Provider)
            const signer = await provider.getSigner()
            walletClient.value = new AarcEthersEthereumSigner(
                signer as JsonRpcSigner
            )
            if (props.debugLog)
                console.debug("Wallet client updated:", walletClient.value)
        } catch (err) {
            console.error("Error setting walletClient:", err)
        }
    },
    { immediate: true }
)

// -- 10. Computed chainId (because chainId might be BigInt or string)
const numericChainId = computed(() => {
    if (!appKitNetwork.value.chainId) return 0
    return Number(appKitNetwork.value.chainId)
})

// -- 11. For convenience, build the `chains` array to pass to the listener
const chains = computed<Chain[]>(() =>
    KNOWN_NETWORKS.map((network) => {
        const chain: Chain = {
            id: Number(network.id),
            name: network.name,
            nativeCurrency: {
                name: network.nativeCurrency.name,
                symbol: network.nativeCurrency.symbol,
                decimals: network.nativeCurrency.decimals,
            },
            blockExplorers: network.blockExplorers,
        }
        return chain
    })
)

class AarcEthersEthereumSigner {
    /**
     * Ethers Signer (e.g. from a BrowserProvider + getSigner()).
     * This Signer must be connected to window.ethereum or your chosen provider.
     */
    public signer: JsonRpcSigner

    /**
     * Optional: track a "current" block explorer URL if you want to store it
     * after switching to a known chain. If not needed, you can remove. it helps internal UI to get the block explorer URL.
     */
    public blockChainExplorerUrl?: string

    constructor(signer: JsonRpcSigner) {
        this.signer = signer
    }

    /**
     * Helper to switch chains in MetaMask (or any injected provider):
     * - If chain is recognized by user’s wallet, it will switch directly.
     * - If not recognized, attempts to 'addChain' first, then switch again.
     */
    private async handleSwitchChain(onChainID: string) {
        // Convert the chainId from string to number
        const chainIdNumber = Number(onChainID)

        // We'll build the hex string for chainId (e.g. "0x1" for Ethereum)
        const chainIdHex = "0x" + chainIdNumber.toString(16)

        try {
            // Attempt to switch to that chain
            await (window.ethereum as unknown as Eip1193Provider)?.request?.({
                method: "wallet_switchEthereumChain",
                params: [{ chainId: chainIdHex }],
            })
        } catch (error: any) {
            // If the chain is not recognized by MetaMask, you may get an error like:
            // "Unrecognized chain ID" or 4902 error code => need to addChain first
            if (
                error.code === 4902 ||
                (typeof error.message === "string" &&
                    error.message.includes("Unrecognized chain ID"))
            ) {
                const chainToAdd = KNOWN_CHAINS[chainIdNumber]
                if (!chainToAdd) {
                    throw new Error(
                        `Chain with ID ${chainIdNumber} not found in KNOWN_CHAINS`
                    )
                }
                // Use wallet_addEthereumChain to register it in MetaMask
                await (
                    window.ethereum as unknown as Eip1193Provider
                )?.request?.({
                    method: "wallet_addEthereumChain",
                    params: [
                        {
                            chainId: chainToAdd.chainIdHex,
                            chainName: chainToAdd.chainName,
                            nativeCurrency: chainToAdd.nativeCurrency,
                            rpcUrls: chainToAdd.rpcUrls,
                            blockExplorerUrls: chainToAdd.blockExplorerUrls,
                        },
                    ],
                })
                // Then switch again
                await (
                    window.ethereum as unknown as Eip1193Provider
                )?.request?.({
                    method: "wallet_switchEthereumChain",
                    params: [{ chainId: chainToAdd.chainIdHex }],
                })
            } else {
                console.error("Error switching chain", error)
                throw error
            }
        }

        // If we have a known chain object, store its explorer
        const knownChain = KNOWN_CHAINS[chainIdNumber]
        if (knownChain && knownChain.blockExplorerUrls?.length > 0) {
            this.blockChainExplorerUrl = knownChain.blockExplorerUrls[0]
        }
    }

    /**
     * Send a generic transaction with 'to', 'data', 'value', etc.
     */
    async sendTransaction(transaction: {
        to: string
        value: string
        gasLimit?: string
        data?: string
        chainId?: string
        from?: string
    }): Promise<string> {
        // Switch chain if needed
        if (transaction.chainId) {
            await this.handleSwitchChain(transaction.chainId)
        }

        const tx = await this.signer.sendTransaction({
            to: transaction.to,
            data: transaction.data || undefined,
            value: transaction.value ? BigInt(transaction.value) : "0x0",
            // Ethers v6 automatically estimates gas if not provided
            gasLimit: transaction.gasLimit
                ? BigInt(transaction.gasLimit)
                : undefined,
        })

        return tx.hash
    }
}
</script>

<template>
    <InitAarcWithEthWalletListener
        :client="client"
        :debug-log="debugLog"
        :chains="chains"
        :chain-id="numericChainId"
        :address="account.address"
        :disconnect-async="combinedDisconnect"
        :on-click-connect="onClickConnect"
        gas-price="undefined"
        :wallet-client="walletClient"
        :switch-chain="switchChain"
    />
</template>
