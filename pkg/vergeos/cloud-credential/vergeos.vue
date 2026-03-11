<script>
import { LabeledInput } from '@components/Form/LabeledInput';
import { Checkbox } from '@components/Form/Checkbox';
import { Banner } from '@components/Banner';
import { _VIEW } from '@shell/config/query-params';

export default {
  components: { Banner, LabeledInput, Checkbox },

  props: {
    value: {
      type:     Object,
      required: true,
    },
    mode: {
      type:    String,
      default: 'create',
    },
  },

  data() {
    return { error: '' };
  },

  computed: {
    isView() {
      return this.mode === _VIEW;
    },

    host: {
      get() {
        return this.value.decodedData?.host || '';
      },
      set(val) {
        this.value.setData('host', val);
        this.updateValidation();
      },
    },

    apiKey: {
      get() {
        return this.value.decodedData?.apiKey || '';
      },
      set(val) {
        this.value.setData('apiKey', val);
        this.updateValidation();
      },
    },

    insecure: {
      get() {
        return this.value.decodedData?.insecure === 'true';
      },
      set(val) {
        this.value.setData('insecure', String(val));
        this.updateValidation();
      },
    },
  },

  methods: {
    updateValidation() {
      this.error = '';
      this.$emit('validationChanged', !!(this.host && this.apiKey));
    },

    async test() {
      this.error = '';

      if (!this.host || !this.apiKey) {
        this.error = 'Host and API Key are required';

        return false;
      }

      try {
        const host = this.host.replace(/\/+$/, '');
        const url = `/meta/proxy/${ host }/api/v4/vms?rows=1`;

        await this.$store.dispatch('management/request', {
          url,
          headers: {
            Accept:              'application/json',
            'X-API-Auth-Header': `Bearer ${ this.apiKey }`,
          },
          method:               'GET',
          redirectUnauthorized: false,
        });

        // Store host in annotations for machine-config to read
        if (!this.value.annotations) {
          this.value.annotations = {};
        }
        this.value.annotations['vergeos.io/host'] = this.host;
        this.value.annotations['vergeos.io/insecure'] = String(this.insecure);

        return true;
      } catch (e) {
        this.error = e?.message || this.t('machine.vergeos.credentialError');

        return false;
      }
    },
  },

  mounted() {
    this.updateValidation();
  },
};
</script>

<template>
  <div>
    <Banner
      v-if="error"
      color="error"
      :label="error"
    />

    <div class="row mt-20">
      <div class="col span-6">
        <LabeledInput
          v-model:value="host"
          :label="t('machine.vergeos.host.label')"
          :placeholder="t('machine.vergeos.host.placeholder')"
          :mode="mode"
          :required="true"
        />
        <p class="text-muted mt-5">
          {{ t('machine.vergeos.host.help') }}
        </p>
      </div>
      <div class="col span-6">
        <LabeledInput
          v-model:value="apiKey"
          :label="t('machine.vergeos.apiKey.label')"
          :mode="mode"
          :required="true"
          type="password"
        />
      </div>
    </div>

    <div class="row mt-20">
      <div class="col span-6">
        <Checkbox
          v-model:value="insecure"
          :label="t('machine.vergeos.insecure.label')"
          :mode="mode"
        />
      </div>
    </div>
  </div>
</template>
