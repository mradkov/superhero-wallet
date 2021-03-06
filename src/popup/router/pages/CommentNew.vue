<template>
  <div class="popup">
    <div class="tip-note-preview mt-15">
      {{ text }}
    </div>

    <Button @click="sendComment" :disabled="!allowTipping">
      {{ $t('pages.tipPage.confirm') }}
    </Button>
    <Button @click="cancel">
      {{ $t('pages.tipPage.cancel') }}
    </Button>

    <Loader v-if="loading" />
  </div>
</template>

<script>
import { mapGetters, mapState } from 'vuex';
import Backend from '../../../lib/backend';
import openUrl from '../../utils/openUrl';

export default {
  data: () => ({ id: 0, parentId: undefined, text: '', loading: false }),
  computed: {
    ...mapState(['sdk']),
    ...mapGetters(['allowTipping']),
    urlParams() {
      return new URL(this.$route.fullPath, window.location).searchParams;
    },
  },
  async created() {
    this.loading = true;
    this.id = +this.urlParams.get('id');
    if (this.urlParams.get('parentId')) this.parentId = +this.urlParams.get('parentId');
    this.text = this.urlParams.get('text');
    if (!this.id || !this.text) {
      this.$router.push('/account');
      throw new Error('CommentNew: Invalid arguments');
    }
    await this.$watchUntilTruly(() => this.sdk);
    this.loading = false;
  },
  methods: {
    openCallbackOrGoHome(paramName) {
      const callbackUrl = this.urlParams.get(paramName);
      if (callbackUrl) openUrl(decodeURIComponent(callbackUrl));
      else this.$router.push('/account');
    },
    async sendComment() {
      this.loading = true;
      try {
        await Backend.sendTipComment(
          this.id,
          this.text,
          await this.sdk.address(),
          async data => Buffer.from(await this.sdk.signMessage(data)).toString('hex'),
          this.parentId,
        );
        this.openCallbackOrGoHome('x-success');
      } catch (e) {
        this.$store.dispatch('modals/open', { name: 'default', type: 'transaction-failed' });
        e.payload = { id: this.id, text: this.text };
        throw e;
      } finally {
        this.loading = false;
      }
    },
    cancel() {
      this.openCallbackOrGoHome('x-cancel');
    },
  },
};
</script>
