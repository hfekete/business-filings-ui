<template>
  <menu id="entity-menu">
    <!-- Staff Comments -->
    <span v-if="isAllowed(AllowableActions.STAFF_COMMENT)">
      <StaffComments
        :axios="axios"
        :businessId="businessId"
        :maxLength="2000"
      />
    </span>

    <!-- View and Change Business Information -->
    <span v-if="isBusiness && !isHistorical">
      <v-btn
        id="company-information-button"
        small
        text
        color="primary"
        :disabled="!isAllowed(AllowableActions.BUSINESS_INFORMATION)"
        @click="promptChangeCompanyInfo()"
      >
        <v-icon medium>mdi-file-document-edit-outline</v-icon>
        <span class="font-13 ml-1">View and Change Business Information</span>
      </v-btn>

      <v-tooltip
        v-if="isPendingDissolution"
        top
        content-class="top-tooltip"
        transition="fade-transition"
      >
        <template #activator="{ on }">
          <v-icon
            color="orange darken-2"
            size="24px"
            class="pr-2"
            v-on="on"
          >mdi-alert</v-icon>
        </template>
        You cannot view or change business information while the business is pending dissolution.
      </v-tooltip>
    </span>

    <!-- Download Business Summary -->
    <span v-if="isAllowed(AllowableActions.BUSINESS_SUMMARY)">
      <v-tooltip
        top
        content-class="top-tooltip"
        transition="fade-transition"
      >
        <template #activator="{ on }">
          <v-btn
            id="download-summary-button"
            small
            text
            color="primary"
            @click="emitDownloadBusinessSummary()"
            v-on="on"
          >
            <img
              src="@/assets/images/business_summary_icon.svg"
              alt=""
              class="pa-1"
            >
            <span class="font-13 ml-1">Business Summary</span>
          </v-btn>
        </template>
        View and download a summary of information about the business.
      </v-tooltip>
    </span>

    <!-- View/Add Digital Credentials -->
    <span v-if="isAllowed(AllowableActions.DIGITAL_CREDENTIALS)">
      <v-tooltip
        top
        content-class="top-tooltip"
        transition="fade-transition"
      >
        <template #activator="{ on }">
          <v-btn
            id="view-add-digital-credentials-button"
            small
            text
            color="primary"
            @click="emitViewAddDigitalCredentials()"
            v-on="on"
          >
            <v-icon medium>mdi-file-certificate-outline</v-icon>
            <span class="font-13 ml-1">Business Digital Credentials</span>
          </v-btn>
        </template>
        Manage the digital credentials generated for the business.
      </v-tooltip>
    </span>

    <!-- More Actions -->
    <span v-if="isBusiness && !isHistorical">
      <v-menu
        v-model="expand"
        offset-y
        transition="slide-y-transition"
      >
        <template #activator="{ on }">
          <v-btn
            small
            text
            color="primary"
            class="menu-btn"
            v-on="on"
          >
            <span class="font-13 ml-1">More Actions</span>
            <v-icon
              v-if="expand"
              medium
            >mdi-menu-up</v-icon>
            <v-icon
              v-else
              medium
            >mdi-menu-down</v-icon>
          </v-btn>
        </template>

        <v-list dense>
          <v-list-item-group color="primary">
            <!-- Dissolve Business -->
            <v-tooltip
              right
              content-class="right-tooltip"
            >
              <template #activator="{ on }">
                <v-list-item
                  id="dissolution-list-item"
                  :disabled="!isAllowed(AllowableActions.VOLUNTARY_DISSOLUTION)"
                  v-on="on"
                  @click="promptDissolve()"
                >
                  <v-list-item-title>
                    <span class="app-blue">Dissolve this Business</span>
                  </v-list-item-title>
                </v-list-item>
              </template>
              Dissolving the business will make this business historical
              and it will be struck from the corporate registry.
            </v-tooltip>

            <!-- Consent to Continue Out -->
            <v-tooltip
              right
              content-class="right-tooltip"
            >
              <template #activator="{ on }">
                <v-list-item
                  v-if="isBenBcCccUlc || isCoop"
                  id="cco-list-item"
                  :disabled="!isAllowed(AllowableActions.CONSENT_CONTINUATION_OUT)"
                  v-on="on"
                  @click="goToConsentContinuationOutFiling()"
                >
                  <v-list-item-title>
                    <span class="app-blue">Consent to Continue Out</span>
                  </v-list-item-title>
                </v-list-item>
              </template>
              Submit a Consent to Continue Out of the province of B.C.
            </v-tooltip>
          </v-list-item-group>
        </v-list>
      </v-menu>
    </span>
  </menu>
