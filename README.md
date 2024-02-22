# Viem + DApp-Kit

[Viem](https://viem.sh/) is a TypeScript interface for Ethereum, offering low-level, stateless primitives for Ethereum interaction. It serves as an alternative to ethers.js and web3.js, prioritizing reliability, efficiency, and an outstanding developer experience.

[Vechain DAppKit](https://github.com/vechain/vechain-dapp-kit) is a TypeScript library designed to enable smooth interactions between Vechain wallets (such as VeWorld and Sync2) and decentralized applications (dApps), thereby improving both user experience and developer convenience.

Originating from distinct blockchains, they can be integrated to unlock a myriad of applications within the Ethereum ecosystem.

This article outlines the steps to set up a sample project utilizing `viem` for reading and writing data on Vechain.

# Project Setup

Set up a basic React project to be managed by Parcel.js:

```shell
yarn init -y 
yarn config set nodeLinker node-modules
yarn add react react-dom react-router-dom ethers
yarn add @types/react @types/react-dom @types/react-router-dom --dev
yarn add @vechain/web3-providers-connex
yarn add @vechain/dapp-kit @vechain/dapp-kit-react
yarn add viem
yarn add parcel --dev
```

Manually complete the scaffolding of the React app by creating three files:
* `src/index.html`
* `src/index.tsx`
* `src/App.tsx`

Their content can be copied from the [example repository on GitHub](https://github.com/vechain-energy/examples-dappkit-viem/src).

Add script shortcuts in `package.json`:

```json
  "scripts": {
    "start": "parcel src/index.html --open",
    "build": "parcel build src/index.html"
  }
```

This enables `yarn start` to launch a development instance and `yarn build` to create a deployment-ready application.
# Vechain Provider

In `App.tsx` the `DAppKitProvider` brings Connex & Vechain-Connectivity into the project:

```tsx
import { DAppKitProvider } from '@vechain/dapp-kit-react';

export default function App() {

    return (
        <DAppKitProvider
            nodeUrl="https://node-testnet.vechain.energy"
            genesis="test"
            usePersistence
        >
            hello
        </DAppKitProvider>
    );
}
```

Within the provider, `useConnex()` provides access to a connex object with the network settings in of the provider.

# Connect Viem

Viem has helper functions that allow the creation of custom providers. With `@vechain/web3-providers-connex` there is a web3-compatible provider available on the Vechain side.

This hook snippet can combine both into a view publicClient:

```tsx
import { useMemo } from 'react'
import { useConnex } from "@vechain/dapp-kit-react"
import { Provider } from '@vechain/web3-providers-connex'
import { createPublicClient, createWalletClient, custom } from 'viem'

export function useViem() {
    const connex = useConnex()

    const publicClient = useMemo(() => {
        return createPublicClient({
            chain: testnet,
            transport: custom(new Provider({ connex }))
        })
    }, [connex]);

    return {
        publicClient,
    }
}
```

For wallet interactions a wallet client needs more details about the chain to be setup:

```tsx
const testnet = {
    id: 1176455790972829965191905223412607679856028701100105089447013101863,
    name: 'vechain_testnet',
    network: 'homestead',
    nativeCurrency: {
        name: 'VeChainThor',
        symbol: 'VET',
        decimals: 18,
    },
    rpcUrls: {
        public: {
            http: ["https://node-testnet.vechain.energy"]
        },
        default: {
            http: ["https://node-testnet.vechain.energy"]
        }
    },
    blockExplorers: {
        default: { name: 'Explorer', url: 'https://explore-testnet.vechain.org' },
    },
}
```

The `id` represents the numerical identifier of the chain's genesis block.

The process of creating the wallet client closely mirrors that of the public client:

```tsx
createWalletClient({
    chain: testnet,
    transport: custom(new Provider({ connex }))
})
```

## Read with Viem

To test the reading functionality, we will read a number from a test contract with a known ABI using Viem:

```tsx
import React from 'react';
import { useViem } from './useViem';
import { getContract } from 'viem'

export default function Counter() {
    const { publicClient } = useViem();

    const Contract = getContract({
        address: '0x8384738c995d49c5b692560ae688fc8b51af1059',
        abi: [
            {
                "inputs": [],
                "name": "counter",
                "outputs": [
                    {
                        "internalType": "uint256",
                        "name": "",
                        "type": "uint256"
                    }
                ],
                "stateMutability": "view",
                "type": "function"
            }
        ],
        publicClient
    })

    const [value, setValue] = React.useState(0);
    React.useEffect(() => {
        Contract.read.counter().then(value => setValue(Number(value)))
    }, [publicClient])

    return (
        <div>
            <p>Current Counter Value: {value}</p>
        </div>
    )
}
```

## Write with View

To submit a transaction, a simulation must be conducted first, following a similar procedure.

Below is an example that demonstrates calling the `increment` function in a sample counter contract. The account provided to the simulation is anticipated to be the transaction's source. This is the reason `useWallet()` is utilized in this example.

```tsx
import React from 'react';
import { useViem } from './useViem';
import { getContract } from 'viem'
import { useWallet } from '@vechain/dapp-kit-react';

export default function Counter() {
    const { walletClient, publicClient } = useViem();
    const { account } = useWallet();

    const Contract = getContract({
        address: '0x8384738c995d49c5b692560ae688fc8b51af1059',
        abi: [
            {
                "inputs": [],
                "name": "increment",
                "outputs": [],
                "stateMutability": "nonpayable",
                "type": "function"
            }
        ],
        publicClient
    })
    const [txId, setTxId] = React.useState('')
    const [txStatus, setTxStatus] = React.useState('')
    const handleIncrement = async () => {
        if (!account) return
        const { request } = await publicClient.simulateContract({
            account: account as `0x${string}`,
            address: Contract.address,
            abi: Contract.abi,
            functionName: 'increment',
            // args: []
        })
        const txId = await walletClient.writeContract(request)
        setTxId(txId)
        setTxStatus('pending')

        const transaction = await publicClient.waitForTransactionReceipt({ hash: txId })
        setTxStatus(transaction.status)
        console.log(transaction)
    }


    return (
        <div>
            <button onClick={handleIncrement}>Increment</button>
            {Boolean(txId) && <p>Sent Transaction ID: {txId} ({txStatus})</p>}
        </div>
    )
}
```

## Conclusion

The purpose of this article was to introduce new tooling. To explore its capabilities further, visit [viem](https://viem.sh/).

A sample project featuring the snippet above has been published on GitHub.
https://github.com/vechain-energy/examples-dappkit-viem