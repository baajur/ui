<script>
import CardTitle from '@/components/Card-Title'
import Label from '@/components/Label'
import Parameters from '@/components/Parameters'
import PrefectSchedule from '@/components/PrefectSchedule'
import { formatTime } from '@/mixins/formatTimeMixin'
import { mapActions } from 'vuex'

export default {
  filters: {
    typeClass: val => val.split('.').pop()
  },
  components: {
    CardTitle,
    Label,
    Parameters,
    PrefectSchedule
  },
  mixins: [formatTime],
  props: {
    flow: {
      type: Object,
      default: () => {}
    },
    flowGroup: {
      type: Object,
      default: () => {}
    },
    fullHeight: {
      required: false,
      type: Boolean,
      default: () => false
    },
    lastFlowRun: {
      type: Object,
      default: () => {}
    }
  },
  data() {
    return {
      copiedText: {},
      tab: 'overview',
      //labels
      noLabelInfo: false,
      newLabel: '',
      labelMenuOpen: false,
      labelEditOpen: false,
      labelSearchInput: '',
      removingLabel: null,
      newLabels: null,
      disableRemove: false,
      disableAdd: false,
      valid: false,
      errorMessage: '',
      duplicateLabel: '',
      rules: {
        labelCheck: value => this.checkLabelInput(value) || this.errorMessage
      }
    }
  },
  computed: {
    filteredStorage() {
      if (!this.flow.storage) return {}

      const filtered = ['__version__', 'prefect_version', 'flows']

      return Object.keys(this.flow.storage)
        .filter(key => !filtered.includes(key))
        .reduce((obj, key) => {
          obj[key] = this.flow.storage[key]
          return obj
        }, {})
    },
    flows() {
      if (!this.flow.storage || !this.flow.storage.flows) return null
      return this.flow.storage.flows
    },
    labels() {
      const labels =
        this.newLabels ||
        this.flowGroup?.labels ||
        this.flow?.environment?.labels
      return labels?.slice().sort()
    },
    labelResetDisabled() {
      const labels = this.newLabels || this.labels
      return (
        Array.isArray(labels) &&
        Array.isArray(this.flow?.environment?.labels) &&
        labels.length === this.flow?.environment?.labels?.length &&
        labels.every(
          (val, index) => val === this.flow?.environment?.labels[index]
        )
      )
    }
  },
  methods: {
    ...mapActions('alert', ['setAlert']),
    checkLabelInput(val) {
      const labels = this.newLabels || this.labels
      if (labels.includes(val) && !this.disableAdd) {
        this.errorMessage = 'Duplicate label'
        this.duplicateLabel = val
        this.valid = false
        return false
      }
      this.duplicateLabel = ''
      this.valid = true
      return true
    },
    removeLabel(labelToRemove) {
      this.removingLabel = labelToRemove
      this.disableRemove = true
      const labels = this.newLabels || this.labels
      const updatedArray = labels.filter(label => {
        return labelToRemove != label
      })
      this.editLabels(updatedArray)
    },
    addLabel() {
      if (!this.valid) return
      this.disableAdd = true
      const labelArray = this.newLabels || this.labels.slice()
      labelArray.push(this.newLabel)
      this.editLabels(labelArray)
    },
    async editLabels(newLabels) {
      try {
        const { data } = await this.$apollo.mutate({
          mutation: require('@/graphql/Mutations/set-labels.gql'),
          variables: {
            flowGroupId: this.flowGroup.id,
            labelArray: newLabels
          },
          errorPolicy: 'all'
        })
        if (data) {
          this.newLabels = newLabels || this.flow.environment.labels
          this.resetLabelSettings()
        } else {
          this.labelsError()
          this.resetLabelSettings()
        }
      } catch (e) {
        this.labelsError()
        this.resetLabelSettings()
      }
    },
    labelReset() {
      this.editLabels(null)
    },
    resetLabelSettings() {
      this.removingLabel = false
      this.disableAdd = false
      this.newLabel = ''
      this.disableRemove = false
    },
    labelsError() {
      this.setAlert({
        alertShow: true,
        alertMessage:
          'There was a problem updating your labels.  Please try again.',
        alertType: 'error'
      })
    },
    clickAndCopyable(field) {
      return ['image_tag', 'image_name', 'registry_url'].includes(field)
    },
    copyToClipboard(value) {
      this.copiedText = {}
      this.copiedText[value] = true
      navigator.clipboard.writeText(value)

      setTimeout(() => {
        this.copiedText = {}
        this.copiedText[value] = false
      }, 600)
    }
  }
}
</script>

