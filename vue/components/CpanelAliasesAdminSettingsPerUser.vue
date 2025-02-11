<template>
  <q-scroll-area class="full-height full-width">
    <div class="q-pa-lg ">
      <div class="row q-mb-md">
        <div class="col text-h5" v-t="'CPANELINTEGRATOR.HEADING_SETTINGS_TAB_ALIASES'"></div>
      </div>
      <q-card flat bordered class="card-edit-settings">
        <q-card-section>
          <div class="row q-mb-md">
            <div class="col-2 q-mt-sm" v-t="'CPANELINTEGRATOR.LABEL_ALIAS'"></div>
            <div class="col-3">
              <q-input outlined dense bg-color="white" v-model="aliasName"/>
            </div>
            <div class="q-ml-sm q-mr-xs">
              <span class="text-h6"><b>@</b></span>
            </div>
            <div>
              <q-select outlined dense bg-color="white" class="domains-select" v-model="selectedDomain"
                        :options="domains" option-label="name"/>
            </div>
            <div class="col-3 q-mt-xs q-ml-md">
              <q-btn unelevated no-caps no-wrap dense class="q-ml-md q-px-sm" :disable="!aliasName.length" :ripple="false" color="primary"
                     :label="$t('CPANELINTEGRATOR.ACTION_ADD_NEW_ALIAS')"
                     @click="addNewAlias"/>
            </div>
          </div>
          <div class="row q-mb-md">
            <div class="col-2"/>
            <div class="col-5">
              <select size="9" class="select" multiple v-model="selectedAliases">
                <option v-for="alias in aliasesList" :key="alias" :value="alias">{{ alias }}</option>
              </select>
            </div>
            <div class="col-3 q-mt-xs q-ml-md" style="position: relative">
              <div style="position: absolute; bottom: 3px;">
                <q-btn unelevated no-caps no-wrap dense class="q-ml-md q-px-sm" :ripple="false" color="primary"
                       :label="$t('CPANELINTEGRATOR.ACTION_DELETE_ALIASES')"
                       @click="deleteAliasesList"/>
              </div>
            </div>
          </div>
        </q-card-section>
      </q-card>
    </div>
    <q-inner-loading style="justify-content: flex-start;" :showing="loading || saving || deleting">
      <q-linear-progress query />
    </q-inner-loading>
  </q-scroll-area>
</template>

<script>
import _ from 'lodash'

import webApi from 'src/utils/web-api'
import errors from 'src/utils/errors'
import notification from 'src/utils/notification'

import typesUtils from 'src/utils/types'

export default {
  name: 'CpanelAliasesAdminSettingsPerUser',

  data () {
    return {
      loading: false,
      saving: false,
      deleting: false,
      aliasName: '',
      selectedDomain: null,
      selectedAliases: [],
      aliasesList: [],
      user: null
    }
  },

  computed: {
    currentTenantId () {
      return this.$store.getters['tenants/getCurrentTenantId']
    },

    domains () {
      const allDomainLists = this.$store.getters['maildomains/getDomains']
      return typesUtils.pArray(allDomainLists[this.currentTenantId])
    }
  },

  watch: {
    domains () {
      if (this.domains.length > 0) {
        this.selectedDomain = this.domains[0]
      }
    },
  },

  beforeRouteLeave (to, from, next) {
    this.doBeforeRouteLeave(to, from, next)
  },

  mounted () {
    this.requestDomains()
    this.parseRoute()
    if (this.selectedDomain === null && this.domains.length > 0) {
      this.selectedDomain = this.domains[0]
    }
  },

  methods: {
    /**
     * Method is used in doBeforeRouteLeave mixin
     */
    hasChanges () {
      return this.aliasName !== ''
    },

    /**
     * Method is used in doBeforeRouteLeave mixin,
     * do not use async methods - just simple and plain reverting of values
     * !! hasChanges method must return true after executing revertChanges method
     */
    revertChanges () {
      this.aliasName = ''
    },

    requestDomains () {
      this.$store.dispatch('maildomains/requestDomainsIfNecessary', {
        tenantId: this.currentTenantId
      })
    },

    parseRoute () {
      const userId = typesUtils.pPositiveInt(this.$route?.params?.id)
      if (this.user?.id !== userId) {
        this.user = {
          id: userId,
        }
        this.populate()
      }
    },

    populate () {
      this.getSettings()
    },

    addNewAlias () {
      if (!this.saving) {
        this.saving = true
        const parameters = {
          UserId: this.user?.id,
          AliasName: this.aliasName,
          AliasDomain: this.selectedDomain?.name,
          TenantId: this.currentTenantId,
        }
        webApi.sendRequest({
          moduleName: 'CpanelIntegrator',
          methodName: 'AddNewAlias',
          parameters,
        }).then(result => {
          this.saving = false
          if (result === true) {
            this.aliasName = ''
            this.populate()
          } else {
            notification.showError(this.$t('COREWEBCLIENT.ERROR_DATA_TRANSFER_FAILED'))
          }
        }, response => {
          this.saving = false
          notification.showError(errors.getTextFromResponse(response, this.$t('COREWEBCLIENT.ERROR_DATA_TRANSFER_FAILED')))
        })
      }
    },

    deleteAliasesList () {
      if (!this.deleting) {
        this.deleting = true
        if (this.selectedAliases.length) {
          const parameters = {
            UserId: this.user?.id,
            Aliases: this.selectedAliases,
            TenantId: this.tenantId,
          }
          webApi.sendRequest({
            moduleName: 'CpanelIntegrator',
            methodName: 'DeleteAlias',
            parameters,
          }).then(result => {
            this.deleting = false
            if (result === true) {
              this.populate()
            }
          }, response => {
            this.deleting = false
            notification.showError(errors.getTextFromResponse(response))
          })
        } else {
          this.deleting = false
          notification.showError(this.$t('CPANELINTEGRATOR.ERROR_EMPTY_ALIASES'))
        }
      }
    },

    getSettings() {
      this.loading = true
      const parameters = {
        UserId: this.user?.id,
        TenantId: this.currentTenantId
      }
      webApi.sendRequest({
        moduleName: 'CpanelIntegrator',
        methodName: 'GetAliases',
        parameters
      }).then(result => {
        this.loading = false
        if (_.isArray(result.Aliases)) {
          this.aliasesList = result.Aliases
        }
      },
      response => {
        this.loading = false
        notification.showError(errors.getTextFromResponse(response))
      })
    }
  }
}
</script>

<style scoped>
.select {
  padding: 7px 9px 6px;
  border: 1px solid #cccccc;
  width: 100%;
  overflow-y: scroll;
  border-radius: 4px;
}
</style>
