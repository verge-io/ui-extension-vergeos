<script>
import Loading from '@shell/components/Loading';
import { Banner } from '@components/Banner';
import { LabeledInput } from '@components/Form/LabeledInput';
import LabeledSelect from '@shell/components/form/LabeledSelect';
import { NORMAN, SECRET } from '@shell/config/types';
import { _VIEW } from '@shell/config/query-params';

const UBUNTU_2404_MATCH = 'Ubuntu 24.04';

export default {
  components: {
    Banner, Loading, LabeledInput, LabeledSelect,
  },

  props: {
    uuid:         { type: String, required: true },
    cluster:      { type: Object, default: () => ({}) },
    credentialId: { type: String, required: true },
    disabled:     { type: Boolean, default: false },
    busy:         { type: Boolean, default: false },
    provider:     { type: String, required: true },
    value:        { type: Object, required: true },
    mode:         { type: String, default: 'create' },
  },

  data() {
    return {
      errors:         [],
      vmOptions:      [],
      networkOptions: [],
      loadingVms:     false,
      loadingNets:    false,
      credential:     null,
      host:           '',
      apiKey:         '',
    };
  },

  computed: {
    isView() {
      return this.mode === _VIEW;
    },

    isValid() {
      return !!(this.value.templateVm && this.value.network);
    },
  },

  watch: {
    credentialId() {
      this.$fetch();
    },

    isValid(val) {
      this.$emit('validationChanged', val);
    },
  },

  async fetch() {
    this.errors = [];
    this.vmOptions = [];
    this.networkOptions = [];

    if (!this.credentialId) {
      return;
    }

    if (this.mode === _VIEW) {
      return;
    }

    try {
      this.credential = await this.$store.dispatch('rancher/find', {
        type: NORMAN.CLOUD_CREDENTIAL,
        id:   this.credentialId,
      });

      this.host = this.credential?.annotations?.['vergeos.io/host'] || '';
      this.apiKey = this.credential?.decodedData?.apiKey || '';

      if (!this.host || !this.apiKey) {
        this.errors.push('Cloud credential is missing host or API key');

        return;
      }

      await Promise.all([
        this.fetchVms(),
        this.fetchNetworks(),
      ]);

      this.$emit('validationChanged', this.isValid);
    } catch (e) {
      this.errors.push(e?.message || this.t('machine.vergeos.fetchError'));
    }
  },

  created() {
    this.initDefaults();
  },

  methods: {
    initDefaults() {
      if (!this.value.cpuCores) {
        this.value.cpuCores = 2;
      }
      if (!this.value.ram) {
        this.value.ram = 4096;
      }
      if (this.value.diskSize === undefined || this.value.diskSize === null) {
        this.value.diskSize = 0;
      }
      if (!this.value.sshUser) {
        this.value.sshUser = 'ubuntu';
      }
      if (!this.value.sshPort) {
        this.value.sshPort = 22;
      }
    },

    apiRequest(path) {
      const host = this.host.replace(/\/+$/, '');

      return this.$store.dispatch('management/request', {
        url:     `/meta/proxy/${ host }${ path }`,
        headers: {
          Accept:        'application/json',
          Authorization: `Bearer ${ this.apiKey }`,
          'X-JSON-Non-Compact': '1',
        },
        method:               'GET',
        redirectUnauthorized: false,
      });
    },

    async fetchVms() {
      this.loadingVms = true;

      try {
        const res = await this.apiRequest('/api/v4/vms');
        const vms = res || [];

        // Filter to non-snapshot VMs with guest agent enabled
        const candidates = vms.filter((vm) => !vm.is_snapshot && vm.guest_agent);

        // Check each candidate for Ubuntu 24.04 via dashboard/guest agent
        const checks = await Promise.allSettled(
          candidates.map(async(vm) => {
            const dashboard = await this.apiRequest(`/api/v4/machines/${ vm.machine }?fields=dashboard`);
            const osName = dashboard?.status?.agent_guest_info?.osinfo?.['pretty-name'] || '';

            if (osName.includes(UBUNTU_2404_MATCH)) {
              return {
                label: `${ vm.name } (${ osName })`,
                value: vm.name,
              };
            }

            return null;
          })
        );

        this.vmOptions = checks
          .filter((r) => r.status === 'fulfilled' && r.value)
          .map((r) => r.value);
      } catch (e) {
        this.errors.push(`Failed to load VMs: ${ e?.message || 'unknown error' }`);
      } finally {
        this.loadingVms = false;
      }
    },

    async fetchNetworks() {
      this.loadingNets = true;

      try {
        const res = await this.apiRequest('/api/v4/vnets');
        const nets = res || [];

        this.networkOptions = nets
          .filter((n) => n.type === 'internal' || n.type === 'external')
          .map((n) => ({
            label: `${ n.name } (${ n.type })`,
            value: n.name,
          }));
      } catch (e) {
        this.errors.push(`Failed to load networks: ${ e?.message || 'unknown error' }`);
      } finally {
        this.loadingNets = false;
      }
    },
  },
};
</script>

