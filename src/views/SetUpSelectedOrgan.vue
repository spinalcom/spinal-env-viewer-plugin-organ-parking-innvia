<!--
Copyright 2021 SpinalCom - www.spinalcom.com

This file is part of SpinalCore.

Please read all of the following terms and conditions
of the Free Software license Agreement ("Agreement")
carefully.

This Agreement is a legally binding contract between
the Licensee (as defined below) and SpinalCom that
sets forth the terms and conditions that govern your
use of the Program. By installing and/or using the
Program, you agree to abide by all the terms and
conditions stated or referenced herein.

If you do not agree to abide by these terms and
conditions, do not demonstrate your acceptance and do
not install or use the Program.
You should have received a copy of the license along
with this file. If not, see
<http://resources.spinalcom.com/licenses.pdf>.
-->

<template>
  <v-card class="spinal-setup-organ-body">
    <set-up-toolbar
      title="Setup Context's equipment to send"
      @back="back"
      @save="save"
    />

    <v-card-text class="spinal-setup-organ-container">
      <form action="none">
        <md-field>
          <label for="Select Spatial Context">Select the Organ</label>
          <md-select
            id="modelselect"
            v-model="selectedOrgan"
            name="modelselect"
            md-dense
          >
            <md-option
              v-for="organCfg in organsCfg"
              :key="organCfg.id"
              :value="organCfg.id"
            >
              {{ organCfg.name }}
            </md-option>
          </md-select>
        </md-field>
        <template v-if="selectedOrgan">
          <md-field>
            <label for="Select Spatial Context"
              >Select the Spatial Context</label
            >
            <md-select
              id="modelselect"
              v-model="selectedSpatialContext"
              name="modelselect"
              md-dense
            >
              <md-optgroup
                v-if="spatialContexts.length > 0"
                label="Spatial contexts"
              >
                <md-option
                  v-for="context in spatialContexts"
                  :key="context.id"
                  :value="context.id"
                >
                  {{ context.name }}
                </md-option>
              </md-optgroup>

              <md-optgroup v-if="contexts.length > 0" label="Other contexts">
                <md-option
                  v-for="context in contexts"
                  :key="context.id"
                  :value="context.id"
                >
                  {{ context.name }}
                </md-option>
              </md-optgroup>
            </md-select>
          </md-field>
          <v-text-field
            v-model="apiLogin"
            placeholder="Api Login"
            autocomplete="off"
            label="Api Login"
          />

          <v-text-field
            v-model="apiPassword"
            autocomplete="off"
            placeholder="Api Password"
            label="Api Password"
            :append-icon="showApiPassword ? 'visibility' : 'visibility_off'"
            :type="showApiPassword ? 'text' : 'password'"
            @click:append="showApiPassword = !showApiPassword"
          />

          <v-text-field
            v-model="apiKey"
            autocomplete="off"
            placeholder="Api Key"
            label="Api Key"
            :append-icon="showApiKey ? 'visibility' : 'visibility_off'"
            :type="showApiKey ? 'text' : 'password'"
            @click:append="showApiKey = !showApiKey"
          />

          <v-text-field
            v-model="pullInterval"
            placeholder="Interval between each data pull"
            type="number"
            label="Interval between each data pull in ms"
          />

          <VueCtkDateTimePicker
            v-model="lastSyncCompu"
            :dark="true"
            :max-date="today"
            label="Last Synchonization"
          />
        </template>
      </form>
    </v-card-text>
  </v-card>
</template>

<script>
import {
  SpinalGraphService,
  SpinalNodePointer,
} from 'spinal-env-viewer-graph-service';
import { FileSystem } from 'spinal-core-connectorjs_type';
import moment from 'moment';
import spinalEnvViewerContextGeographicService from 'spinal-env-viewer-context-geographic-service';
import SetUpToolbar from './SetUpToolbar.vue';

const CONTEXT_GEO_TYPE =
  spinalEnvViewerContextGeographicService.constants.CONTEXT_TYPE;
