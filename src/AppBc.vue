<template>
  <v-app>
    <v-app-bar app light>
      <p class="primary--text title pt-4">Binance Chain</p>

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
          <v-text-field
            v-model="amount"
            class="subtitle-1 font-weight-medium"
            label="Amount"
            :suffix="selectToken.symbol"
            :hint="`${balance} BNB`"
            persistent-hint
            placeholder="0.0"
            filled
          ></v-text-field>
          <v-text-field
            v-model="recipient"
            class="subtitle-1 font-weight-medium"
            label="Recipient"
            placeholder="0x55da3246865921072b14f2bca5cf55f4ae017cde"
            filled
          ></v-text-field>
        </v-card-text>
        <v-card-actions>
          <v-btn
            class="error px-6"
            @click="sendTransaction()"
            large
            :disabled="!(address && amount.length)"
          >Transfer</v-btn>
        </v-card-actions>
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
import BN from "bn.js";
import { parseAmount, formatAmount } from "./utils/Format";
export default {
  name: "Bc",
  data: () => ({
    showSnackbar: false,
    snackbarTimeout: 2000,
    snackbarColor: "error",
    snackbarText: "",
    address: "",
    mathExtensionHandler: null,
    // Tokens
    tokens: [
      {
        symbol: "BNB",
        decimals: 8,
      },
    ],
    selectToken: null,
    // Balance
    balance: "", // BNB
    // Recipient
    recipient: "",
    // Transfer Amount
    amount: "", // unit(BNB，8)
    // Http Provider
    httpProvider: null,
    rpcUrl: "https://dex.binance.org",
  }),
  created() {
    this.selectToken = this.tokens[0];
  },
  mounted() {
    this.mathExtensionHandler = setInterval(() => {
      if (window.mathExtension) {
        clearInterval(this.mathExtensionHandler);
        this.login();
      }
    }, 500);
  },
  methods: {
    login() {
      window.mathExtension
        .getIdentity({
          blockchain: "binance",
          chainId: "Binance-Chain-Tigris",
        })
        .then((identity) => {
          this.address = identity.account;
          this.httpProvider = window.mathExtension.httpProvider(this.rpcUrl);
          this.getBalance();
        });
    },
    getBalance() {
      this.httpProvider
        .get(`/api/v1/account/${this.address}`)
        .then((response) => {
          const balances = response.result.balances;
          if (balances && balances.length > 0) {
            const b = balances.find(
              (balance) => balance.symbol === this.selectToken.symbol
            );
            this.balance = b ? b.free : "0";
          } else {
            this.balance = "0";
          }
        });
    },
    sendTransaction() {
      const amountBN = new BN(
        parseAmount(
          this.amount.length ? this.amount : "0",
          this.selectToken.decimals
        )
      );
      const balanceBN = new BN(
        parseAmount(
          this.balance.length ? this.balance : "0",
          this.selectToken.decimals
        )
      );
      if (balanceBN.cmp(amountBN) == -1) {
        this.showSnackbar = true;
        this.snackbarColor = "error";
        this.snackbarText = this.tokens[0].symbol + " balance insufficient";
        return;
      }
      this.sendTransactionAsync(
        this.address,
        this.recipient,
        this.amount,
        this.selectToken
      )
        .then((a) => {
          this.showSnackbar = true;
          this.snackbarColor = "success";
          this.snackbarText = "success";
        })
        .catch((error) => {
          this.showSnackbar = true;
          this.snackbarColor = "error";
          this.snackbarText = error.message || "Unknow error";
        });
    },
    async sendTransactionAsync(from, to, amount, selectToken) {
      const account = await this.httpProvider.get(
        `/api/v1/account/${this.address}`
      );
      const node = await this.httpProvider.get(`/api/v1/node-info`);

      const sequence = account.result && account.result.sequence;
      const accountNumber = account.result && account.result.account_number;
      const chainId = node.result.node_info.network;
      //  TransferOutMsg
      var transaction = {
        chain_id: chainId,
        account_number: accountNumber,
        sequence: sequence,
        memo: "",
        type: "TransferOutMsg",
        msg: {
          from,
          to,
          amount: { denom: selectToken.symbol, amount: amount },
          expire_time: Date.parse(new Date()) / 1000 + 60 * 3,
        },
      };
      const signTransaction = await window.mathExtension.requestSignature(
        transaction
      );
      console.log(signTransaction, result);
      const opts = {
        data: signTransaction.tx,
        headers: {
          "Content-Type": "text/plain",
        },
      };
      const result = await this.httpProvider.post(
        "/api/v1/broadcast?sync=true",
        null,
        opts
      );
      return result;
    },
  },
};
</script>
