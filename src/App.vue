<template>
  <div id="app">
    <header>
      <img src="@/assets/logo.png" alt="DE-FI-AT Logo" class="logo">
    </header>
    <div class="content">
      <h1 class="main-title">DEFI-AT for Web3 Workers</h1>
      <div v-if="!userSession.isUserSignedIn()" class="connect-wallet">
        <button @click="connectWallet" class="btn btn-primary">Connect Wallet</button>
      </div>
      <div v-else class="user-dashboard">
        <h2 class="welcome-message">Welcome, {{ userAddress }}</h2>
        <div class="how-it-works">
          <h3>How It Works</h3>
          <div class="steps">
            <div class="step">
              <i class="fas fa-link animated-icon"></i>
              <p>Contract Linking</p>
            </div>
            <div class="step">
              <i class="fas fa-check-circle animated-icon"></i>
              <p>Payments Verification</p>
            </div>
            <div class="step">
              <i class="fas fa-handshake animated-icon"></i>
              <p>Financial Partnerships</p>
            </div>
          </div>
        </div>
        <div class="benefits">
          <h3>Benefits</h3>
          <div class="benefit-items">
            <div class="benefit">
              <img src="@/assets/loan.svg" alt="Loans">
              <p>Access to Loans</p>
            </div>
            <div class="benefit">
              <img src="@/assets/house.svg" alt="House">
              <p>Home Purchases</p>
            </div>
            <div class="benefit">
              <img src="@/assets/credit.svg" alt="Credit">
              <p>Credit Access</p>
            </div>
            <div class="benefit">
              <img src="@/assets/car.svg" alt="Car">
              <p>Vehicle Financing</p>
            </div>
          </div>
        </div>
        <div class="contract-creation">
          <h3>Create Contract</h3>
          <form @submit.prevent="createContract" class="contract-form">
            <div class="form-group">
              <label for="file-upload">Upload File</label>
              <input id="file-upload" type="file" @change="handleFileUpload" class="file-input">
            </div>
            <div class="form-group">
              <label for="contract-amount">Contract Amount (in USD)</label>
              <input id="contract-amount" v-model="contractAmount" type="number" placeholder="Enter amount"
                class="amount-input">
            </div>
            <button type="submit" class="btn btn-generate">Create Contract</button>
          </form>
        </div>
        <div class="active-contracts">
          <h3>Active Contracts</h3>
          <ul v-if="contracts.length > 0" class="contract-list">
            <li v-for="contract in contracts" :key="contract.id" class="contract-item">
              <strong>Contract ID:</strong> {{ contract.id }}, <strong>Amount:</strong> {{ contract['contract-amount'] }}
            </li>
          </ul>
          <p v-else class="no-contracts-message">You don't have any active contracts.</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { AppConfig, UserSession, showConnect } from '@stacks/connect';
import {
  makeContractCall,
  stringUtf8CV,
  uintCV,
  standardPrincipalCV
} from '@stacks/transactions';
import { StacksTestnet } from '@stacks/network';
import { BigNum } from '@stacks/common';  

const appConfig = new AppConfig(['store_write', 'publish_data', 'read_contract']);
const userSession = new UserSession({ appConfig });

export default {
  name: 'App',
  data() {
    return {
      userSession,
      userAddress: '',
      privateKey: '',
      contractFile: null,
      contractAmount: 0,
      contracts: []
    }
  },
  methods: {
    connectWallet() {
      showConnect({
        appDetails: {
          name: 'Income Certificates App',
          icon: window.location.origin + '/logo.png',
        },
        redirectTo: '/',
        onFinish: () => {
          const userData = userSession.loadUserData();
          this.userAddress = userData.profile.stxAddress.testnet;
          this.privateKey = userData.appPrivateKey;
          this.fetchContracts();
        },
      });
    },
    handleFileUpload(event) {
      this.contractFile = event.target.files[0];
    },
    async createContract() {
      if (!this.contractFile || !this.contractAmount) {
        alert('Please provide both a file and an amount');
        return;
      }

      const fileReader = new FileReader();
      fileReader.onload = async (e) => {
        let fileContent = e.target.result;

        // Ensure fileContent is a string and truncate/pad to 256 characters
        fileContent = String(fileContent).slice(0, 256).padEnd(256, ' ');
        let number = new BigNum(this.contractAmount)
        console.log('File content:', stringUtf8CV(fileContent));
        console.log('Contract amount:', this.contractAmount);
        console.log('Private key:', this.privateKey);
        console.log('User address:', this.userAddress);
        console.log('Number:', number);
        try {
          const txOptions = {
            contractAddress: 'ST239W49AXK04H8G5JTPV9W2YK8JQZMNYBHVEZ35C',
            contractName: 'income-certificates-contract-v1',
            functionName: 'upload-contract',
            functionArgs: [
              stringUtf8CV(fileContent),
              number
            ],
            senderKey: this.privateKey,
            validateWithAbi: true,    
            network: new StacksTestnet()
          };

          const transaction = await makeContractCall(txOptions);
          console.log('Transaction:', transaction);
          this.fetchContracts();
        } catch (error) {
          console.error('Failed to create contract:', error);
        }
      };
      fileReader.readAsText(this.contractFile);
    },
    async fetchContracts() {
      console.log('Fetching contracts for user:', this.userAddress);
      
      if (!this.privateKey) {
        console.error('Private key not available');
        return;
      }

      try {
        const contractCountResponse = await makeContractCall({
          contractAddress: 'ST239W49AXK04H8G5JTPV9W2YK8JQZMNYBHVEZ35C',
          contractName: 'income-certificates-contract-v1',
          functionName: 'get-user-contract-count',
          functionArgs: [standardPrincipalCV(this.userAddress)],
          senderAddress: this.userAddress,
          validateWithAbi: true,
          network: new StacksTestnet(),
        });

        console.log('Contract Count Response:', contractCountResponse?.value); // Log the response

        if (contractCountResponse?.value == 0) {
          console.error("Invalid contract count response");
          return;
        }
        let contractCount = contractCountResponse?.value;
        
        this.contracts = [];
        for (let i = 1; i <= contractCount; i++) {
          const contractResponse = await makeContractCall({
            contractAddress: 'ST239W49AXK04H8G5JTPV9W2YK8JQZMNYBHVEZ35C',
            contractName: 'income-certificates-contract-v1',
            functionName: 'get-user-contract',
            functionArgs: [standardPrincipalCV(this.userAddress), uintCV(i)],
            senderAddress: this.userAddress,
            validateWithAbi: true,
            network: new StacksTestnet(),
          });

          console.log('Contract Response:', contractResponse?.value);

          if (!contractResponse || !contractResponse.value) {
            console.error('Invalid contract response:', contractResponse);
            continue; // Skip this iteration if the contract is not found
          }

          const contract = contractResponse.value;
          this.contracts.push({ id: i, ...contract });
        }
      } catch (error) {
        console.error('Failed to fetch contracts:', error);
      }
    }
  },
  mounted() {
    if (userSession.isUserSignedIn()) {
      this.userAddress = userSession.loadUserData().profile.stxAddress.testnet;
      this.fetchContracts();
    }
  }
}
</script>

