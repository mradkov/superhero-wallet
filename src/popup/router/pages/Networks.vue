<template>
  <div class="popup">
    <div v-if="mode === 'list'" data-cy="networks">
      <ListItem v-for="(n, key, index) in networks" :key="index" class="network-row">
        <CheckBox
          :value="n.name === current.network"
          type="radio"
          name="activeNetwork"
          @click.native="selectNetwork(n.name)"
          prevent
        />
        <div class="mr-auto text-left">
          <p class="f-16" data-cy="network-name">{{ n.name }}</p>
          <p class="f-12 url" data-cy="network-url">
            <b>{{ $t('pages.network.url') }}</b> {{ n.url }}
          </p>
          <p class="f-12 url" data-cy="network-middleware">
            <b>{{ $t('pages.network.middleware') }}</b> {{ n.middlewareUrl }}
          </p>
        </div>
        <ae-dropdown direction="right" v-if="!n.system" data-cy="more">
          <ae-icon name="more" size="20px" slot="button" />
          <li @click="setNetworkEdit(n, index)" data-cy="edit">
            <ae-icon name="edit" />
            {{ $t('pages.network.edit') }}
          </li>
          <li @click="deleteNetwork(n, index)" data-cy="delete">
            <ae-icon name="delete" />
            {{ $t('pages.network.delete') }}
          </li>
        </ae-dropdown>
      </ListItem>
      <Button extend @click="mode = 'add'" class="mt-20" data-cy="to-add">{{
        $t('pages.network.addNetwork')
      }}</Button>
    </div>
    <div v-if="mode === 'add' || mode === 'edit'" class="mt-10">
      <Input
        :placeholder="$t('pages.network.networkNamePlaceholder')"
        :label="$t('pages.network.networkNameLabel')"
        v-model="network.name"
        data-cy="network"
      />
      <Input
        :placeholder="$t('pages.network.networkUrlPlaceholder')"
        :label="$t('pages.network.networkUrlLabel')"
        v-model="network.url"
        data-cy="url"
      />
      <Input
        :placeholder="$t('pages.network.networkMiddlewarePlaceholder')"
        :label="$t('pages.network.networkMiddlewareLabel')"
        v-model="network.middlewareUrl"
        data-cy="middleware"
      />
      <Input
        :placeholder="$t('pages.network.networkCompilerPlaceholder')"
        :label="$t('pages.network.networkCompilerLabel')"
        v-model="network.compilerUrl"
        data-cy="compiler"
      />
      <Button half @click="cancel" data-cy="cancel">{{ $t('pages.network.cancel') }}</Button>
      <Button
        class="danger"
        half
        @click="addNetwork"
        :disabled="
          !network.name ||
            !network.url ||
            !network.middlewareUrl ||
            !network.compilerUrl ||
            network.error !== false
        "
        data-cy="connect"
      >
        {{ $t('pages.network.save') }}
      </Button>
      <div v-if="network.error" class="error-msg" v-html="network.error" data-cy="error-msg"></div>
    </div>
  </div>
</template>

<script>
import { mapGetters, mapState } from 'vuex';
import Button from '../components/Button';
import Input from '../components/Input';
import CheckBox from '../components/CheckBox';
import ListItem from '../components/ListItem';
import { defaultNetworks, DEFAULT_NETWORK, AEX2_METHODS } from '../../utils/constants';
import wallet from '../../../lib/wallet';
import { postMessage } from '../../utils/connection';

const networkProps = {
  name: null,
  url: null,
  middlewareUrl: null,
  compilerUrl: null,
  error: false,
};

export default {
  components: {
    Button,
    Input,
    ListItem,
    CheckBox,
  },
  data() {
    return {
      mode: 'list',
      network: networkProps,
    };
  },
  computed: {
    ...mapState(['current']),
    ...mapGetters(['networks', 'allowTipping']),
  },
  methods: {
    async selectNetwork(network) {
      await this.$store.dispatch('switchNetwork', network);
      this.$store.commit('SET_NODE_STATUS', 'connecting');
      if (process.env.IS_EXTENSION)
        postMessage({ type: AEX2_METHODS.SWITCH_NETWORK, payload: network });
      const { title, msg } = this.$t('modals.tip-mainnet-warning');
      wallet.initSdk().then(() => {
        if (!this.allowTipping)
          this.$store.dispatch('modals/open', {
            name: 'default',
            title,
            msg,
          });
      });
    },
    cancel() {
      this.mode = 'list';
      this.network = networkProps;
    },
    setNetworkEdit(n, idx) {
      this.mode = 'edit';
      this.network = { ...networkProps, ...n, idx };
    },
    async deleteNetwork(network, idx) {
      if (network.name !== DEFAULT_NETWORK) this.selectNetwork(DEFAULT_NETWORK);
      const allNetworks = Object.values(this.networks).filter((n, i) => idx !== i);
      await this.$store.commit(
        'SET_NETWORKS',
        allNetworks.reduce((p, n) => ({ ...p, [n.name]: { ...n } }), {}),
      );
      this.$store.commit(
        'SET_USERNETWORKS',
        allNetworks.filter(({ name }) => name !== DEFAULT_NETWORK),
      );
    },
    async addNetwork() {
      try {
        this.network.error = false;
        if (!this.network.name) throw new Error('Enter network name');
        const url = new URL(this.network.url);
        const middleware = new URL(this.network.middlewareUrl);
        const compiler = new URL(this.network.compilerUrl);
        if (!url.hostname || !middleware.hostname || !compiler.hostname)
          throw new Error('Invalid hostname');

        const exist = (name, idx) =>
          (this.network.idx >= 0
            ? name === this.network.name && idx !== this.network.idx
            : name === this.network.name) || this.network.name === DEFAULT_NETWORK;
        const allNetworks = Object.values(this.networks);
        if (allNetworks.find(({ name }, idx) => exist(name, idx)))
          throw new Error('Network with this name exist');
        const newNetwork = {
          ...defaultNetworks[DEFAULT_NETWORK],
          url: this.network.url,
          internalUrl: this.network.url,
          middlewareUrl: this.network.middlewareUrl,
          compilerUrl: this.network.compilerUrl,
          name: this.network.name,
        };
        if (this.network.idx >= 0) {
          allNetworks[this.network.idx] = newNetwork;
          await this.$store.commit(
            'SET_NETWORKS',
            allNetworks.reduce((p, n) => ({ ...p, [n.name]: { ...n } }), {}),
          );
          this.selectNetwork(this.network.name);
        } else {
          allNetworks.push(newNetwork);
          await this.$store.commit('ADD_NETWORK', newNetwork);
        }
        this.$store.commit(
          'SET_USERNETWORKS',
          allNetworks.filter(({ name }) => name !== DEFAULT_NETWORK),
        );
        this.mode = 'list';
      } catch (e) {
        this.network.error = e.message;
      }
    },
  },
};
</script>
<style lang="scss">
.network-row li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
  border-top: 1px solid #100c0d !important;

  p {
    margin: 0;
  }

  p.url {
    font-weight: normal;
  }
}

.edit-btn {
  margin-left: 5px;
  margin-right: 0;
}
</style>
