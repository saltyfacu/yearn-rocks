<template lang="pug">
v-app
  v-app-bar(app) Test Vaults Registry
  v-main
    v-container(fluid)
      div.columns
        v-alert(type="warning" dense border="left") These vaults are experimental. They are extremely risky and will probably be discarded when production ones are deployed
      div.columns
        div.column.is-half
          v-card(class="mx-auto"
                  max-width="324"
                  outlined)
            v-card-title 🚀 Yearn Vaults
              v-card-actions(v-for="vault in yearnVaultsActive" v-bind:key="vault.TITLE")
                  v-btn
                    a( class="links" :href="'/' + vault.URL") {{ vault.LOGO }} <span class="text">{{ vault.TITLE }}</span>
                    status-tag(:status="vault.VAULT_STATUS")
              v-card-actions(v-for="vault in yearnVaultsOther" v-bind:key="vault.TITLE")
                  v-btn
                    a( class="links" :href="'/' + vault.URL") {{ vault.LOGO }} <span class="text">{{ vault.TITLE }}</span>
                    status-tag(:status="vault.VAULT_STATUS")
        div.column.is-half
          v-card(class="mx-auto"
                  max-width="324"
                  outlined)
            v-card-title 🧠 Experiments
              v-card-actions(v-for="vault in experimentVaults" v-bind:key="vault.TITLE")
                  v-btn
                    a( class="links" :href="'/' + vault.URL") {{ vault.LOGO }} <span class="text">{{ vault.TITLE }}</span>
                    status-tag(:status="vault.VAULT_STATUS")
</template>

<script>
import StatusTag from './components/StatusTag';

import { mapGetters } from "vuex";

export default {
  name: "Home",
  components: {
    StatusTag
  },
  props: ['allConfig'],
  data() {
    return {
    };
  },
  filters: {},
  methods: {},
  computed: {
    yearnVaults() {
      var items = this.allConfig;
  
      var result = Object.keys(items)
        .map(((key) => ({
          ...items[key],
          URL: key
        })))
        .filter(item => item.VAULT_TYPE === 'yearn')
        .slice().reverse()

      return result;
    },
    yearnVaultsActive() {
      var items = this.allConfig;
  
      var result = Object.keys(items)
        .map(((key) => ({
          ...items[key],
          URL: key
        })))
        .filter(item => item.VAULT_TYPE === 'yearn' && item.VAULT_STATUS === 'active')
        .slice().reverse()

      return result;
    },
    yearnVaultsOther() {
      var items = this.allConfig;
  
      var result = Object.keys(items)
        .map(((key) => ({
          ...items[key],
          URL: key
        })))
        .filter(item => 
          item.VAULT_TYPE === 'yearn' 
          && item.VAULT_STATUS != 'active'
          && item.VAULT_STATUS != 'stealth'
        )
        .slice().reverse()

      return result;
    },
    experimentVaults() {
      var items = this.allConfig;
  
      var result = Object.keys(items)
        .map(((key) => ({
          ...items[key],
          URL: key
        })))
        .filter(
          item => item.VAULT_TYPE === 'experiment' 
          && item.VAULT_STATUS != 'stealth'
        )
        .slice().reverse()

      return result;
    }
  },
  created() {},
}
</script>