export default {
  name: 'SetUpSelectedOrgan',
  components: { SetUpToolbar },
  props: {
    contextId: { required: true, type: String },
  },
  data: function () {
    const today = new Date();
    return {
      spin: false,
      today: today.toISOString(),
      selectedOrgan: null,
      spatialContexts: [],
      contexts: [],
      selectedSpatialContext: null,
      apiLogin: '',
      apiPassword: '',
      apiKey: '',
      showApiPassword: false,
      showApiKey: false,
      pullInterval: 5 * 60 * 1000, // 5 min
      lastSync: NaN,
      organsCfg: [],
      // selectedContextId: ""
    };
  },
  computed: {
    lastSyncCompu: {
      get() {
        if (!this.lastSync) {
          return null;
        } else {
          return new Date(this.lastSync).toISOString();
        }
      },
      set(value) {
        this.lastSync = moment.utc(value, 'YYYY-MM-DD HH:mm:ss');
      },
    },
  },
  watch: {
    async selectedOrgan() {
      const selectedFile = FileSystem._objects[this.selectedOrgan];
      // eslint-disable-next-line no-undef
      const node = await spinal.spinalSystem.loadModelPtr(selectedFile);
      this.apiLogin = node.client.apiLogin.get();
      this.apiPassword = node.client.apiPassword.get();
      this.apiKey = node.client.apiKey.get();
      this.pullInterval = node.client.pullInterval.get();
      this.lastSync = node.client.lastSync.get();
      this.selectedSpatialContext = node.spatialContextID.get();
    },
  },
  async mounted() {
    // eslint-disable-next-line no-undef
    const files = await spinal.spinalSystem.load('/etc/Organs/powerbat');
    const organ = await this.getLinkedOrgan();
    for (let idx = 0; idx < files.length; idx++) {
      const file = files[idx];
      this.organsCfg.push({
        name: file.name.get(),
        id: file._server_id,
        ptr: file._ptr.data.value,
      });
      if (organ && organ._server_id === file._ptr.data.value) {
        this.selectedOrgan = file._server_id;
      }
    }

    const graph = await window.spinal.spinalSystem.getModel();
    const children = await graph.getChildren();
    this.contexts = children.reduce((acc, itm) => {
      if (itm.info.type.get() === CONTEXT_GEO_TYPE) {
        this.spatialContexts.push({
          id: itm.info.id.get(),
          name: itm.info.name.get(),
        });
        return acc;
      }
      acc.push({
        id: itm.info.id.get(),
        name: itm.info.name.get(),
      });
      return acc;
    }, []);
  },
  methods: {
    async getLinkedOrgan() {
      const node = SpinalGraphService.getRealNode(this.contextId);
      if (!node.element) {
        return null;
      }
      const element = await node.element.load();
      if (element.contextId && this.contextId === element.contextId.get()) {
        return element;
      }
      return null;
    },

    saveSetting() {
      console.log('SaveSetting', this);
    },
    back() {
      this.$emit('close');
    },
    async save() {
      const selectedFile = FileSystem._objects[this.selectedOrgan];
      // eslint-disable-next-line no-undef
      const organCfgModel = await spinal.spinalSystem.loadModelPtr(
        selectedFile
      );
      const node = SpinalGraphService.getRealNode(this.contextId);
      if (node.element === undefined) {
        node.add_attr('element', new SpinalNodePointer(organCfgModel));
      } else if (node.element.ptr.data !== organCfgModel._server_id) {
        node.element.setElement(organCfgModel);
      }
      // eslint-disable-next-line no-undef
      organCfgModel.digitalTwinPath.set(spinal.spinalSystem.getPath());
      organCfgModel.contextId.set(this.contextId);
      organCfgModel.spatialContextID.set(this.selectedSpatialContext);
      organCfgModel.client.apiLogin.set(this.apiLogin);
      organCfgModel.client.apiPassword.set(this.apiPassword);
      organCfgModel.client.apiKey.set(this.apiKey);
      organCfgModel.client.pullInterval.set(this.pullInterval);
      organCfgModel.client.lastSync.set(this.lastSync);
      organCfgModel.restart.set(true);
      this.$emit('close');
    },
  },
};
</script>

<style scoped>
.spinal-setup-organ-body {
  height: 100%;
  position: relative;
  overflow: hidden;
}

.spinal-setup-organ-container {
  height: calc(100% - 50px);
  position: relative;
  overflow: auto;
}
</style>
