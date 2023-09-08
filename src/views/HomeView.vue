<template>
  <main
    class="bg-gray-900 min-h-screen flex items-center justify-center flex-col p-20"
  >
    <div
      class="rounded-md border border-gray-700 text-white bg-gray-800 p-6 mx-auto w-full max-w-[600px]"
    >
      <h1 class="text-2xl mb-4">𝕏 (Twitter) Descentralizado</h1>
      <p class="text-base mb-4">
        Esse é um twitter descentralizado, conecte sua sua carteira blockchain e
        use seus Ethereums para enviar uma mensagem. Cada post enviado você terá
        chance de ganhar um valor de Ethereum de volta.
      </p>
      <div class="mb-2" v-if="currentAccount">
        <label
          for="post"
          class="block mb-2 text-sm font-medium text-gray-900 dark:text-white"
          >Post</label
        >
        <input
          v-model="message"
          type="text"
          id="post"
          class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
          placeholder="John"
          required
        />
      </div>
      <button
        v-if="currentAccount"
        @click="wave"
        type="button"
        class="text-white bg-blue-700 hover:bg-blue-800 focus:ring-4 focus:ring-blue-300 font-medium rounded-lg text-sm px-5 py-2.5 mr-2 mb-2 dark:bg-blue-600 dark:hover:bg-blue-700 focus:outline-none dark:focus:ring-blue-800"
      >
        <span v-if="loading">Carregando...</span>
        <span v-else>Enviar post</span>
      </button>
      <button
        v-if="!currentAccount"
        @click="connectWallet"
        type="button"
        class="py-2.5 px-5 mr-2 mb-2 text-sm font-medium text-gray-900 focus:outline-none bg-white rounded-lg border border-gray-200 hover:bg-gray-100 hover:text-blue-700 focus:z-10 focus:ring-4 focus:ring-gray-200 dark:focus:ring-gray-700 dark:bg-gray-800 dark:text-gray-400 dark:border-gray-600 dark:hover:text-white dark:hover:bg-gray-700"
      >
        <span v-if="loading">Carregando...</span>
        <span v-else>Conectar carteira</span>
      </button>
      <div v-else class="flex items-center">
        <span
          class="bg-green-100 text-green-800 text-xs font-semibold mr-2 px-2.5 py-0.5 rounded dark:bg-green-200 dark:text-green-900 h-fit"
        >
          Conectado!
        </span>
        <span class="truncate">
          {{ currentAccount }}
        </span>
      </div>
    </div>
    <div
      class="mt-8 rounded-md border border-gray-700 text-white bg-gray-800 p-6 mx-auto w-full max-w-[600px]"
      v-if="allWaves?.length > 0"
    >
      <h1 class="text-white text-lg mb-4">Todos os posts</h1>
      <div v-if="loadingWave" class="text-center mb-4">Carregando...</div>
      <div
        class="border border-gray-700 p-4 rounded mb-2"
        v-for="wave in allWaves"
        :key="wave"
      >
        <div class="truncate">Carteira: {{ wave.address }}</div>
        <div>Data: {{ wave.timestamp.toString() }}</div>
        <div>Mensagem: {{ wave.message }}</div>
      </div>
    </div>
  </main>
</template>

<script setup>
import { ref, onMounted, watchEffect } from "vue";
import { ethers } from "ethers";
import abi from "@/utils/WavePortal.json";

const currentAccount = ref(null);
const loading = ref(false);
const loadingWave = ref(false);
const message = ref(null);

const contractAddress = "0x250896E502c2E09F6C440bA9d7DAb63276EF9368";
const contractABI = abi.abi;

const getEthereumObject = () => window.ethereum;

const findMetaMaskAccount = async () => {
  loading.value = true;
  try {
    const ethereum = getEthereumObject();

    /*
     * First make sure we have access to the Ethereum object.
     */
    if (!ethereum) {
      console.error("Make sure you have Metamask!");
      return null;
    }

    console.log("We have the Ethereum object", ethereum);
    const accounts = await ethereum.request({ method: "eth_accounts" });
    loading.value = false;

    if (accounts.length !== 0) {
      const account = accounts[0];
      console.log("Found an authorized account:", account);
      currentAccount.value = account;
      return account;
    } else {
      console.error("No authorized account found");
      return null;
    }
  } catch (error) {
    loading.value = false;
    console.error(error);
    return null;
  }
};

const connectWallet = async () => {
  loading.value = true;
  try {
    const ethereum = getEthereumObject();
    if (!ethereum) {
      alert("Get MetaMask!");
      return;
    }

    const accounts = await ethereum.request({
      method: "eth_requestAccounts",
    });

    console.log("Connected", accounts[0]);
    currentAccount.value = accounts[0];
    getAllWaves();
  } catch (error) {
    console.error(error);
  }
  loading.value = false;
};

const wave = async () => {
  loadingWave.value = true;
  try {
    const { ethereum } = window;

    if (ethereum) {
      const provider = new ethers.providers.Web3Provider(ethereum);
      const signer = provider.getSigner();
      const wavePortalContract = new ethers.Contract(
        contractAddress,
        contractABI,
        signer
      );

      let count = await wavePortalContract.getTotalWaves();
      console.log("Retrieved total wave count...", count.toNumber());

      /*
       * Execute the actual wave from your smart contract
       */
      const waveTxn = await wavePortalContract.wave(message.value, {
        gasLimit: 300000,
      });
      console.log("Mining...", waveTxn.hash);

      await waveTxn.wait();
      console.log("Mined -- ", waveTxn.hash);

      count = await wavePortalContract.getTotalWaves();
      console.log("Retrieved total wave count...", count.toNumber());
    } else {
      console.log("Ethereum object doesn't exist!");
    }
  } catch (error) {
    console.log(error);
  } finally {
    loadingWave.value = false;
  }
};

const allWaves = ref([]);

const getAllWaves = async () => {
  loading.value = true;
  const { ethereum } = window;

  try {
    if (ethereum) {
      const provider = new ethers.providers.Web3Provider(ethereum);
      const signer = provider.getSigner();
      const wavePortalContract = new ethers.Contract(
        contractAddress,
        contractABI,
        signer
      );
      const waves = await wavePortalContract.getAllWaves();

      const wavesCleaned = waves
        .map((wave) => {
          return {
            address: wave.waver,
            timestamp: new Date(wave.timestamp * 1000),
            message: wave.message,
          };
        })
        .sort((a, b) => new Date(b.timestamp) - new Date(a.timestamp));

      allWaves.value = wavesCleaned;
    } else {
      console.log("Ethereum object doesn't exist!");
    }
  } catch (error) {
    console.log(error);
  }
  loading.value = false;
};

watchEffect(async (onCleanup) => {
  let wavePortalContract;

  const onNewWave = (from, timestamp, message) => {
    console.log("NewWave", from, timestamp, message);
    allWaves.value = [
      ...allWaves.value,
      {
        address: from,
        timestamp: new Date(timestamp * 1000),
        message: message,
      },
    ];
  };

  if (window.ethereum) {
    const provider = new ethers.providers.Web3Provider(window.ethereum);
    const signer = provider.getSigner();

    wavePortalContract = new ethers.Contract(
      contractAddress,
      contractABI,
      signer
    );
    wavePortalContract.on("NewWave", onNewWave);
  }

  onCleanup(() => {
    if (wavePortalContract) {
      wavePortalContract.off("NewWave", onNewWave);
    }
  });
});

onMounted(async () => {
  await findMetaMaskAccount();
  await getAllWaves();
});
</script>
