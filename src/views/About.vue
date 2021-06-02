<template>
  <div class="about">

    <div class="container">
      <div class="row">
        <div class="col-md-12">
          <h1>Web3modal Example</h1>

          <p v-if="!isConnect">钱包未连接</p>
          <p v-if="isConnect">钱包已连接</p>

          <div class="alert alert-danger" id="alert-error-https" style="display: none">
            You can run this example only over HTTPS connection.
          </div>

          <div id="prepare">
            <button class="btn btn-primary" v-if="!isConnect"  id="btn-connect" @click="onConnect">
             连接钱包
            </button>
          </div>

          <div id="connected" v-if="isConnect">

            <button class="btn btn-primary" id="btn-disconnect" @click="onDisconnect" >
              断开连接
            </button>

            <hr>

            <div id="network">
              <p>
              <strong>连接的网络:</strong> <span id="network-name">{{networkName}}</span>
              </p>

              <p>
              <strong>连接的账号:</strong> <span id="selected-account">{{selectAccount}}</span>
              </p>

            </div>

            <hr>

            <h3>所有钱包的余额</h3>

            <table class="table table-listing">
              <thead>
                <th>Address</th>
                <th>ETH balance</th>
              </thead>

              <tbody id="accounts" v-for="item in dataList">
                <th>{{item.address}}</th>
                <th>{{item.balance}}</th>
              </tbody>
            </table>


          </div>

          <br>

          <div class="well">
            <p class="text-muted">See also the <a href="https://web3modal.com/">TypeScript and React example application</a></p>
          </div>

        </div>
      </div>
    </div>
  </div>
</template>



<script>
import Web3 from "web3";
import Web3Modal from "web3modal";
import WalletConnectProvider from "@walletconnect/web3-provider";

// Web3modal instance
let web3Modal
// Chosen wallet provider given by the dialog window
let provider;
// Address of the selected account
let selectedAccount;
// web3
let web3;

const EvmChains = window.evmChains;
const Fortmatic = window.Fortmatic;


