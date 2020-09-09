<template>
  <v-app>
    <v-app-bar app light>
      <p class="primary--text title pt-4">Binance Smart Chain</p>

      <v-spacer></v-spacer>

      <template v-if="address">
        <span
          class="mr-2 subtitle-2"
        >{{address.substring(0,8) + "..." + address.substring(address.length-6,address.length)}}</span>
        <v-icon>mdi-account</v-icon>
      </template>
      <v-btn @click="login()" text v-else>
        <span class="mr-2">LOG IN</span>
        <v-icon>mdi-account</v-icon>
      </v-btn>
    </v-app-bar>

    <v-content class="px-3 my-3">
      <v-card class="mx-auto">
        <v-card-text class="mx-auto">
          <v-select
            v-model="selectToken"
            label="Select a Token"
            :items="tokens"
            item-text="symbol"
            item-value="symbol"
            class="mb-3"
            return-object
            dense
            filled
          ></v-select>
          <v-tabs v-model="selectTab" background-color="transparent" color="amber">
            <v-tab>{{ tabs[0] }}</v-tab>
            <v-tab v-if="!isBNBToken()">{{ tabs[1] }}</v-tab>
          </v-tabs>
          <v-tabs-items v-model="selectTab">
            <v-tab-item>
              <v-card flat>
                <v-card-text>
                  <v-text-field
                    v-model="amount"
                    class="subtitle-1 font-weight-medium"
                    label="Amount"
                    :suffix="selectToken.symbol"
                    :hint="getTokenBalanceMessage()"
                    persistent-hint
                    placeholder="0.0"
                    filled
                  ></v-text-field>
                  <v-text-field
                    v-model="recipient"
                    class="subtitle-1 font-weight-medium"
                    label="Recipient"
                    :hint="recipientHint"
                    persistent-hint
                    placeholder="bnbxxxxxxxxxxxxx..."
                    append-icon="mdi-chevron-down"
                    @click:append="walletPicker"
                    filled
                  ></v-text-field>
                  <v-input
                    class="primary--text body-2"
                    :hint="feeMessage"
                    persistent-hint
                    disabled
                    dense
                  >Fee</v-input>
                </v-card-text>
                <v-card-actions>
                  <v-btn
                    class="error px-6"
                    @click="sendTransaction()"
                    large
                    :disabled="!(web3 && amount.length && recipientHex.length)"
                  >Transfer</v-btn>
                </v-card-actions>
              </v-card>
            </v-tab-item>
            <v-tab-item>
              <v-card flat>
                <v-card-title class="subtitle-2 grey--text">Approved</v-card-title>
                <v-card-subtitle
                  class="primary--text display-1 font-weight-medium"
                >{{allowance?`${allowance} ${selectToken.symbol}`:""}}</v-card-subtitle>
                <v-card-text>
                  <v-text-field
                    v-model="amount"
                    class="subtitle-1 font-weight-medium"
                    label="Amount"
                    :suffix="selectToken.symbol"
                    :hint="getTokenBalanceMessage(true)"
                    persistent-hint
                    placeholder="0.0"
                    filled
                  ></v-text-field>

                  <v-input
                    class="primary--text body-2"
                    :hint="feeMessage"
                    persistent-hint
                    disabled
                    dense
                  >Fee</v-input>
                </v-card-text>
                <v-card-actions>
                  <v-btn
                    class="error px-6"
                    @click="apporveTransaction()"
                    large
                    :disabled="!(web3 && amount.length)"
                  >Approve</v-btn>
                </v-card-actions>
              </v-card>
            </v-tab-item>
          </v-tabs-items>
        </v-card-text>
      </v-card>
    </v-content>
    <v-snackbar
      :color="snackbarColor"
      v-model="showSnackbar"
      :timeout="snackbarTimeout"
      top
    >{{ snackbarText }}</v-snackbar>
  </v-app>
</template>

<script>
import BSCABI from "./abis/abi";
import Web3 from "web3";

import BN from "bn.js";
import * as Bech32 from "bech32";
import CryptoUtils from "./utils/CryptoUtils";
import { parseAmount, formatAmount } from "./utils/Format";
import mathwallet from "math-js-sdk";

