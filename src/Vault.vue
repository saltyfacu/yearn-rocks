<template lang="pug">
#vault(v-if="isDrizzleInitialized")
  v-app
    v-app-bar(app) {{ config.TITLE }}
    v-main
      v-container(fluid)
        div.columns
          v-alert(type="warning" dense border="left") These vaults are experimental. They are extremely risky and will probably be discarded when production ones are deployed
        div.columns
                div.column.is-half
                        info-message(:status="config.VAULT_STATUS")
        v-card
          v-card-title Vault
          v-card-text
            div Version: {{ vault_version }}
            div Performance Fee: {{ vault_perfFee }}%
            div {{ config.WANT_SYMBOL }} price (CoinGecko 🦎): {{ want_price | toCurrency(4) }}
            div Deposit Limit: {{ vault_deposit_limit | fromWei(2, vault_decimals) }}        {{ config.WANT_SYMBOL }}
            div Total Assets: {{ vault_total_assets | fromWei(2, vault_decimals) }}        {{ config.WANT_SYMBOL }}
            div Total AUM: {{ vault_total_aum | toCurrency(2, vault_decimals) }}
            div.spacer
            div Price Per Share: {{ vault_price_per_share | fromWei(8, vault_decimals) }}
            div Available limit: {{ vault_available_limit | fromWei(2, vault_decimals) }} {{ config.WANT_SYMBOL }}
            v-progress-linear(:value="progress_limit" height="20px") {{ Math.round(progress_limit) }}%
            v-card-actions
              v-btn <a :href="'https://etherscan.io/address/' + config.VAULT_ADDR + '#code'" target="_blank"> 📃Contract </a>
        div.spacer
        v-card
          v-card-title Strategies
          v-card-text
            div(v-for="(strategy, index) in strategies")
                div Strat. {{ index }}: {{ strategy.name }}
                    v-card-actions
                        v-btn <a :href="'https://etherscan.io/address/' + strategy.address + '#code'" target="_blank"> 📃Contract </a>
        div.spacer
        v-card
          v-card-title Wallet
          v-card-text
            div Your Account: {{ username || activeAccount }}
            div Your Vault shares: {{ yvtoken_balance | fromWei(2, vault_decimals) }}
            div Your {{ config.WANT_SYMBOL }} Balance: {{ want_balance | fromWei(2, vault_decimals) }}
            div Your ETH Balance: {{ eth_balance | fromWei(2) }}
        div.spacer

        div(v-if="is_guest || yfi_needed <= 0")
                span <h1>You are a guest. Welcome to the Citadel 🏰</h1>
                div.spacer
                v-card
                  v-card-title Manange
                  v-card-text
                    div.spacer(v-if="want_balance > 0")
                        v-text-field(v-if="want_balance > 0",v-model.number="amount", size="is-small", type="number",min=0,label="Deposit Amount")
                    span(v-if="vault_available_limit <= 0") Deposits closed.
                    div.spacer(v-if="want_balance > 0")
                    v-btn(
                            v-if="vault_available_limit > 0",
                            :disabled="has_allowance_vault",
                            @click.prevent="on_approve_vault"
                    ) {{ has_allowance_vault ? '✅ Approved' : '🚀 Approve Vault' }}
                    v-btn(
                            v-if="vault_available_limit > 0",
                            :disabled="!has_allowance_vault",
                            @click.prevent="on_deposit"
                    ) 🏦 Deposit
                    v-btn(
                            v-if="vault_available_limit > 0",
                            :disabled="!has_allowance_vault",
                            @click.prevent="on_deposit_all"
                    ) 🏦 Deposit All
                    v-btn(:disabled="!has_yvtoken_balance", @click.prevent="on_withdraw_all") 💸 Withdraw All
        div(v-else)
                .red
                        span ⛔ You need {{ yfi_needed | fromWei(4) }} YFI more to enter the Citadel ⛔
                <div v-konami @konami="bribe_unlocked = !bribe_unlocked"></div>
                div(v-if="bribe_unlocked")
                        span If you still want to join the party...
                        |
                        v-btn(v-if="has_allowance_bribe", @click.prevent="on_bribe_the_bouncer") 💰 Bribe the bouncer with ({{ bribe_cost | fromWei(3) }} YFI)
                        v-btn(v-if="!has_allowance_bribe", @click.prevent="on_approve_bribe") 🚀 Approve Bribe
                div(v-else)
                        span Remember Konami 🎮
        .red(v-if="error")
                span {{ error }}
        div.spacer
                .muted
                        span Made with 💙
                        | 
                        span yVault:
                        | 
                        a(:href="'https://twitter.com/' + config.VAULT_DEV", target="_blank") {{ config.VAULT_DEV }}
                        | 
                        span - Guest List:
                        | 
                        a(href="https://twitter.com/bantg", target="_blank") bantg
                        | 
                        span - UI:
                        | 
                        a(href="https://twitter.com/fameal", target="_blank") fameal
                        |, 
                        a(href="https://twitter.com/emilianobonassi", target="_blank") emiliano
                        |, 
                        a(href="https://twitter.com/akshaynexust", target="_blank") akshaynexust
