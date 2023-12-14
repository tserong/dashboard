<script>
import { HCI } from '../types';
import { ADD_ONS } from '../config/harvester-map';
import { NODE } from '@shell/config/types';
import { STATE, AGE, SIMPLE_NAME } from '@shell/config/table-headers';
import { allHash } from '@shell/utils/promise';
import Banner from '@components/Banner/Banner.vue';
import Loading from '@shell/components/Loading';
import MessageLink from '@shell/components/MessageLink';
import ResourceTable from '@shell/components/ResourceTable';

export default {
  name: 'ListSriovGpuDevices',

  components: {
    Banner,
    Loading,
    MessageLink,
    ResourceTable,
  },

  async fetch() {
    const inStore = this.$store.getters['currentProduct'].inStore;

    this.schema = this.$store.getters[`${ inStore }/schemaFor`](HCI.SR_IOVGPU_DEVICE);
    this.hasAddonSchema = this.$store.getters[`${ inStore }/schemaFor`](HCI.ADD_ONS);

    if (this.hasSchema) {
      try {
        const inStore = this.$store.getters['currentProduct'].inStore;

        const hash = await allHash({
          sriovgpus:   this.$store.dispatch(`${ inStore }/findAll`, { type: HCI.SR_IOVGPU_DEVICE }),
          vGpuDevices: this.$store.dispatch(`${ inStore }/findAll`, { type: HCI.VGPU_DEVICE }),
          addons:      this.$store.dispatch(`${ inStore }/findAll`, { type: HCI.ADD_ONS }),
        });

        const hasPCIAddon = hash.addons.find(addon => addon.name === ADD_ONS.PCI_DEVICE_CONTROLLER)?.spec?.enabled === true;
        const hasSriovgpuAddon = hash.addons.find(addon => addon.name === ADD_ONS.NVIDIA_DRIVER_TOOLKIT_CONTROLLER)?.spec?.enabled === true;

        this.enabledSriovgpu = hasPCIAddon && hasSriovgpuAddon;
      } catch (e) {}
    }
  },

  data() {
    const inStore = this.$store.getters['currentProduct'].inStore;

    return {
      hasAddonSchema:  false,
      enabledSriovgpu: false,
      schema:          null,
      hasNode:         this.$store.getters[`${ inStore }/schemaFor`](NODE),
      to:              `${ HCI.ADD_ONS }/harvester-system/${ ADD_ONS.NVIDIA_DRIVER_TOOLKIT_CONTROLLER }?mode=edit`
    };
  },

  computed: {
    hasSchema() {
      return !!this.schema;
    },

    rows() {
      const inStore = this.$store.getters['currentProduct'].inStore;
      const rows = this.$store.getters[`${ inStore }/all`](HCI.SR_IOVGPU_DEVICE);

      return rows;
    },

    headers() {
      const nodeCol = {
        name:      'node',
        label:     'Node',
        value:     'realNodeName',
        sort:      ['realNodeName'],
        formatter: 'CopyToClipboard',
        labelKey:  'tableHeaders.node'
      };

      const cols = [
        STATE,
        SIMPLE_NAME,
        {
          name:  'address',
          label: 'Address',
          value: 'spec.address',
          sort:  ['spec.address']
        },
        {
          name:        'numVFs',
          label:       'Num VFs',
          sort:        ['numVFs'],
          value:       'numVFs',
          formatter:   'HarvesterVFsNum',
          align:       'center',
          labelKey:    'harvester.sriovgpu.numVFs',
          dashIfEmpty: true,
        },
        {
          name:        'vfAddresses',
          label:       'VF Addresses',
          labelKey:    'harvester.sriovgpu.vfAddresses',
          sort:        ['status.vfAddresses'],
          value:       'status.vfAddresses',
          formatter:   'HarvesterVFAddress',
          align:       'center',
          dashIfEmpty: true,
        },
        {
          name:        'vGpuDevices',
          label:       'VGpu Devices',
          labelKey:    'harvester.sriovgpu.vGpuDevices',
          sort:        ['status.vGPUDevices'],
          value:       'status.vGPUDevices',
          formatter:   'HarvesterVGpuDevices',
          align:       'center',
          dashIfEmpty: true,
        },
        {
          ...AGE,
          sort: 'metadata.creationTimestamp:desc',
        }
      ];

      if (this.hasNode) {
        cols.splice(-1, 0, nodeCol);
      }

      return cols;
    },
  }
};
</script>

<template>
  <Loading v-if="$fetchState.pending" />
  <div v-else-if="!hasAddonSchema">
    <Banner color="warning">
      {{ t('harvester.sriovgpu.noPermission') }}
    </Banner>
  </div>
  <ResourceTable
    v-else-if="hasSchema && enabledSriovgpu"
    v-bind="$attrs"
    :groupable="false"
    :namespaced="false"
    :headers="headers"
    :schema="schema"
    :rows="rows"
    key-field="_key"
  />
  <div v-else>
    <Banner color="warning">
      <MessageLink
        :to="to"
        prefix-label="harvester.sriovgpu.goSetting.prefix"
        middle-label="harvester.sriovgpu.goSetting.middle"
        suffix-label="harvester.sriovgpu.goSetting.suffix"
      />
    </Banner>
  </div>
</template>