<template>
  <v-card
    class="pa-2 pr-0"
    tile
    :style="{
      height: fullHeight ? '100%' : 'auto',
      'max-height': fullHeight ? '100%' : 'auto'
    }"
  >
    <v-system-bar
      :color="flow.archived ? 'grey' : lastFlowRun ? lastFlowRun.state : 'grey'"
      :height="5"
      absolute
    >
      <!-- We should include a state icon here when we've got those -->
      <!-- <v-icon>{{ flow.flow_runs[0].state }}</v-icon> -->
    </v-system-bar>
    <CardTitle :icon="flow.archived ? 'archive' : 'pi-flow'">
      <v-row slot="title" no-gutters class="d-flex align-center">
        <v-col cols="8">
          <div class="text-truncate pb-1">
            {{ flow.name }}
          </div>
          <div
            class="subtitle-2 grey--text text--darken-2 caption position-absolute font-weight-medium"
            style="bottom: 2px;"
          >
            {{ `Version ${flow.version}` }}
          </div>
        </v-col>
      </v-row>

      <div slot="action" class="d-flex flex-column align-end">
        <v-btn
          depressed
          small
          tile
          icon
          class="button-transition w-100 d-flex justify-end"
          :color="tab == 'overview' ? 'primary' : ''"
          :style="{
            'border-right': `3px solid ${
              tab == 'overview' ? 'var(--v-primary-base)' : 'transparent'
            }`,
            'box-sizing': 'content-box',
            'min-width': '100px'
          }"
          @click="tab = 'overview'"
        >
          Overview
          <v-icon small>calendar_view_day</v-icon>
        </v-btn>

        <v-btn
          depressed
          small
          tile
          icon
          class="button-transition w-100 d-flex justify-end"
          :color="tab == 'details' ? 'primary' : ''"
          :style="{
            'border-right': `3px solid ${
              tab == 'details' ? 'var(--v-primary-base)' : 'transparent'
            }`,
            'box-sizing': 'content-box',
            'min-width': '100px'
          }"
          @click="tab = 'details'"
        >
          Details
          <v-icon small>notes</v-icon>
        </v-btn>
      </div>
    </CardTitle>

    <v-card-text class="pl-12 pt-2 card-content">
      <v-fade-transition hide-on-leave>
        <div v-if="tab == 'overview'">
          <v-list-item dense class="px-0">
            <v-list-item-content>
              <v-list-item-subtitle class="caption">
                Created by
              </v-list-item-subtitle>
              <div class="subtitle-2">
                {{
                  flow.created_by
                    ? flow.created_by.username
                    : 'A nonexistent user'
                }}
              </div>
              <div class="caption">{{ formatTime(flow.created) }} </div>
            </v-list-item-content>
          </v-list-item>

          <v-list-item v-if="flow.core_version" dense class="px-0">
            <v-list-item-content>
              <v-list-item-subtitle class="caption">
                Prefect Core Version:
              </v-list-item-subtitle>
              <div class="subtitle-2">{{ flow.core_version }}</div>
            </v-list-item-content>
          </v-list-item>

          <v-list-item dense class="px-0">
            <v-list-item-content>
              <v-list-item-subtitle class="caption">
                Schedule
              </v-list-item-subtitle>
              <div class="subtitle-2">
                <PrefectSchedule
                  v-if="flow.schedule"
                  :schedule="flow.schedule"
                />
                <span v-else>None</span>
              </div>
            </v-list-item-content>
          </v-list-item>
          <v-list-item dense class="px-0">
            <v-list-item-content width="800px" class="overflow-x-scroll">
              <v-list-item-subtitle class="caption">
                Labels
                <v-menu
                  v-model="noLabelInfo"
                  :close-on-content-click="false"
                  offset-y
                  open-on-hover
                >
                  <template v-slot:activator="{ on }">
                    <v-btn
                      :color="labels && labels.length < 1 ? 'info' : null"
                      text
                      icon
                      x-small
                      v-on="on"
                    >
                      <v-icon>
                        info
                      </v-icon>
                    </v-btn>
                  </template>
                  <v-card tile class="pa-0" max-width="320">
                    <v-card-title class="subtitle pb-1"
                      >Flow labels</v-card-title
                    >

                    <v-card-text class="pt-0">
                      Flows and agents have optional labels which allow you to
                      determine where your flows are executed. For more
                      information see
                      <a
                        href="https://docs.prefect.io/orchestration/execution/overview.html#labels"
                        target="_blank"
                        >the docs on labels</a
                      >.

                      <v-alert
                        v-if="labels && labels.length < 1"
                        border="left"
                        colored-border
                        type="info"
                        tile
                        class="body-2 mt-2"
                      >
                        If you want to add labels, click
                        <v-icon class="px-1" x-small color="primary"
                          >fa-plus</v-icon
                        >
                        to add new labels or
                        <v-icon class="px-1" x-small color="codePink"
                          >fa-undo</v-icon
                        >
                        to reset your labels to the ones set at flow
                        registration.
                      </v-alert>
                    </v-card-text>
                  </v-card>
                </v-menu>
                <span v-if="$vuetify.breakpoint.sm" class="ml-8">
                  <v-menu
                    v-model="labelEditOpen"
                    :close-on-content-click="false"
                    offset-y
                  >
                    <template v-slot:activator="{ on: menu, attrs }">
                      <v-tooltip bottom>
                        <template v-slot:activator="{ on: tooltip }">
                          <v-btn
                            small
                            icon
                            aria-label="Add label"
                            color="primary"
                            v-bind="attrs"
                            v-on="{ ...tooltip, ...menu }"
                          >
                            <v-icon small dense>fa-plus</v-icon>
                          </v-btn>
                        </template>
                        <span>Add a label</span>
                      </v-tooltip>
                    </template>
                    <v-card width="800px" class="overflow-y-scroll py-0">
                      <v-card-title class="subtitle pr-2 pt-2 pb-0"
                        >Flow labels</v-card-title
                      >

                      <v-card-text class="py-0 width=1500px">
                        <div class="width=800px">
                          Flows and agents have optional labels which allow you
                          to determine where your flows are executed. For more
                          information see
                          <a
                            href="https://docs.prefect.io/orchestration/execution/overview.html#labels"
                            target="_blank"
                            >the docs on labels</a
                          >.
                        </div>
                        <v-text-field
                          v-model="newLabel"
                          :rules="[rules.labelCheck]"
                          color="primary"
                          clearable
                          class="mr-2 width=2000px"
                          :disabled="disableAdd"
                          @keyup.enter="addLabel"
                        >
                          <template v-slot:prepend-inner>
                            <Label
                              v-for="(label, i) in newLabels || labels"
                              :key="i"
                              closable
                              :duplicate="duplicateLabel === label"
                              :loading="removingLabel === label"
                              :disabled="disableRemove"
                              class="mr-1 mb-1"
                              @remove="removeLabel"
                              >{{ label }}</Label
                            >
                          </template>
                        </v-text-field>
                      </v-card-text>
                    </v-card>
                  </v-menu>
                  <v-tooltip top>
                    <template v-slot:activator="{ on }">
                      <v-btn
                        class="mt-0"
                        icon
                        :disabled="labelResetDisabled"
                        color="codePink"
                        small
                        @click="labelReset"
                        v-on="on"
                      >
                        <v-icon small dense>fa-undo</v-icon>
                      </v-btn>
                    </template>
                    <span>Reset to labels from flow registration</span>
                  </v-tooltip>
                </span>
              </v-list-item-subtitle>

              <div
                v-if="
                  (newLabels && newLabels.length > 0) ||
                    (labels && labels.length > 0)
                "
              >
                <Label
                  v-for="(label, i) in newLabels || labels"
                  :key="i"
                  closable
                  :duplicate="duplicateLabel === label"
                  :loading="removingLabel === label"
                  :disabled="disableRemove"
                  class="mr-1 mb-1"
                  @remove="removeLabel"
                  >{{ label }}</Label
                >
              </div>
              <div v-else class="subtitle-2">
                None
              </div>
            </v-list-item-content>
            <v-list-item-action v-if="!$vuetify.breakpoint.sm" class="ma-0">
              <v-tooltip top>
                <template v-slot:activator="{ on }">
                  <v-btn
                    class="mt-0"
                    icon
                    :disabled="labelResetDisabled"
                    color="codePink"
                    small
                    @click="labelReset"
                    v-on="on"
                  >
                    <v-icon small dense>fa-undo</v-icon>
                  </v-btn>
                </template>
                <span>Reset to labels from flow registration</span>
              </v-tooltip>
              <v-menu
                v-model="labelEditOpen"
                :close-on-content-click="false"
                offset-y
              >
                <template v-slot:activator="{ on: menu, attrs }">
                  <v-tooltip top>
                    <template v-slot:activator="{ on: tooltip }">
                      <v-btn
                        small
                        icon
                        aria-label="Add label"
                        color="primary"
                        v-bind="attrs"
                        v-on="{ ...tooltip, ...menu }"
                      >
                        <v-icon small dense>fa-plus</v-icon>
                      </v-btn>
                    </template>
                    <span>Add a label</span>
                  </v-tooltip>
                </template>
                <v-card width="800px" class="overflow-y-scroll py-0">
                  <v-card-title class="subtitle pr-2 pt-2 pb-0"
                    >Flow labels</v-card-title
                  >

                  <v-card-text class="py-0 width=1500px">
                    <div class="width=800px">
                      Flows and agents have optional labels which allow you to
                      determine where your flows are executed. For more
                      information see
                      <a
                        href="https://docs.prefect.io/orchestration/execution/overview.html#labels"
                        target="_blank"
                        >the docs on labels</a
                      >.
                    </div>
                    <v-text-field
                      v-model="newLabel"
                      :rules="[rules.labelCheck]"
                      color="primary"
                      clearable
                      class="mr-2 width=2000px"
                      :disabled="disableAdd"
                      @keyup.enter="addLabel"
                    >
                      <template v-slot:prepend-inner>
                        <Label
                          v-for="(label, i) in newLabels || labels"
                          :key="i"
                          closable
                          :duplicate="duplicateLabel === label"
                          :loading="removingLabel === label"
                          :disabled="disableRemove"
                          class="mr-1 mb-1"
                          @remove="removeLabel"
                          >{{ label }}</Label
                        >
                      </template>
                    </v-text-field>
                  </v-card-text>
                </v-card>
              </v-menu>
            </v-list-item-action>
          </v-list-item>
        </div>
      </v-fade-transition>

      <v-fade-transition hide-on-leave>
        <div v-if="tab == 'details'">
          <v-list-item dense class="px-0">
            <v-list-item-content>
              <v-list-item-subtitle class="grey--text text--darken-3">
                General
              </v-list-item-subtitle>
              <v-divider style="max-width: 50%;" />
              <v-list-item-subtitle class="caption">
                <v-row no-gutters>
                  <v-col cols="6">
                    Flow ID
                  </v-col>
                  <v-col
                    cols="6"
                    class="text-right font-weight-bold text-truncate"
                  >
                    <v-tooltip bottom>
                      <template v-slot:activator="{ on }">
                        <span
                          class="cursor-pointer show-icon-hover-focus-only pa-2px"
                          role="button"
                          @click="copyToClipboard(flow.id)"
                          v-on="on"
                        >
                          <v-icon x-small class="mb-2px mr-2" tabindex="0">
                            {{ copiedText[flow.id] ? 'check' : 'file_copy' }}
                          </v-icon>
                          {{ flow.id }}
                        </span>
                      </template>
                      <span>
                        {{ flow.id }}
                      </span>
                    </v-tooltip>
                  </v-col>
                </v-row>
              </v-list-item-subtitle>
              <v-list-item-subtitle class="caption">
                <v-row no-gutters>
                  <v-col cols="6">
                    Flow Group ID
                  </v-col>
                  <v-col
                    cols="6"
                    class="text-right font-weight-bold text-truncate"
                  >
                    <v-tooltip bottom>
                      <template v-slot:activator="{ on }">
                        <span
                          class="cursor-pointer show-icon-hover-focus-only pa-2px"
                          role="button"
                          @click="copyToClipboard(flow.flow_group_id)"
                          v-on="on"
                        >
                          <v-icon x-small class="mb-2px mr-2" tabindex="0">
                            {{
                              copiedText[flow.flow_group_id]
                                ? 'check'
                                : 'file_copy'
                            }}
                          </v-icon>
                          {{ flow.flow_group_id }}
                        </span>
                      </template>
                      <span>
                        {{ flow.flow_group_id }}
                      </span>
                    </v-tooltip>
                  </v-col>
                </v-row>
              </v-list-item-subtitle>
            </v-list-item-content>
          </v-list-item>

          <v-list-item dense class="px-0">
            <v-list-item-content>
              <v-list-item-subtitle class="grey--text text--darken-3">
                Environment
              </v-list-item-subtitle>
              <v-divider style="max-width: 50%;" />
              <v-list-item-subtitle class="caption">
                <v-row v-if="flow.environment.type" no-gutters>
                  <v-col cols="6">
                    Type
                  </v-col>
                  <v-col cols="6" class="text-right font-weight-bold">
                    {{ flow.environment.type }}
                  </v-col>
                </v-row>

                <v-row v-if="flow.environment.executor" no-gutters>
                  <v-col cols="6">
                    Executor
                  </v-col>
                  <v-col cols="6" class="text-right font-weight-bold">
                    {{ flow.environment.executor | typeClass }}
                  </v-col>
                </v-row>
              </v-list-item-subtitle>
            </v-list-item-content>
          </v-list-item>

          <v-list-item v-if="flow.storage" dense class="px-0 mt-2">
            <v-list-item-content>
              <v-list-item-subtitle class="grey--text text--darken-3">
                Storage
              </v-list-item-subtitle>
              <v-divider style="max-width: 50%;" />
              <v-list-item-subtitle class="caption">
                <v-row
                  v-for="(value, key) in filteredStorage"
                  :key="key"
                  no-gutters
                >
                  <v-col cols="4">
                    {{ key }}
                  </v-col>
                  <v-col
                    cols="8"
                    class="text-right font-weight-bold text-truncate"
                  >
                    <v-tooltip
                      v-if="clickAndCopyable(key)"
                      bottom
                      max-width="300"
                    >
                      <template v-slot:activator="{ on }">
                        <span
                          class="cursor-pointer show-icon-hover-focus-only pa-2px"
                          :class="{
                            'bg-gray-transition': copiedText[value],
                            'bg-white-transition': !copiedText[value]
                          }"
                          role="button"
                          @click="copyToClipboard(value)"
                          v-on="on"
                        >
                          <v-icon x-small class="mb-2px mr-2" tabindex="0">{{
                            copiedText[value] ? 'check' : 'file_copy'
                          }}</v-icon
                          >{{ value }}
                        </span>
                      </template>
                      <span>
                        {{ value }}
                      </span>
                    </v-tooltip>
                    <span v-else>
                      {{ value }}
                    </span>
                  </v-col>
                </v-row>
              </v-list-item-subtitle>
            </v-list-item-content>
          </v-list-item>

          <v-list-item v-if="flows" dense class="px-0 mt-2">
            <v-list-item-content>
              <v-list-item-subtitle class="grey--text text--darken-3">
                Flow Locations
              </v-list-item-subtitle>
              <v-divider style="max-width: 50%;" />
              <v-list-item-subtitle class="caption">
                <v-row
                  v-for="(value, key) in flows"
                  :key="key"
                  no-gutters
                  class="my-1"
                >
                  <v-col cols="12">
                    {{ key }}
                  </v-col>
                  <v-col cols="12" class="font-weight-bold">
                    {{ value }}
                  </v-col>
                </v-row>
              </v-list-item-subtitle>
            </v-list-item-content>
          </v-list-item>

          <v-list-item
            v-if="flow.parameters && flow.parameters.length > 0"
            dense
            two-line
            class="px-0"
          >
            <v-list-item-content>
              <v-list-item-subtitle class="grey--text text--darken-3">
                Parameters
              </v-list-item-subtitle>
              <v-divider style="max-width: 50%;" />
              <v-list-item-subtitle>
                <Parameters :parameters="flow.parameters"></Parameters>
              </v-list-item-subtitle>
            </v-list-item-content>
          </v-list-item>
        </div>
      </v-fade-transition>
    </v-card-text>
  </v-card>
</template>

<style lang="scss">
.bg-gray-transition {
  background-color: #efefef;
  transition: background-color 300ms;
}

.bg-white-transition {
  background-color: #fff;
  transition: background-color 300ms;
}

.card-content {
  max-height: 254px;
  overflow-y: scroll;
}

.cursor-pointer {
  cursor: pointer;
}

.label-search {
  border-radius: 0 !important;
  font-size: 0.85rem !important;

  .v-icon {
    font-size: 20px !important;
  }
}

.max-h-300 {
  max-height: 300px;
}

.mb-2px {
  margin-bottom: 2px;
}

.pa-2px {
  padding: 2px;
}

.show-icon-hover-focus-only {
  .v-icon {
    visibility: hidden;
  }

  &:hover,
  &:focus {
    .v-icon {
      visibility: visible;
    }
  }
}
/* stylelint-disable */

.v-list-item__action--stack {
  flex-direction: row;
  align-items: flex-start;
}

.v-text-field input {
  width: 100px;
}

.v-list-item__content {
  overflow: scroll;
}

.w-100 {
  width: 100% !important;
}
</style>
