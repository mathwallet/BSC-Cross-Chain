<template>
  <v-app>
    <v-app-bar app light>
      <h3 class="primary--text">Binance Chain</h3>

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
        <v-card-text class="mx-auto" v-if="selectToken">
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
            :hint="`${tokenBalance} ${selectToken.symbol}`"
            persistent-hint
            placeholder="0.0"
            filled
          ></v-text-field>
          <v-text-field
            v-model="recipient"
            class="subtitle-1 font-weight-medium"
            label="Recipient"
            placeholder="0x0000000000000000..."
            append-icon="mdi-chevron-down"
            @click:append="walletPicker"
            filled
          ></v-text-field>

          <v-input
            class="primary--text body-2"
            :hint="`${getFeeText()} ${tokens[0].symbol}`"
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
import Web3 from "web3";
import mathwallet from "math-js-sdk";

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
    bnbBalance: "0", // BNB
    tokenBalance: "0", // Select Token
    // Recipient
    recipient: "",
    // Transfer Amount
    amount: "", // unit(BNB，8)
    // Http Provider
    httpProvider: null,
    rpcUrl: "https://dex.binance.org",
    // Fee
    fee: "400000", // 0.004 BNB
  }),
  created() {
    this.selectToken = this.tokens[0];
    this.getTokenList();
  },
  watch: {
    selectToken(newSelectToken) {
      // Reset Data
      this.amount = "";
      this.recipient = "";
      this.bnbBalance = "0";
      this.tokenBalance = "0";

      // Get Data
      if (this.address) {
        this.getBalance();
      }
    },
  },
  mounted() {
    this.mathExtensionHandler = setInterval(() => {
      if (window.mathExtension) {
        clearInterval(this.mathExtensionHandler);
        this.login();
      }
    }, 500);

    setInterval(() => {
      if (this.address) {
        this.getBalance();
      }
    }, 5000);
  },
  methods: {
    getTokenList() {
      this.axios
        .get("https://market.maiziqianbao.net/api/getCrossChainTokens?type=14")
        .then((res) => {
          let tokens = res.data.data;
          if (!tokens || tokens.length == 0) {
            return;
          }

          this.tokens.push(
            ...tokens.filter((v) => {
              return v.symbol != this.tokens[0].symbol;
            })
          );
        })
        .catch(() => {});
    },
    walletPicker() {
      if (mathwallet.isMath()) {
        mathwallet.walletPicker("BSC").then((account) => {
          this.recipient = account.address;
        });
      }
    },
    login() {
      window.mathExtension
        .getIdentity({
          blockchain: "binance",
          chainId: "Binance-Chain-Tigris",
        })
        .then(
          (identity) => {
            this.address = identity.account;
            this.httpProvider = window.mathExtension.httpProvider(this.rpcUrl);
            this.getBalance();
          },
          (err) => {
            console.log(err);
          }
        );
    },
    getFeeText() {
      return formatAmount(this.fee, this.tokens[0].decimals);
    },
    getBalance() {
      this.httpProvider
        .get(`/api/v1/account/${this.address}`)
        .then((response) => {
          const balances = response.result.balances;
          if (balances && balances.length > 0) {
            balances.forEach((b) => {
              if (b.symbol === this.selectToken.symbol) {
                this.tokenBalance = b.free;
              } else if (b.symbol === this.tokens[0].symbol) {
                this.bnbBalance = b.free;
              }
            });
          } else {
            this.tokenBalance = "0";
            this.bnbBalance = "0";
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
      const feeBN = new BN(this.fee);
      const bnbBalanceBN = new BN(
        parseAmount(
          this.bnbBalance.length ? this.bnbBalance : "0",
          this.tokens[0].decimals
        )
      );
      const tokenBalanceBN = new BN(
        parseAmount(
          this.tokenBalance.length ? this.tokenBalance : "0",
          this.selectToken.decimals
        )
      );
      // BNB
      if (this.selectToken.symbol == this.tokens[0].symbol) {
        if (tokenBalanceBN.cmp(amountBN.add(feeBN)) == -1) {
          this.showSnackbar = true;
          this.snackbarColor = "error";
          this.snackbarText = this.selectToken.symbol + " balance insufficient";
          return;
        }
      } else {
        if (bnbBalanceBN.cmp(feeBN) == -1) {
          this.showSnackbar = true;
          this.snackbarColor = "error";
          this.snackbarText = this.tokens[0].symbol + " balance insufficient";
          return;
        }
        if (tokenBalanceBN.cmp(amountBN) == -1) {
          this.showSnackbar = true;
          this.snackbarColor = "error";
          this.snackbarText = this.selectToken.symbol + " balance insufficient";
          return;
        }
      }
      var to;
      try {
        to = Web3.utils.toChecksumAddress(this.recipient.toLowerCase());
      } catch (error) {
        to = null;
      }
      if (!to) {
        this.showSnackbar = true;
        this.snackbarColor = "error";
        this.snackbarText = "Invalid address";
        return;
      }
      this.sendTransactionAsync(this.address, to, this.amount, this.selectToken)
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
