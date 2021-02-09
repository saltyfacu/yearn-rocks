<template lang="pug">
v-app
  v-app-bar(app) Test Vaults Registry
  v-main
    v-container(fluid)
      div(v-if="isDrizzleInitialized || isHome", id="app")
        Section(:config="config" :allConfig="allConfig")
      div(v-else)
        v-progress-linear(indeterminate color="primary")
</template>

<script>
import config from './config.js'
import Vault from './Vault'
import LidoVault from './LidoVault'
import stETHLPVault from './stETHLPVault'
import Home from './Home'
import NotFound from './NotFound'
import { mapGetters } from 'vuex'

const vaultPath = window.location.pathname.substring(1)
const vaultConfig = config[vaultPath] || null;

let VaultType;

switch (window.location.pathname) {
  case '/yvsteth':
    VaultType = LidoVault;
    break;
  case '/stecrv':
    VaultType = stETHLPVault;
    break;
  default:
    VaultType = Object.prototype.hasOwnProperty.call(config, vaultPath) ? Vault : NotFound;
}

/*
const VaultType = window.location.pathname === '/yvsteth' ? LidoVault : (
  Object.prototype.hasOwnProperty.call(config, vaultPath) ? Vault : NotFound
)
*/

const Section = window.location.pathname === '/' ? Home : VaultType

export default {
  name: 'app',
  components: {
    Section,
  },
  data() {
    return {
      config: vaultConfig,
      allConfig: config,
      isHome: window.location.pathname === '/',
    }
  },
  computed: mapGetters('drizzle', ['isDrizzleInitialized'])
}
</script>
