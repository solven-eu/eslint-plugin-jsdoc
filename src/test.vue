<template>
  <div v-if="initialized && (i18nLanguageReady || profileShared)">
    <!-- set-step is the variable to set the step for the breadcrumb-->
    <!-- if we don't have any data on the user -->
    <div v-if="!userInfo.profiles">
      <!-- if no data is found -->
      <div class="row info-message">
        <div class="content col-12 text-center">
          <div class="sad-cloud" />
        </div>
        <div class="content col-12 text-center">
          <h1 class="info-message-title">
            Oops
          </h1>
          <h4 class="info-message-description">
            {{ $t('message.GUI_did_not_find_client_infos') }}
          </h4>
        </div>
        <div class="content col-12">
          <router-link
            :to="{ name: 'ChooseDp', params: {
              client_id: clientId,
              sharing_flow_id: sharingFlowId }
            }"
          >
            <button class="btn btn-primary btn-big">
              {{ $t('message.GUI_choose_another_provider') }}
            </button>
            <br>
          </router-link>
        </div>
      </div>
    </div>

    <div v-else>
      <!-- mobile display -->
      <div class="col-10 offset-1 d-md-none">
        <div
          v-if="userInfo.profiles"
          class="row item-list"
        >
          <div
            v-for="(profile, profileKey) in showAllData(validProfiles)"
            :key="profileKey"
            class="col-12"
          >
            <ProfileContainer
              :original-content="profile.data.display"
              :profile-data-raw="profile.data.raw"
              :profile-data-jwe="profile.data.jwe"
              :is-valid-profile="isValidProfile(profile.data.display)"
              :is-selected-profile="isSelectedProfile(profile.data.jwe)"
              :select-profile="selectProfile"
              :unique-profile-id="profileKey"
              :incomplete-card="isIncompleteCard(profile.data.display)"
              :show-invalid-profiles="showInvalidProfiles"
              :count-invalid-profiles="countInvalidProfiles"
            />
          </div>
        </div>
      </div>

      <!-- desktop -->
      <div class="row mx-auto">
        <div class="d-none d-md-block col-12">
          <b-carousel
            v-if="userInfo.profiles"
            id="carousel1"
            :interval="0"
            img-width="570"
            img-height="250"
            :controls="determineControlsVisibility"
          >
            <b-carousel-slide
              v-for="page in splitInPages(2)"
              :key="page.id"
              img-blank
              img-alt="Identity Providers"
            >
              <div class="row carousel-row align-items-center">
                <div
                  v-for="(profile, profileKey) in page.items"
                  :key="profileKey"
                  class="col-6"
                >
                  <ProfileContainer
                    :original-content="profile.data.display"
                    :profile-data-raw="profile.data.raw"
                    :profile-data-jwe="profile.data.jwe"
                    :is-valid-profile="isValidProfile(profile.data.display)"
                    :is-selected-profile="isSelectedProfile(profile.data.jwe)"
                    :select-profile="selectProfile"
                    :unique-profile-id="profileKey"
                    :incomplete-card="isIncompleteCard(profile.data.display)"
                    :show-invalid-profiles="showInvalidProfiles"
                    :count-invalid-profiles="countInvalidProfiles"
                  />
                </div>
              </div>
            </b-carousel-slide>
          </b-carousel>
        </div>
      </div>
      <LegalMentions
        :client-id="clientId"
        :dp-name="chosenDpName"
        :informations="chosenDpInformations"
        :sp-name="spData.name"
        :sp-dpo-contact="spData.dpo"
        legal-info="GUI_profiles_clicking_terms"
        legal-autoupdate-info="GUI_profiles_clicking_terms"
        :autoupdate-consent-approved="autoUpdateConsent"
        legal-text="GUI_profiles_legal"
      />
      <!-- confirmation form -->
      <b-row class="mx-auto">
        <b-col>
          <!-- Because 2 actions were sent at the same time one on submit and one on the CTA button  -->
          <!--  so now we handle those actions with the submit  -->
          <!--  Ref : https://alligator.io/vuejs/vue-form-handling/  -->
          <form
            v-if="selectedProfile >= 0 && isValidProfile(userInfo.profiles[selectedProfile].display)"
            id="confirmation-form"
            ref="sharingForm"
            name="confirmation-form"
            @submit.prevent="doShare"
          >
            <template v-if="isAutoUpdate && !autoUpdateIsMandatory">
              <b-form-checkbox
                id="autoUdpate-consent-checkbox"
                v-model="autoUpdateConsentValue"
                value="GUI_share_autoupdate_consent_approved"
                unchecked-value="GUI_share_autoupdate_consent_declined"
                @change="updateAutoupdateConsent"
              >
                <!-- eslint-disable vue/no-v-html -->
                <span v-html="$t(autoupdateConsentMessage)" />
                <!-- eslint-enable vue/no-v-html -->
              </b-form-checkbox>
              <br>
            </template>
            <label id="share-profile">
              <!-- This button should not contain any adblockers keywords or else it will be hide
                  somes examples of adBlock keywords : https://gist.github.com/spyesx/42fe84c0ef757d1c38a4
                  Github Ref: https://github.com/solven-eu/mitrust-datasharing/issues/5786  -->
              <b-button
                id="authorize-btn"
                v-observe-visibility="ctaVisibilityChanged"
                name="authorize"
                type="submit"
                class="btn-authorize btn-big"
                :class="{ shared: clickedApproval }"
                variant="primary"
                :disabled="clickedApproval"
              >
                <template v-if="clickedApproval">
                  <Loader
                    color="#FFFFFF"
                    color2="#FFFFFF"
                    color3="#FFFFFF"
                    size="30px"
                  />
                </template>
                <template v-else>
                  {{ fetchCtaMessage }}
                </template>
              </b-button>
            </label>
          </form>
          <div v-else-if="noValidProfiles">
            <CheckBoxCreateAccount
              :checked="createAccountIsChecked"
              @click="openCreateAccountModal()"
            />
            <b-button
              v-if="canGoBackToChooseDp"
              class="btn-big grey-button"
              @click="backToDps"
            >
              {{ $t('message.GUI_not_the_infos_i_want_to_share') }}
            </b-button>
          </div>
          <span v-if="!noValidProfiles && (canGoBackToChooseDp || chosenDpType === 'external')">
            <b-button
              class="back-to-dps"
              variant="link"
              :disabled="clickedApproval"
              @click="backToDps"
            >
              {{ $t('message.GUI_not_the_infos_i_want_to_share') }}
            </b-button>
          </span>
          <span v-if="!noValidProfiles && isSingleDp && chosenDpType !== 'external'">
            <b-button
              class="back-to-credentials"
              variant="link"
              :disabled="clickedApproval"
              @click="backToCredentials"
            >
              {{ $t('message.GUI_retry_with') }} {{ chosenDpName }}
            </b-button>
          </span>
        </b-col>
      </b-row>
      <div
        v-if="showFastForwardIcon"
        class="fast-forward-button-container"
      >
        <a
          class="icon-container"
          @click="scrollToCta"
        >
          <img
            :src="fastForwardIcon"
            :alt="$t('message.GUI_scroll_to_CTA')"
          >
        </a>
      </div>
    </div>
    <CreateAccountModal @success="accountCreatedCallback()" />
    <BaClick v-if="clientId === 'prd_MiTrust_NewDataSourcesProgram_fr'" />
  </div>
  <div v-else>
    <div class="page-loader">
      <Loader
        :title="$t('message.GUI_loading_profiles')"
        variant="logo"
      />
    </div>
  </div>