</template>

<script lang="ts">
import { Component, Emit, Mixins, Prop } from 'vue-property-decorator'
import { Getter } from 'pinia-class'
import axios from '@/axios-auth'
import { StaffComments } from '@bcrs-shared-components/staff-comments'
import { AllowableActions, NigsMessage, Routes } from '@/enums'
import { AllowableActionsMixin } from '@/mixins'
import { navigate } from '@/utils'
import { useBusinessStore, useConfigurationStore, useRootStore } from '@/stores'

@Component({
  components: { StaffComments }
})
export default class EntityMenu extends Mixins(AllowableActionsMixin) {
  @Prop({ required: true }) readonly businessId!: string // may be null

  @Getter(useConfigurationStore) getEditUrl!: string
  @Getter(useBusinessStore) getIdentifier!: string
  @Getter(useBusinessStore) isBenBcCccUlc!: boolean
  @Getter(useBusinessStore) isHistorical!: boolean
  @Getter(useRootStore) isPendingDissolution!: boolean

  expand = false

  // enums for template
  readonly axios = axios
  readonly AllowableActions = AllowableActions

  /** Whether this entity is a business (and not a draft IA/Registration). */
  get isBusiness (): boolean {
    return !!this.businessId
  }

  /**
   * Emits an event to display NIGS dialog if company is not in good standing except Coop
   * Otherwise, navigates to the Edit UI to view or change company information.
   */
  promptChangeCompanyInfo (): void {
    const base = `${this.getEditUrl}${this.getIdentifier}`

    if (this.isCoop) {
      navigate(`${base}/special-resolution`)
    } else if (!this.isGoodStanding) {
      this.emitNotInGoodStanding(NigsMessage.CHANGE_COMPANY_INFO)
    } else {
      const route = this.isFirm ? '/change' : '/alteration'
      navigate(`${base}${route}`)
    }
  }

  /**
   * Emits an event to display NIGS dialog if company is not in good standing.
   * Otherwise, emits an event to prompt user to confirm voluntary dissolution.
   */
  promptDissolve (): void {
    if (!this.isGoodStanding) {
      this.emitNotInGoodStanding(NigsMessage.DISSOLVE)
      return
    }
    this.emitConfirmDissolution()
  }

  goToConsentContinuationOutFiling (): void {
    // 0 means "new filing"
    this.$router.push({ name: Routes.CONSENT_CONTINUATION_OUT, params: { filingId: '0' } })
  }

  /** Emits an event to confirm dissolution. */
  @Emit('confirmDissolution')
  emitConfirmDissolution (): void {}

  /** Emits an event to download the business summary. */
  @Emit('downloadBusinessSummary')
  emitDownloadBusinessSummary (): void {}

  /** Emits an event to indicate business is not in good standing. */
  @Emit('notInGoodStanding')
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  emitNotInGoodStanding (message: NigsMessage): void {}

  /** Emits an event to view / add digital credentials. */
  @Emit('viewAddDigitalCredentials')
  emitViewAddDigitalCredentials (): void {}
}
</script>

<!-- eslint-disable max-len -->
<style lang="scss" scoped>
@import '@/assets/styles/theme.scss';

:deep(.theme--light.v-list-item--disabled) {
  opacity: 0.38 !important;
}

</style>