export default {
  data() {
    return {
      isConnect: false,
      networkName: "",
      selectAccount: "",
      dataList: [
        {
          address: "",
          balance: ""
        }
      ]

    }
  },
  mounted() {
    // 初始化
    this.init();
  },
  methods: {
    init(){
      // 初始化
      console.log("Initializing example");
      console.log("WalletConnectProvider is", WalletConnectProvider);
      console.log("Fortmatic is", Fortmatic);

      // Tell Web3modal what providers we have available.
      // Built-in web browser provider (only one can exist as a time)
      // like MetaMask, Brave or Opera is added automatically by Web3modal
      const providerOptions = {
        /* See Provider Options Section */
        walletconnect: {
          package: WalletConnectProvider, // required
          options: {
            infuraId: "9321e08afdc04e2c81cabc499dc5d569" // required
          }
        }
      };

       web3Modal = new Web3Modal({
        network: "mainnet", // optional
        cacheProvider: false, // optional
        disableInjectedProvider: false,
        providerOptions // required
      });
    },


    /**
     * Connect wallet button pressed.
     */
    async onConnect() {

      console.log("Opening a dialog", web3Modal);
      try {
        provider = await web3Modal.connect();

        // Get a Web3 instance for the wallet
        web3 = new Web3(provider);

        console.log("provider=====", "connect");
      } catch(e) {
        console.log("Could not get a wallet connection", e);
        return;
      }

      // Subscribe to accounts change
      provider.on("accountsChanged", (accounts) => {
        console.log("accountsChanged=====", "changed");
        this.fetchAccountData();
      });

      // Subscribe to chainId change
      provider.on("chainChanged", (chainId) => {
        this.fetchAccountData();
      });

      // Subscribe to networkId change
      provider.on("networkChanged", (networkId) => {
        //fetchAccountData();
      });

      // Subscribe to provider connection
      provider.on("connect", (info) => {

        console.log("===========", "connect");
        console.log(info);
      });

      // Subscribe to provider disconnection
      provider.on("disconnect", (error) => {
        console.log("===========", "disconnect");
        console.log(error);
        this.onDisconnect();
      });

      await this.refreshAccountData();
    },


    /**
     * Kick in the UI action after Web3modal dialog has chosen a provider
     */
    async fetchAccountData() {


      console.log("Web3 instance is", web3);

      // Get connected chain id from Ethereum node
      const chainId = await web3.eth.getChainId();
      // Load chain information over an HTTP API
      const chainData = await EvmChains.getChain(chainId);
      // 给networkName赋值
      this.networkName = chainData.name;

      // Get list of accounts of the connected wallet
      const accounts = await web3.eth.getAccounts();

      // MetaMask does not give you all accounts, only the selected account
      console.log("Got accounts", accounts);
      selectedAccount = accounts[0];

      // 给选择的账号赋值
      this.selectAccount = selectedAccount;

      // Get a handl
      //const template = document.querySelector("#template-balance");
      //const accountContainer = document.querySelector("#accounts");

      // Purge UI elements any previously loaded accounts
      //accountContainer.innerHTML = '';

      // Go through all accounts and get their ETH balance
      console.log("accounts-----------", accounts);
      const rowResolvers = accounts.map(async (address) => {
        const balance = await web3.eth.getBalance(address);
        // ethBalance is a BigNumber instance
        // https://github.com/indutny/bn.js/
        const ethBalance = web3.utils.fromWei(balance, "ether");
        const humanFriendlyBalance = parseFloat(ethBalance).toFixed(4);
        this.dataList.length = 0;
        this.dataList.push({
          address: address,
          balance: humanFriendlyBalance
        });



        // Fill in the templated row and put in the document
        //const clone = template.content.cloneNode(true);
        //clone.querySelector(".address").textContent = address;
        //clone.querySelector(".balance").textContent = humanFriendlyBalance;
        //accountContainer.appendChild(clone);
      });

      // Because rendering account does its own RPC commucation
      // with Ethereum node, we do not want to display any results
      // until data for all accounts is loaded
      await Promise.all(rowResolvers);

      // Display fully loaded UI for wallet data
      //document.querySelector("#prepare").style.display = "none";
      //document.querySelector("#connected").style.display = "block";
      this.isConnect = true;
    },

    /**
     * Fetch account data for UI when
     * - User switches accounts in wallet
     * - User switches networks in wallet
     * - User connects wallet initially
     */
    async refreshAccountData() {

      // If any current data is displayed when
      // the user is switching acounts in the wallet
      // immediate hide this data
      //document.querySelector("#connected").style.display = "none";
      //document.querySelector("#prepare").style.display = "block";
      this.isConnect = false;

      // Disable button while UI is loading.
      // fetchAccountData() will take a while as it communicates
      // with Ethereum node via JSON-RPC and loads chain data
      // over an API call.
      //document.querySelector("#btn-connect").setAttribute("disabled", "disabled")
      await this.fetchAccountData(provider);
      //document.querySelector("#btn-connect").removeAttribute("disabled")
    },



    /**
     * Disconnect wallet button pressed.
     */
    async onDisconnect() {

      console.log("Killing the wallet connection", provider);

      // TODO: Which providers have close method?
      if (provider != null){
        if(provider.close) {
          await provider.close();

          // If the cached provider is not cleared,
          // WalletConnect will default to the existing session
          // and does not allow to re-scan the QR code with a new wallet.
          // Depending on your use case you may want or want not his behavir.
          await web3Modal.clearCachedProvider();
          provider = null;
          web3 = null;
        }
      }

      selectedAccount = null;
      // 清空数据
      this.dataList.length = 0;

      // Set the UI back to the initial state
      //document.querySelector("#prepare").style.display = "block";
      //document.querySelector("#connected").style.display = "none";
      this.isConnect = false;
    }







  }  // method 结束

}
</script>