</template>

<script>
import { BCarousel } from 'bootstrap-vue';

import BaClick from '@/components/BaClick.vue';
import LegalMentions from '@/components/LegalMentions.vue';
import Loader from '@/components/Loader.vue';
import CreateAccountModal from '@/components/Modals/CreateAccountModal.vue';
import CheckBoxCreateAccount from '@/components/Profile/CheckboxCreateAccount.vue';
import ProfileContainer from '@/components/Profile/ProfileContainer.vue';
import i18nManager from '@/tools/i18nManager';
import RequestManager from '@/tools/requestManager';
import Shared from '@/tools/shared';
import StateManager from '@/tools/stateManager';
import urlPrefix from '@/tools/urlPrefix';

// for testing purpose only, usually commented
// import mockProfileBP from '../../../tests/unit/mockProfile/mockProfileBP';

export default {
  name: 'Share',
  components: {
    ProfileContainer,
    CheckBoxCreateAccount,
    CreateAccountModal,
    LegalMentions,
    Loader,
    BCarousel,
    BaClick,
  },
  data() {
    return {
      initialized: false,
      // CSRF have been deactivated for now
      // csrf: this.$route.query._csrf,
      userInfo: {
        profiles: [],
      },
      selectedProfile: -1,
      selectedProfileJwe: undefined,
      // Used to prevent the user to click twice approval
      clickedApproval: false,
      validProfiles: [],
      invalidProfiles: [],
      hideInvalidProfiles: true,
      createAccountIsChecked: false,

      sessionData: {},
      spData: {},

      transientSpState: null,
      appLanguage: undefined,
      chosenDpName: undefined,
      chosenDpType: undefined,
      chosenDpInformations: undefined,
      // auto update
      isAutoUpdate: false,
      credentialsObject: {},
      autoUpdateConsent: false,
      autoUpdateConsentValue: 'GUI_share_autoupdate_consent_approved',
      canGoBackToChooseDp: true,
      noValidProfiles: false,
      redirectUri: undefined,
      isSingleDp: false,
      dataFetchedText: undefined,
      dataFetchedTextAcceptAutoupdate: undefined,
      showFastForwardIcon: false,
    };
  },
  computed: {
    descriptionToShow() {
      return this.autoUpdateConsent ? this.dataFetchedTextAcceptAutoupdate : this.dataFetchedText;
    },
    autoUpdateIsMandatory() {
      // "autoupdate" have few possible values:
      // none: the flow is not autoupdate
      // essential: the flow is a mandatory autoupdate
      // optin: the flow is an autoupdate optin/optout case
      return this.spData.autoUpdateType === 'essential';
    },
    dpId() {
      return this.$route.params.dp_id;
    },
    sharingFlowId() {
      return this.$route.params.sharing_flow_id;
    },
    clientId() {
      return this.$route.params.client_id;
    },
    profileShared() {
      return StateManager.get('profileShared');
    },
    i18nLanguageReady() {
      return StateManager.get('i18n/ready');
    },
    fetchCtaMessage() {
      return i18nManager.replaceInternalI18nVariables(
        this.$t(`message.GUI${this.autoUpdateConsent ? '_autoupdate' : ''}_share_cta`),
        { SP_NAME: this.spData.name },
      );
    },
    autoupdateConsentMessage() {
      return `message.${this.autoUpdateConsentValue}`;
    },
    currentLanguage() {
      return StateManager.get('i18n/i18n_language');
    },
    countInvalidProfiles() {
      return this.invalidProfiles.length;
    },
    determineControlsVisibility() {
      let display = false;
      if (this.validProfiles.length >= 3) display = true;
      else if (this.invalidProfiles.length && this.validProfiles.length + 1 >= 3) display = true;
      return display;
    },
    oauthAuthorizeUrl() {
      // TODO Must drop this function
      // TODO We must save the oAuth URL received on /make_session and re-use it here
      // ! forward parameter is to be received as a boolean
      return `${urlPrefix.get()}/oauth/authorize?${this.oauth2Suffix}`;
    },
    oauth2Suffix() {
      console.log('client_id in oauth2Suffix:', this.clientId);
      return (
        `sharing_flow_id=${
          encodeURIComponent(this.sharingFlowId)
        }&state=${
          encodeURIComponent(this.transientSpState)
        }&client_id=${
          encodeURIComponent(this.clientId)
        }&forward=false`
      );
    },
    fastForwardIcon() {
      return Shared.generateCdnUrl('v2/icon/icon_down.arrow.svg');
    },

  },
  created() {
    // List of transient variables that must persist after a clearApplicationStorages
    this.sessionData = StateManager.get('sessionsData')[this.sharingFlowId];
    const { chosenDp } = this.sessionData;
    this.chosenDpName = chosenDp.name;
    this.chosenDpType = chosenDp.type;
    this.chosenDpInformations = chosenDp.scope_i18n;
    this.canGoBackToChooseDp = !StateManager.get('skipChooseDp');
    // compute this variable at creation of the component, no need for it to be a (responsive) computed property
    // https://github.com/solven-eu/mitrust-datasharing/issues/6602
    const { dps } = Shared.getServiceProviderData(this.clientId);
    this.isSingleDp = dps ? dps.length === 1 : false;
    // cache these values to avoid problems when clearApplicationStorages
    // https://github.com/solven-eu/mitrust-datasharing/issues/6638
    // pass this.chosenDpName as parameter for easier tests & better logic
    // https://github.com/solven-eu/mitrust-datasharing/issues/6868
    this.dataFetchedText = this.dataFetchedDescription('GUI_data_fetched_description', this.chosenDpName);
    this.dataFetchedTextAcceptAutoupdate = this.dataFetchedDescription(
      'GUI_data_fetched_description_accepted',
      this.chosenDpName,
    );
    // end of list
  },
  beforeDestroy() {
    // when leaving Share Profile page, we always want to remove encrypted credentials from the store
    // remove profiles too => https://github.com/cormoran-io/mitrust-backend/issues/4071
    // particularly useful in case of browser Back
    StateManager.do('removeCredentials');
    StateManager.set('profilesToShow', undefined);
  },
  beforeMount() {
    // those variable are transient because they will removed when we shared the profile
    // therefore no computed using those Transient data can be used
    this.spData = StateManager.get('serviceProviders')[this.clientId];
    this.transientSpState = this.sessionData.state;

    this.isAutoUpdate = this.spData.isAutoUpdate;
    this.autoUpdateConsent = this.isAutoUpdate;
    // Set the step of the breadcrumb to step 3
    StateManager.set('breadcrumbStep', 3);
    StateManager.set('breadcrumbStepTitle', this.$t('message.GUI_header_title_share-profile'));

    // REMOVE CREDENTIALS WHEN DISPLAYING RESULTS only in case it's not AutoUpdate or AutoVerif
    // otherwise, encryptedCredentials will be removed as part of cleaning routing
    // right after submitting the form by Shared.clearApplicationStorages
    // OR on beforeDestroy this component
    if (!this.isAutoUpdate) {
      StateManager.do('removeCredentials');
    } else {
      this.fetchCredentialsObject();
    }
    if (Shared.sharingFlowDataIntegrityCheck(this.sharingFlowId)) {
      if (!this.sessionData.sharingSession) {
        console.log('missing', 'sharing_session');
        const error = new Error(`sharing_session is missing for SFID: ${this.sharingFlowId}`);
        Shared.goToErrorPage(this, this.dpId, 'internal', error);
      } else if (!this.transientSpState) {
        // Required to fetch profiles to be shared
        console.log('missing', 'state');
        const error = new Error(`state is missing for SFID: ${this.sharingFlowId}`);
        Shared.goToErrorPage(this, this.dpId, 'internal', error);
      }
      // CSRF have been deactivated for now
      // if (this.csrf) {
      //   console.log('csrf', this.csrf);
      //   urlForDetails += `&_csrf=${this.csrf}`;
      // } else {
      //   this.vueStatus = "Missing 'CSRF'";
      //   this.backToDps();
      //   return;
      // }

      const profilesToShow = StateManager.get('profilesToShow');
      if (profilesToShow) {
        this.initialized = true;
        // we get profiles from The Store and need to select the first one
        this.userInfo.profiles = profilesToShow;

        // for testing purpose only, usually commented
        // this.userInfo.profiles.push(mockProfileBP.profiles[0]);

        this.userInfo.profiles.forEach((profile) => {
          if (this.isValidProfile(profile.display)) {
            console.debug('valid profile found', Object.keys(profile.display.content));
            this.validProfiles.push(profile);
          } else {
            const invalidReasonsItemToString = profile.display.invalid_reasons.join(', ');
            const invalidReasonsToLog = `Invalid profile found, reason: ${invalidReasonsItemToString}`;
            Shared.logStatus(invalidReasonsToLog, this.dpId);
            console.debug(invalidReasonsToLog);
            if ('missing_required' in profile.display) {
              const missingRequiredItemsToString = profile.display.missing_required.join(', ');
              const missingRequiredToLog = `Invalid profile found, Missing required: ${missingRequiredItemsToString}`;
              Shared.logStatus(missingRequiredToLog, this.dpId);
              console.debug(missingRequiredToLog);
            }
            this.invalidProfiles.push(profile);
          }
        });

        // now we check to see if profiles are valid via isValidProfile()
        for (let i = 0; i < this.userInfo.profiles.length; i += 1) {
          if (this.isValidProfile(this.userInfo.profiles[i].display)) {
            this.selectProfile(this.userInfo.profiles[i].jwe);
            // breaking to select only the first profile
            break;
          } else {
            // that means no profiles are valid
          }
        }

        // count invalid profiles. Evaluate this only once, on component creation
        this.noValidProfiles = this.countInvalidProfiles === this.userInfo.profiles.length;
      } else {
        // We received no profile. Will throw an error
        this.userInfo.profiles = [];
      }
      const countProfiles = this.userInfo.profiles.length;
      const countCompleteProfiles = countProfiles - this.countInvalidProfiles;

      // if there are no profiles, go to error
      if (this.noProfiles(this.userInfo.profiles)) {
        const error = new Error(`No profiles available for SFID: ${this.sharingFlowId}`);
        Shared.goToErrorPage(this, this.dpId, 'no_profiles', error);
      } else {
        Shared.logStatus(`Profiles: Complete ${countCompleteProfiles} / Total ${countProfiles}`);
      }
      // track page view
      Shared.matomoTMSManager({
        last_dp_id: this.dpId,
        nb_profiles: countProfiles,
        // https://github.com/solven-eu/mitrust-datasharing/issues/6313
        nb_complete_profiles: countCompleteProfiles,
        nb_incomplete_profiles: this.countInvalidProfiles,
      });
    } else {
      // we miss some sharingFlowId related, essentials data
      const error = new Error(`Sharing Flow Data Integrity Check Failure for SFID: ${this.sharingFlowId}`);
      Shared.goToErrorPage(this, this.dpId, 'internal', error);
    }
  },
  mounted() {
  // the page is ready, emit the event to Sharing.vue (currently used to display the Header and BackToSpButton)
    this.$emit('show-page');
  },
  methods: {
    dataFetchedDescription(key, chosenDpName) {
      const i18nKey = i18nManager.transformMessageForAutoupdate(key, this);
      return i18nManager.replaceInternalI18nVariables(
        // deprecate IDP_NAME in favor of DP_NAME, let them co-exist for a while
        this.$t(`message.${i18nKey}`), { IDP_NAME: chosenDpName, DP_NAME: chosenDpName },
      );
    },
    updateAutoupdateConsent(state) {
      this.autoUpdateConsent = state.indexOf('approved') >= 0;
    },
    fetchCredentialsObject() {
      const encryptedCredentials = StateManager.get('transientEncryptedCredentials');
      this.credentialsObject = JSON.stringify(encryptedCredentials);
    },
    isSelectedProfile(profileJwe) {
      if (this.selectedProfile >= 0) {
        return this.userInfo.profiles[this.selectedProfile].jwe === profileJwe;
      }
      return false;
    },
    noProfiles(profiles) {
      return !profiles || profiles.length === 0;
    },
    openCreateAccountModal() {
      this.$bvModal.show('create-account-modal');
    },
    accountCreatedCallback() {
      this.$bvModal.hide('create-account-modal');
      this.createAccountIsChecked = true;
    },
    backToDps() {
      Shared.logStatus('Choose another Provider - ( Sharing Page )');
      this.$router.push({
        name: 'ChooseDp',
        params: {
          client_id: this.clientId,
          sharing_flow_id: this.sharingFlowId,
          language: this.currentLanguage,
        },
      });
    },
    backToCredentials() {
      this.$router.push({
        name: 'EnterCredentials',
        params: {
          client_id: this.clientId,
          sharing_flow_id: this.sharingFlowId,
          language: this.currentLanguage,
          improvement_program: false,
          dp_id: this.dpId,
        },
      });
    },
    selectProfile(profileJwe) {
      for (let i = 0; i < this.userInfo.profiles.length; i += 1) {
        if (this.userInfo.profiles[i].jwe === profileJwe) {
          this.selectedProfile = i;
          console.log('Selected profile: ', Object.keys(this.userInfo.profiles[i].display.content));
          this.selectedProfileJwe = profileJwe;
          break;
        }
      }
    },
    /**
     *
     * @description  A profile is valid if it has no invalid_reasons:
     *   invalid_reasons hold the reason of the failure:
     *   missing_essential if missing an essential claim,
     *   too_big if too big
     * @see https://github.com/solven-eu/mitrust-datasharing/issues/7081#issuecomment-749485311
     * @see modules\dp-mvc\src\main\java\io\mitrust\dp\form\SfProfileHelper.java
     * @param {object} profile - The profile object
     * @return {boolean} TRUE if a profile is valid
     *
     * @param profile
     */
    isValidProfile(profile) {
      return !Array.isArray(profile.invalid_reasons) || profile.invalid_reasons.length === 0;
    },
    splitInPages(itemsPerPage) {
      return Shared.splitInPages(itemsPerPage, this.validProfiles, this.invalidProfiles, this.hideInvalidProfiles);
    },
    showAllData(data) {
      return Shared.showAllData(data, this.invalidProfiles, this.hideInvalidProfiles);
    },
    doShare() {
      Shared.logStatus('sharing CTA clicked');
      StateManager.set('profileShared', true);
      this.clickedApproval = true;
      // Matomo track goal https://github.com/cormoran-io/mitrust-backend/issues/1068
      // https://github.com/cormoran-io/mitrust-backend/issues/4028
      Shared.matomoTMSManager({ event: 'sharing_click' });
      // submit the form via $ref.sharingForm.submit() to avoid conflicts between form's submit and action
      // adding a delay of 500ms should help
      // it is an attempt at resolving https://github.com/cormoran-io/mitrust-backend/issues/2079
      // src: https://matomo.org/faq/troubleshooting/#faq_96

      // craft formData to allow the submission of the form via axios
      // https://github.com/solven-eu/mitrust-datasharing/issues/6400
      const formData = new FormData();
      formData.append('user_oauth_approval', 'true');
      if (this.selectedProfileJwe) {
        // send profile_jwe in case we have one...
        formData.append('profile_jwe', this.selectedProfileJwe);
      } else {
        // ..else send profile_index
        formData.append('profile_index', this.selectedProfile);
      }

      if (this.isAutoUpdate) {
        if (this.autoUpdateConsent) {
          formData.append('credentials_json', this.credentialsObject);
          formData.append('credentials_dp_id', this.dpId);
        }
      }
      RequestManager.apiCall(
        'post',
        this.oauthAuthorizeUrl,
        formData,
        { withCredentials: true },
      )
        .then(this.saveRedirectUri)
        .then(() => Shared.setTransientSessionInfo('already_shared'))
        .then(() => Shared.clearApplicationStorages('User shared his data with the SP'))
        .then(() => Shared.assignLocation(this.redirectUri))
        .catch((error) => {
          Shared.goToErrorPage(this, this.dpId, 'internal', error);
        });
    },
    saveRedirectUri(response) {
      this.redirectUri = response.data.data.redirect_uri;
      return Promise.resolve();
    },
    showInvalidProfiles() {
      this.validProfiles = this.validProfiles.concat(this.invalidProfiles);
      this.invalidProfiles = [];
      this.hideInvalidProfiles = false;
    },
    isIncompleteCard(profileDisplay) {
      return !!(profileDisplay && profileDisplay.incomplete);
    },
    ctaVisibilityChanged(isVisible) {
      if (!isVisible) {
        Shared.addHashToRoute(this, '#profiles');
      }
      this.showFastForwardIcon = !isVisible;
    },
    scrollToCta() {
      Shared.addHashToRoute(this, '#share-profile');
      Shared.logStatus('The user clicked on Scroll to CTA button');
    },
  },
};
</script>

