<script setup>
import { ref, onMounted } from "vue";
import axios from "axios";
import { createAlchemyWeb3 } from "@alch/alchemy-web3";

const generalError = ref("");
const succeedNotification = ref("");

const wallet = ref("");

const title = ref("");
const description = ref("");

const noAsset = ref(true);
const fileChange = (e) => (noAsset.value = !e.target.files.length);

const contractAddress = import.meta.env.VITE_CONTRACT_ADDRESS;
import contractABI from "../contract-abi.json";

const web3 = createAlchemyWeb3(import.meta.env.VITE_ALCHEMY_URL);

const connectToWallet = async () => {
    try {
        const addresses = await window.ethereum.request({
            method: "eth_requestAccounts",
        });
        wallet.value = addresses[0];
    } catch (err) {
        generalError.value = err.message;
    }
};

onMounted(async () => {
    if (!window.ethereum) {
        generalError.value = "Please make sure Metamask is installed.";
        return;
    }

    try {
        const addressArray = await window.ethereum.request({
            method: "eth_accounts",
        });
        if (addressArray.length > 0) {
            wallet.value = addressArray[0];
            generalError.value = "";
        }
    } catch (err) {
        generalError.value = err.message;
    }
});

const uploadToIpfs = async () => {
    const url = import.meta.env.VITE_PINATA_ENDPOINT;
    try {
        const formdata = new FormData();
        const file = document.querySelector("input[type=file]").files[0];

        formdata.append("file", file);
        formdata.append(
            "pinataMetadata",
            JSON.stringify({
                name: file.name,
            })
        );

        const response = await axios({
            method: "post",
            maxBodyLength: "Infinity",
            url: `${url}/pinFileToIpfs`,
            data: formdata,
            headers: {
                "Content-Type": `multipart/form-data; boundary=${formdata._boundary}`,
                pinata_api_key: import.meta.env.VITE_PINATA_API_KEY,
                pinata_secret_api_key: import.meta.env.VITE_PINATA_API_SECRET,
            },
        });

        const assetUrl = `https://gateway.pinata.cloud/ipfs/${response.data.IpfsHash}`;

        const res = await axios.post(
            `${url}/pinJSONToIpfs`,
            {
                pinataMetadata: {
                    name: title.value,
                },
                pinataContent: {
                    name: title.value,
                    image: assetUrl,
                    description: description.value,
                },
            },
            {
                headers: {
                    pinata_api_key: import.meta.env.VITE_PINATA_API_KEY,
                    pinata_secret_api_key: import.meta.env
                        .VITE_PINATA_API_SECRET,
                },
            }
        );
        return `https://gateway.pinata.cloud/ipfs/${res.data.IpfsHash}`;
    } catch (ex) {
        generalError.value = ex.message;
    }
};

const onSubmit = async (e) => {
    e.preventDefault();
    generalError.value = "";
    succeedNotification.value = "";

    if (!wallet.value) {
        generalError.value = "Please connect to your metamask wallet.";
        return;
    }

    if (!description.value || !title.value || noAsset.value) {
        return;
    }

    const tokenURI = await uploadToIpfs();
    window.contract = await new web3.eth.Contract(contractABI, contractAddress);

    const transactionParameters = {
        to: contractAddress,
        from: window.ethereum.selectedAddress,
        data: window.contract.methods
            .awardItem(window.ethereum.selectedAddress, tokenURI)
            .encodeABI(),
    };

    try {
        const txHash = await window.ethereum.request({
            method: "eth_sendTransaction",
            params: [transactionParameters],
        });
        succeedNotification.value = `<a href="https://rinkeby.etherscan.io/tx/${txHash}" target="_blank" class="underline">Check your transaction on etherscan</a>`;
    } catch (error) {
        generalError.value = error.message;
    }
};
</script>

<template>
    <div class="w-4/12 mx-auto px-4 bg-white p-5 rounded-md">
        <div class="flex justify-between">
            <h1 class="text-3xl font-bold mb-5">NFT Minter</h1>
            <div>
                <button
                    class="border-teal-500 hover:border-teal-700 hover:text-teal-700 text-sm border-4 text-teal-500 p-2 rounded-xl font-bold"
                    type="button"
                    @click="connectToWallet"
                >
                    {{
                        wallet
                            ? `Connected: ${wallet.substr(0, 10)}...`
                            : "Connect Wallet"
                    }}
                </button>
            </div>
        </div>

        <form class="w-full" @submit="onSubmit">
            <p
                class="text-red-500 text-s italic pb-5"
                :class="!generalError ? 'hidden' : ''"
            >
                {{ generalError }}
            </p>

            <p
                class="text-teal-500 text-s italic pb-5"
                :class="!succeedNotification ? 'hidden' : ''"
                v-html="succeedNotification"
            ></p>

            <div class="flex flex-wrap -mx-3 mb-6">
                <div class="w-full px-3">
                    <label
                        class="block uppercase tracking-wide text-gray-700 text-xs font-bold mb-2"
                        for="title"
                    >
                        Title
                    </label>
                    <input
                        class="appearance-none block w-full bg-gray-200 text-gray-700 border rounded py-3 px-4 mb-3 leading-tight focus:outline-none focus:bg-white"
                        :class="
                            title
                                ? 'border-gray-200 focus:border-gray-500'
                                : 'border-red-500'
                        "
                        id="title"
                        name="title"
                        type="text"
                        placeholder="Title"
                        required
                        v-model="title"
                    />
                    <p v-if="!title" class="text-red-500 text-xs italic">
                        Please fill out this field.
                    </p>
                </div>
            </div>
            <div class="flex flex-wrap -mx-3 mb-6">
                <div class="w-full px-3">
                    <label
                        class="block uppercase tracking-wide text-gray-700 text-xs font-bold mb-2"
                        for="description"
                    >
                        Description
                    </label>
                    <input
                        class="appearance-none block w-full bg-gray-200 text-gray-700 border rounded py-3 px-4 mb-3 leading-tight focus:outline-none focus:bg-white"
                        :class="
                            description
                                ? 'border-gray-200 focus:border-gray-500'
                                : 'border-red-500'
                        "
                        id="description"
                        type="text"
                        placeholder="Description"
                        required
                        v-model="description"
                    />
                    <p v-if="!description" class="text-red-500 text-xs italic">
                        Please fill out this field.
                    </p>
                </div>
            </div>
            <div class="flex flex-wrap -mx-3 mb-6">
                <div class="w-full px-3">
                    <label
                        class="block uppercase tracking-wide text-gray-700 text-xs font-bold mb-2"
                        for="asset"
                    >
                        Asset
                    </label>
                    <input
                        class="appearance-none block w-full bg-gray-200 text-gray-700 border rounded py-3 px-4 mb-3 leading-tight focus:outline-none focus:bg-white"
                        :class="
                            !noAsset
                                ? 'border-gray-200 focus:border-gray-500'
                                : 'border-red-500'
                        "
                        id="asset"
                        type="file"
                        placeholder="Asset"
                        required
                        @change="fileChange"
                    />
                    <p v-if="noAsset" class="text-red-500 text-xs italic">
                        Please fill out this field.
                    </p>
                </div>
            </div>
            <div>
                <button
                    class="flex-shrink-0 bg-teal-500 hover:bg-teal-700 border-teal-500 hover:border-teal-700 text-sm border-4 text-white py-1 px-2 rounded"
                    type="submit"
                >
                    Mint NFT
                </button>
            </div>
        </form>
    </div>
</template>
