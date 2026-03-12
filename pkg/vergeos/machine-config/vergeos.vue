<script>
import { Banner } from '@components/Banner';
import { LabeledInput } from '@components/Form/LabeledInput';
import { _VIEW } from '@shell/config/query-params';

export default {
  components: {
    Banner, LabeledInput,
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

  computed: {
    isView() {
      return this.mode === _VIEW;
    },

    isValid() {
      return !!(this.value.templateVm && this.value.network);
    },
  },

  watch: {
    isValid(val) {
      this.$emit('validationChanged', val);
    },
  },

  created() {
    this.initDefaults();
    this.$emit('validationChanged', this.isValid);
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
  },
};
</script>

<template>
  <div>
    <!-- VM Configuration -->
    <div class="row mt-20">
      <div class="col span-6">
        <LabeledInput
          v-model:value="value.templateVm"
          :label="t('machine.vergeos.templateVm.label')"
          :placeholder="t('machine.vergeos.templateVm.placeholder')"
          :mode="mode"
          :required="true"
          :disabled="disabled"
        />
      </div>
      <div class="col span-6">
        <LabeledInput
          v-model:value="value.network"
          :label="t('machine.vergeos.network.label')"
          :placeholder="t('machine.vergeos.network.placeholder')"
          :mode="mode"
          :required="true"
          :disabled="disabled"
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