<style scoped lang="scss">
.back-to-dps, .back-to-credentials {
    text-decoration: underline;
    margin-bottom: 5rem;
  }

  ul {
    padding: 0 !important;
  }

  .shared {
    width: 75vw;
    animation: shared 0.3s ease-out forwards;
  }

  /deep/ {
    .carousel-control-prev {
      left: -15%;
    }

    .carousel-control-next {
      right: -15%;
    }

    .carousel-caption {
      left: 3px;
      right: 3px;

      .row {
        &.carousel-row {
          height: 100%;
        }

        .col-6 {
          &:only-child {
            transform: translate(50%);
          }
        }
      }
    }
  }

  #confirmation-form {
    // https://github.com/cormoran-io/mitrust-backend/issues/1752#issuecomment-508504300
    margin-top: 0.2rem;
  }

  @keyframes shared {
    from {
      width: 75vw;
    }
    to {
      width: 300px;
    }
  }

  .ButtonCreateAccountWrapper {
    margin-bottom: 20px;
  }
  .grey-button {
    background-color: #929292 !important;
    color: #FFF !important;
    &:hover {
      background-color: #868686 !important;
    }
  }
  .fast-forward-button-container {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    padding-bottom: 20px;
    padding-top: 50px;
    background: linear-gradient(0deg, rgba(255,255,255,1) 60%, rgba(255,255,255,0) 100%);
    .icon-container {
      border-radius: 100%;
      background-color: #35c5f3;
      display: block;
      cursor: pointer;
      margin: 0 auto;
      width: 50px;
      height: 50px;
      line-height: 50px;
      img {
        width: 60%;
      }
    }
  }
</style>