<template>
  <Loading v-if="$fetchState.pending" :delayed="true" />

  <div v-else-if="errors.length">
    <Banner
      v-for="(err, idx) in errors"
      :key="idx"
      color="error"
      :label="err"
    />
  </div>

  <div v-else>
    <!-- VM Configuration -->
    <div class="row mt-20">
      <div class="col span-6">
        <LabeledSelect
          v-model:value="value.templateVm"
          :label="t('machine.vergeos.templateVm.label')"
          :placeholder="t('machine.vergeos.templateVm.placeholder')"
          :options="vmOptions"
          :mode="mode"
          :required="true"
          :disabled="disabled"
          :loading="loadingVms"
          :searchable="true"
        />
      </div>
      <div class="col span-6">
        <LabeledSelect
          v-model:value="value.network"
          :label="t('machine.vergeos.network.label')"
          :placeholder="t('machine.vergeos.network.placeholder')"
          :options="networkOptions"
          :mode="mode"
          :required="true"
          :disabled="disabled"
          :loading="loadingNets"
          :searchable="true"
        />
      </div>
    </div>

    <div class="row mt-20">
      <div class="col span-4">
        <LabeledInput
          v-model:value.number="value.cpuCores"
          :label="t('machine.vergeos.cpuCores.label')"
          :mode="mode"
          :disabled="disabled"
          type="number"
          min="1"
          max="32"
        />
      </div>
      <div class="col span-4">
        <LabeledInput
          v-model:value.number="value.ram"
          :label="t('machine.vergeos.ram.label')"
          :mode="mode"
          :disabled="disabled"
          type="number"
          min="1024"
        />
      </div>
      <div class="col span-4">
        <LabeledInput
          v-model:value.number="value.diskSize"
          :label="t('machine.vergeos.diskSize.label')"
          :mode="mode"
          :disabled="disabled"
          type="number"
          min="0"
        />
        <p class="text-muted mt-5">
          {{ t('machine.vergeos.diskSize.help') }}
        </p>
      </div>
    </div>

    <div class="row mt-20">
      <div class="col span-12">
        <LabeledInput
          v-model:value="value.userdata"
          :label="t('machine.vergeos.userdata.label')"
          :placeholder="t('machine.vergeos.userdata.placeholder')"
          :mode="mode"
          :disabled="disabled"
          type="multiline"
        />
      </div>
    </div>

    <!-- SSH -->
    <div class="row mt-20">
      <div class="col span-6">
        <LabeledInput
          v-model:value="value.sshUser"
          :label="t('machine.vergeos.sshUser.label')"
          :mode="mode"
          :disabled="disabled"
        />
      </div>
      <div class="col span-6">
        <LabeledInput
          v-model:value.number="value.sshPort"
          :label="t('machine.vergeos.sshPort.label')"
          :mode="mode"
          :disabled="disabled"
          type="number"
          min="1"
          max="65535"
        />
      </div>
    </div>
  </div>
</template>