<style scoped>
#app {
  font-family: 'Arial', sans-serif;
  background: linear-gradient(135deg, #7B68EE, #FFD700);
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  color: #000000;
}

header {
  width: 100%;
  padding: 1rem;
  background-color: rgba(0, 0, 0, 0.8);
  display: flex;
  justify-content: center;
  align-items: center;
}

.logo {
  height: 80px; /* Increase this value to make the logo bigger */
  width: auto; /* This maintains the aspect ratio */
  max-width: 100%; /* Ensures the logo doesn't overflow on smaller screens */
}

.content {
  background: rgba(255, 255, 255, 0.9);
  padding: 2rem;
  border-radius: 10px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
  max-width: 800px;
  width: 100%;
  margin-top: 2rem;
}

h1, h2, h3 {
  color: #7B68EE;
  margin-bottom: 1rem;
}

.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.btn-primary {
  background-color: #7B68EE;
  color: white;
}

.btn-generate {
  background-color: #FFD700;
  color: #000000;
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0% {
    box-shadow: 0 0 0 0 rgba(255, 215, 0, 0.7);
  }
  70% {
    box-shadow: 0 0 0 10px rgba(255, 215, 0, 0);
  }
  100% {
    box-shadow: 0 0 0 0 rgba(255, 215, 0, 0);
  }
}

.how-it-works, .benefits {
  margin-bottom: 2rem;
}

.steps, .benefit-items {
  display: flex;
  justify-content: space-around;
  margin-top: 1rem;
}

.step, .benefit {
  text-align: center;
}

.animated-icon {
  font-size: 2rem;
  color: #7B68EE;
  animation: bounce 2s infinite;
}

@keyframes bounce {
  0%, 20%, 50%, 80%, 100% {
    transform: translateY(0);
  }
  40% {
    transform: translateY(-30px);
  }
  60% {
    transform: translateY(-15px);
  }
}

.benefit img {
  width: 64px;
  height: 64px;
  margin-bottom: 0.5rem;
}

.form-group {
  margin-bottom: 1rem;
}

.file-input, .amount-input {
  width: 100%;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.contract-list {
  list-style-type: none;
  padding: 0;
}

.contract-item {
  background-color: #f0f0f0;
  margin-bottom: 10px;
  padding: 15px;
  border-radius: 5px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.welcome-message {
  color: #000000; /* This sets the text color to black */
  font-weight: bold; /* This makes the text bold */
  margin-bottom: 1rem; /* This adds some space below the message */
}

.no-contracts-message {
  text-align: center;
  padding: 1rem;
  background-color: #f0f0f0;
  border-radius: 5px;
  color: #7B68EE;
  font-style: italic;
}

.connect-wallet {
  display: flex;
  justify-content: center;
  margin-top: 2rem;
}

.btn-primary {
  background-color: #7B68EE;
  color: white;
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 1.1em;
  transition: background-color 0.3s ease;
}

.btn-primary:hover {
  background-color: #6A5ACD;
}

.main-title {
  text-align: center;
  color: #7B68EE; /* This uses the purple color from your logo */
  font-size: 2.5rem; /* Adjust this value to make the title larger or smaller */
  margin-bottom: 2rem; /* This adds some space below the title */
  font-weight: bold;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.1); /* This adds a subtle shadow for depth */
}
</style>