div(v-else)
        div Loading yApp...
</template>

<script>

import { mapGetters } from "vuex";
import ethers from "ethers";
import axios from "axios";
import ProgressBar from './components/ProgressBar';
import InfoMessage from './components/InfoMessage';
import GuestList from "./abi/GuestList.json";
import yVaultV2 from "./abi/yVaultV2.json";
import yStrategy from "./abi/yStrategy.json";
import ERC20 from "./abi/ERC20.json";

import Web3 from "web3";

let web3 = new Web3(Web3.givenProvider);

const max_uint = new ethers.BigNumber.from(2).pow(256).sub(1).toString();
const BN_ZERO = new ethers.BigNumber.from(0);
const ADDRESS_ZERO = "0x0000000000000000000000000000000000000000";

const ERROR_NEGATIVE = "You have to deposit a positive number of tokens 🐀";
const ERROR_NEGATIVE_ALL = "You don't have tokens to deposit 🐀";
const ERROR_NEGATIVE_WITHDRAW = "You don't have any vault shares";
const ERROR_GUEST_LIMIT = "That would exceed your guest limit. Try less.";
const ERROR_GUEST_LIMIT_ALL =
  "That would exceed your guest limit. Try not doing all in.";

export default {
  name: "Vault",
  components: {
    ProgressBar,
  },
  props: ['config'],
  data() {
    return {
      username: null,
      want_price: 0,
      amount: 0,
      amount_wrap: 0,
      strategies: [],
      strategies_balance: 0,
      average_price: 0,
      error: null,
      contractGuestList: null,
      is_guest: false,
      entrance_cost: new ethers.BigNumber.from("1"),
      total_yfi: new ethers.BigNumber.from("0"),
      bribe_unlocked: false,
      bribe_cost: new ethers.BigNumber.from("0"),
      vault_activation: 0,
      roi_week: 0,
    };
  },
  filters: {
    fromWei(data, precision, decimals) {
      if (decimals === undefined) decimals = 18;
      if (data === "loading") return data;
      if (data > 2 ** 255) return "♾️";
      let value = ethers.utils.commify(ethers.utils.formatUnits(data, decimals));
      let parts = value.split(".");

      if (precision === 0) return parts[0];

      return parts[0] + "." + parts[1].slice(0, precision);
    },
    fromWei15(data, precision) {
      if (data === "loading") return data;
      if (data > 2 ** 255) return "♾️";
      let value = ethers.utils.commify(ethers.utils.formatUnits(data, 15));
      let parts = value.split(".");

      if (precision === 0) return parts[0];

      return parts[0] + "." + parts[1].slice(0, precision);
    },
    toPct(data, precision) {
      if (isNaN(data)) return "-";
      return `${(data * 100).toFixed(precision)}%`;
    },
    toCurrency(data, precision) {
      if ( !data ) return "-";
      
      if (typeof data !== "number") {
        data = parseFloat(data);
      }
      var formatter = new Intl.NumberFormat("en-US", {
        style: "currency",
        currency: "USD",
        minimumFractionDigits: precision,
      });
      return formatter.format(data);
    },
  },
  methods: {
    on_approve_vault() {
      this.drizzleInstance.contracts["WANT"].methods["approve"].cacheSend(
        this.vault,
        max_uint,
        { from: this.activeAccount }
      );
    },
    on_approve_bribe() {
      this.drizzleInstance.contracts["YFI"].methods["approve"].cacheSend(
        this.vault,
        max_uint,
        { from: this.activeAccount }
      );
    },
    on_deposit() {
      this.error = null;

      if (this.amount <= 0) {
        this.error = ERROR_NEGATIVE;
        this.amount = 0;
        return;
      }

      this.drizzleInstance.contracts["Vault"].methods["deposit"].cacheSend(
        ethers.utils.parseUnits(this.amount.toString(), this.vault_decimals).toString(),
        {
          from: this.activeAccount,
        }
      );
    },
    on_deposit_all() {
      if (this.want_balance <= 0) {
        this.error = ERROR_NEGATIVE_ALL;
        this.amount = 0;
        return;
      }

      this.drizzleInstance.contracts["Vault"].methods["deposit"].cacheSend({
        from: this.activeAccount,
      });
    },
    on_bribe_the_bouncer() {
      console.log(this.contractGuestList.methods);
      this.contractGuestList.methods
        .bribe_the_bouncer()
        .send({ from: this.activeAccount })
        .then((response) => {
          console.log(response);
        });
    },
    on_withdraw_all() {
      if (this.yvtoken_balance <= 0) {
        this.error = ERROR_NEGATIVE_WITHDRAW;
        this.amount = 0;
        return;
      }
      this.drizzleInstance.contracts["Vault"].methods["withdraw"].cacheSend({
        from: this.activeAccount,
      });
    },
    async load_reverse_ens() {
      let lookup = this.activeAccount.toLowerCase().substr(2) + ".addr.reverse";
      let resolver = await this.drizzleInstance.web3.eth.ens.resolver(lookup);
      let namehash = ethers.utils.namehash(lookup);
      this.username = await resolver.methods.name(namehash).call();
    },
    async get_strategies(vault) {
      for (let i = 0, p = Promise.resolve(); i < 20; i++) {
        p = p.then(
          (_) =>
            new Promise((resolve) =>
              vault.methods
                .withdrawalQueue(i)
                .call()
                .then((strat_addr) => {
                  if (strat_addr !== ADDRESS_ZERO) {
                    let Strategy = new web3.eth.Contract(yStrategy, strat_addr);
                    let data = {
                      address: strat_addr,
                      balance: null,
                    };

                    // Add to strat address to array
                    this.$set(this.strategies, i, data);

                    Strategy.methods
                      .name()
                      .call()
                      .then((name) => {
                        console.log(name);
                        this.$set(this.strategies[i], "name", name);
                      });
                  }

                  resolve();
                })
            )
        );
      }
    },
    get_block_timestamp(timestamp) {
      return axios.get(`https://api.etherscan.io/api?module=block&action=getblocknobytime&timestamp=${timestamp}&closest=before&apikey=JXRIIVMTAN887F9D7NCTVQ7NMGNT1A4KA3`)      
    },
    call(contract, method, args, out = "number") {
      let key = this.drizzleInstance.contracts[contract].methods[
        method
      ].cacheCall(...args);
      let value;
      try {
        value = this.contractInstances[contract][method][key].value;
      } catch (error) {
        value = null;
      }
      switch (out) {
        case "number":
          if (value === null) value = 0;
          return new ethers.BigNumber.from(value);
        case "address":
          return value;
        default:
          return value;
      }
    },
  },
  computed: {
    ...mapGetters("accounts", ["activeAccount", "activeBalance"]),
    ...mapGetters("drizzle", ["drizzleInstance", "isDrizzleInitialized"]),
    ...mapGetters("contracts", ["getContractData", "contractInstances"]),

    user() {
      return this.activeAccount;
    },
    vault() {
      return this.drizzleInstance.contracts["Vault"].address;
    },
    vault_perfFee() {
      return this.call("Vault", "performanceFee", [], "string") / 100;
    },
    vault_version() {
      return this.call("Vault", "apiVersion", [], "string");
    },
    vault_supply() {
      return this.call("Vault", "totalSupply", []);
    },
    vault_deposit_limit() {
      return this.call("Vault", "depositLimit", []);
    },
    vault_total_assets() {
      return this.call("Vault", "totalAssets", []);
    },
    vault_available_limit() {
      if (this.config.VAULT_STATUS == 'active' || this.config.VAULT_STATUS == '') {
        return this.call("Vault", "availableDepositLimit", []);
      } else {
        return BN_ZERO;
      }
    },
    vault_total_aum() {
      let toFloat = new ethers.BigNumber.from(10).pow(this.vault_decimals.sub(2)).toString();
      let numAum = this.vault_total_assets.div(toFloat).toNumber();
      return (numAum / 100) * this.want_price;
    },
    vault_price_per_share() {
      return this.call("Vault", "pricePerShare", []);
    },
    vault_decimals() {
      return this.call("Vault", "decimals", []);
    },
    yvtoken_balance() {
      return this.call("Vault", "balanceOf", [this.activeAccount]);
    },
    want_balance() {
      return this.call("WANT", "balanceOf", [this.activeAccount]);
    },
    eth_balance() {
      return this.activeBalance;
    },
    progress_limit() {
      return (this.vault_deposit_limit.isZero()?0:(this.vault_deposit_limit - this.vault_available_limit) / this.vault_deposit_limit);
    },
    yfi_needed() {
      return this.entrance_cost.sub(this.total_yfi);
    },
    has_allowance_bribe() {
      return !this.call("YFI", "allowance", [
        this.activeAccount,
        this.vault,
      ]).isZero();
    },
    has_allowance_vault() {
      return !this.call("WANT", "allowance", [
        this.activeAccount,
        this.vault,
      ]).isZero();
    },
    has_want_balance() {
      return this.want_balance > 0;
    },
    has_yvtoken_balance() {
      return this.yvtoken_balance > 0;
    },
  },
  async created() {
    axios
      .get(
        "https://api.coingecko.com/api/v3/simple/price?ids=" +
          this.config.COINGECKO_SYMBOL.toLowerCase() +
          "&vs_currencies=usd"
      )
      .then((response) => {
        this.want_price = response.data[this.config.COINGECKO_SYMBOL.toLowerCase()].usd;
      });

    //Active account is defined?
    if (this.activeAccount !== undefined) this.load_reverse_ens();

    let Vault = new web3.eth.Contract(yVaultV2, this.vault);
    console.log(Vault);
    this.get_strategies(Vault);

    // Get GuestList contract and use it :)
    Vault.methods.guestList().call().then((response) => {
        if (response == ADDRESS_ZERO) {
          //if there's not guest list, everyone is a guest ;)
          console.log("No guest list. Everyone is invited!");
          this.is_guest = true;
          this.total_yfi = this.entrance_cost;
        } else {
          this.contractGuestList = new web3.eth.Contract(GuestList, response);

          this.contractGuestList.methods
            .guests(this.activeAccount)
            .call()
            .then((response) => {
              this.is_guest = response;
            });
        }

        this.contractGuestList.methods.total_yfi(this.activeAccount).call().then((response) => {
            console.log("Total YFI: " + response.toString());
            this.total_yfi = new ethers.BigNumber.from(response.toString());
          });

        Vault.methods.activation().call().then((vault_activation) => {
            this.contractGuestList.methods
              .entrance_cost(vault_activation)
              .call()
              .then((response) => {
                console.log("Entrance cost: " + response.toString());
                this.entrance_cost = new ethers.BigNumber.from(
                  response.toString()
                );
              });
          });

        this.contractGuestList.methods
          .bribe_cost()
          .call()
          .then((response) => {
            console.log("Bribe cost: " + response.toString());
            this.bribe_cost = new ethers.BigNumber.from(response.toString());
          });
      });

    // Get blocknumber and calc APY
    Vault.methods.pricePerShare().call().then( currentPrice => {
      const seconds_in_a_year = 31536000;
      const now = Math.round(Date.now() / 1000);

      // 1 week ago
      const one_week_ago = (now - 60 * 60 * 24 * 7);
      const ts_past = one_week_ago < this.config.BLOCK_ACTIVATED?this.config.BLOCK_ACTIVATED:one_week_ago;

      const ts_diff = now - ts_past;

      console.log("TS Past: " + one_week_ago);
      console.log("TS Activation: " + this.config.BLOCK_ACTIVATED);

      this.get_block_timestamp(ts_past).then(response => {
        console.log("Past block: " + response.data.result);
        Vault.methods.pricePerShare().call({}, response.data.result).then( pastPrice => {
          let roi = (currentPrice / pastPrice - 1) * 100;
          console.log("Current Price: " + currentPrice);
          console.log("Past Price: " + pastPrice);
          this.roi_week = roi/ts_diff*seconds_in_a_year;
          console.log("ROI week: " + roi);
          console.log("ROI year: " + this.roi_week);
        });
      });
    });

    // Iterate through strats
  },
};
</script>