export default {
  name: "App",
  data: () => ({
    showSnackbar: false,
    snackbarTimeout: 2000,
    snackbarColor: "error",
    snackbarText: "",
    feeMessage: "",
    // Tabs
    tabs: ["Transfer", "Approve"],
    selectTab: null,
    // Injected Web3
    web3Handler: null,
    injectedWeb3: null,
    web3: null,
    // Address
    address: "",
    // Token Hub
    tokenHubContractAddress: "0x0000000000000000000000000000000000001004",
    // Tokens
    tokens: [
      {
        symbol: "BNB",
        contract: "0x0000000000000000000000000000000000000000",
        decimals: 18,
      },
      {
        symbol: "BTCB",
        contract: "0x7130d2A12B9BCbFAe4f2634d864A1Ee1Ce3Ead9c",
        decimals: 18,
      },
      {
        symbol: "BUSD",
        contract: "0xe9e7CEA3DedcA5984780Bafc599bD69ADd087D56",
        decimals: 18,
      },
    ],
    selectToken: null,
    // Balance
    balance: "", // BNB
    tokenBalance: "", // BTC\ETH\XRP
    // Allowance
    allowance: "",
    // Recipient
    recipient: "",
    recipientHex: "",
    recipientHint: "",
    // Transfer Amount
    amount: "", // unit(BNB，18)
    // Relay fee
    relayFee: "2000000000000000", /// unit(BNB，18)
    // Gas fee
    gasLimit: "200000",
    gasPrice: "20000000000",
  }),
  watch: {
    recipient(newRecipient, oldRecipient) {
      try {
        const { prefix, words } = Bech32.decode(newRecipient);
        if (prefix == "bnb" || prefix == "tbnb") {
          const addressHex =
            "0x" + CryptoUtils.bytesToHex(Bech32.fromWords(words));
          this.recipientHint = "↳" + addressHex;
          this.recipientHex = addressHex;
        } else {
          this.recipientHint = newRecipient.length ? "Invalid Address" : "";
          this.recipientHex = "";
        }
      } catch (e) {
        this.recipientHint = newRecipient.length ? "Invalid Address" : "";
        this.recipientHex = "";
      }
    },
    selectToken(newSelectToken) {
      // Reset Data
      this.amount = "";
      this.recipient = "";
      this.allowance = "";
      this.balance = "";
      this.tokenBalance = "";
      this.selectTab = this.tabs[0];

      // Get Data
      this.setUIOfTokenData();
    },
    selectTab(newTab) {
      this.amount = "";
      this.recipient = "";
    },
  },
  created() {
    this.selectToken = this.tokens[0];
    this.selectTab = this.tabs[0];
  },
  mounted() {
    this.web3Handler = setInterval(() => {
      if (window.web3 || window.ethereum) {
        clearInterval(this.web3Handler);
        this.login();
      }
    }, 500);

    setInterval(() => {
      this.setUIOfTokenData();
    }, 5000);
  },
  methods: {
    walletPicker() {
      if (mathwallet.isMath()) {
        mathwallet.walletPicker("BINANCE").then((account) => {
          this.recipient = account.address;
        });
      }
    },
    isBNBToken() {
      return this.selectToken.symbol == this.tokens[0].symbol;
    },
    getTokenBalanceMessage(approve = false) {
      if (this.isBNBToken()) {
        return this.balance
          ? `↳Balance ${this.balance} ${this.selectToken.symbol}`
          : "";
      } else if (approve) {
        return this.tokenBalance
          ? `↳Balance ${this.tokenBalance} ${this.selectToken.symbol}`
          : "";
      } else {
        return this.tokenBalance
          ? `↳Available ${this.allowance} ${this.selectToken.symbol}, Total ${this.tokenBalance} ${this.selectToken.symbol}`
          : "";
      }
    },
    login() {
      this.injectedWeb3 = window.web3;
      // Log in
      this.injectedWeb3.currentProvider.enable().then((accounts) => {
        this.address = accounts[0];
        this.web3 = new Web3(this.injectedWeb3.currentProvider);
      });
    },
    setUIOfTokenData() {
      // Fee
      const relayFee = new BN(this.relayFee);
      const gasPrice = new BN(this.gasPrice);
      const gasLimit = new BN(this.gasLimit);
      this.feeMessage = `${formatAmount(
        gasPrice.mul(gasLimit).add(relayFee).toString(),
        this.tokens[0].decimals
      )} ${this.tokens[0].symbol}`;
      // Check if Web3 is successfully injected
      if (!this.web3) return;
      // Balance of BNB
      this.web3.eth.getBalance(this.address).then((result) => {
        this.balance = formatAmount(result, this.tokens[0].decimals);
      });
      if (this.isBNBToken()) return;
      // Balance of BEP2E Token
      const contract = new this.web3.eth.Contract(
        new BSCABI().BEP2E,
        this.selectToken.contract
      );
      const balanceTransaction = {
        from: this.address,
        to: this.selectToken.contract,
        data: contract.methods.balanceOf(this.address).encodeABI(),
      };
      this.web3.eth.call(balanceTransaction, (error, data) => {
        if (!error) {
          this.tokenBalance = formatAmount(
            new BN(CryptoUtils.hexToBytes(data), "hex").toString(),
            this.selectToken.decimals
          );
        }
      });
      // Approved balance of BEP2E Token
      const allowanceTransaction = {
        from: this.address,
        to: this.selectToken.contract,
        data: contract.methods
          .allowance(this.address, this.tokenHubContractAddress)
          .encodeABI(),
      };
      this.web3.eth.call(allowanceTransaction, (error, data) => {
        if (!error) {
          this.allowance = formatAmount(
            new BN(CryptoUtils.hexToBytes(data), "hex").toString(),
            this.selectToken.decimals
          );
        }
      });
    },
    sendTransaction() {
      const amountBN = new BN(
        parseAmount(this.amount, this.selectToken.decimals)
      );
      const relayFeeBN = new BN(this.relayFee);

      // Check if fee is sufficient
      if (!this.isBNBToken()) {
        const allowanceBalance = new BN(
          parseAmount(
            this.allowance.length ? this.allowance : "0",
            this.selectToken.decimals
          )
        );
        if (allowanceBalance.cmp(amountBN) == -1) {
          this.showSnackbar = true;
          this.snackbarColor = "error";
          this.snackbarText =
            this.selectToken.symbol + " allowance insufficient";
          return;
        }
      }
      const value = this.isBNBToken() ? amountBN.add(relayFeeBN) : relayFeeBN; // BNB->(txValue = amount + relayFeeBN),Other->->(txValue = relayFeeBN)
      const gasLimit = new BN(this.gasLimit);
      const gasPrice = new BN(this.gasPrice);

      const tolal = gasLimit.mul(gasPrice).add(value);
      const balance = new BN(
        parseAmount(
          this.balance.length ? this.balance : "0",
          this.tokens[0].decimals
        )
      );
      if (balance.cmp(tolal) == -1) {
        this.showSnackbar = true;
        this.snackbarColor = "error";
        this.snackbarText = this.tokens[0].symbol + " balance insufficient";
        return;
      }
      // Transaction
      const contract = new this.web3.eth.Contract(
        new BSCABI().TOKEN_HUB,
        this.tokenHubContractAddress
      );
      const encodeABI = contract.methods
        .transferOut(
          this.selectToken.contract, // BEP2E contract address
          this.recipientHex, // decode bech32 address to hex string
          amountBN.toString(), // BEP2E token amount
          String(Date.parse(new Date()) / 1000 + 60 * 60) // expireTime	uint256	Timestamp, counted by second
        )
        .encodeABI();
      const transaction = {
        from: this.address,
        to: this.tokenHubContractAddress,
        value: value.toString(),
        gas: gasLimit.toString(),
        gasPrice: gasPrice.toString(),
        data: encodeABI,
      };
      console.log(transaction);
      this.web3.eth.sendTransaction(transaction, (error, hash) => {
        console.log(error, hash);
        if (error || !(hash && typeof hash == "string")) {
          this.showSnackbar = true;
          this.snackbarColor = "error";
          this.snackbarText = error
            ? error.message
            : "Failed to send transaction";
        } else {
          this.showSnackbar = true;
          this.snackbarColor = "success";
          this.snackbarText = "Successfully";
        }
      });
    },
    apporveTransaction() {
      const amountBN = new BN(
        parseAmount(this.amount, this.selectToken.decimals)
      );

      // Check if fee is sufficient
      if (!this.isBNBToken()) {
        const tokenBalance = new BN(
          parseAmount(
            this.tokenBalance.length ? this.tokenBalance : "0",
            this.selectToken.decimals
          )
        );
        if (tokenBalance.cmp(amountBN) == -1) {
          this.showSnackbar = true;
          this.snackbarColor = "error";
          this.snackbarText = this.selectToken.symbol + " balance insufficient";
          return;
        }
      }
      const gasLimit = new BN(this.gasLimit);
      const gasPrice = new BN(this.gasPrice);
      const fee = gasLimit.mul(gasPrice);
      const balance = new BN(
        parseAmount(
          this.balance.length ? this.balance : "0",
          this.tokens[0].decimals
        )
      );
      if (balance.cmp(fee) == -1) {
        this.showSnackbar = true;
        this.snackbarColor = "error";
        this.snackbarText = this.tokens[0].symbol + " balance insufficient";
        return;
      }

      // Transaction
      const contract = new this.web3.eth.Contract(
        new BSCABI().BEP2E,
        this.selectToken.contract
      );
      const encodeABI = contract.methods
        .approve(this.tokenHubContractAddress, amountBN.toString())
        .encodeABI();
      const transaction = {
        from: this.address,
        to: this.selectToken.contract,
        gas: gasLimit.toString(),
        gasPrice: gasPrice.toString(),
        data: encodeABI,
      };

      this.web3.eth.sendTransaction(transaction, (error, hash) => {
        if (error) {
          this.showSnackbar = true;
          this.snackbarColor = "error";
          this.snackbarText = error.message;
        } else {
          console.log(hash);

          this.showSnackbar = true;
          this.snackbarColor = "success";
          this.snackbarText = "Successfully";
        }
      });
    },
  },
};
</script>
