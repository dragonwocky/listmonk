<template>
  <section class="campaign">
    <header class="columns page-header">
      <div class="column is-6">
        <p v-if="isEditing && data.status" class="tags">
          <b-tag v-if="isEditing" :class="data.status">
            {{ $t(`campaigns.status.${data.status}`) }}
          </b-tag>
          <b-tag v-if="data.type === 'optin'" :class="data.type">
            {{ $t('lists.optin') }}
          </b-tag>
          <span v-if="isEditing" class="has-text-grey-light is-size-7" :data-campaign-id="data.id">
            {{ $t('globals.fields.id') }}: <copy-text :text="`${data.id}`" />
            {{ $t('globals.fields.uuid') }}: <copy-text :text="data.uuid" />
          </span>
        </p>
        <h4 v-if="isEditing" class="title is-4">
          {{ data.name }}
        </h4>
        <h4 v-else class="title is-4">
          {{ $t('campaigns.newCampaign') }}
        </h4>
      </div>

      <div class="column is-6">
        <div v-if="canManage" class="buttons">
          <b-field grouped v-if="isEditing && canEdit">
            <b-field expanded>
              <b-button expanded @click="() => onSubmit('update')" :loading="loading.campaigns" type="is-primary"
                icon-left="content-save-outline" data-cy="btn-save" aria-keyshortcuts="ctrl+s">
                <span class="has-kbd">{{ $t('globals.buttons.saveChanges') }} <span class="kbd">Ctrl+S</span></span>
              </b-button>
            </b-field>
            <b-field expanded v-if="canStart">
              <b-button expanded @click="startCampaign" :loading="loading.campaigns" type="is-primary"
                icon-left="rocket-launch-outline" data-cy="btn-start">
                {{ $t('campaigns.start') }}
              </b-button>
            </b-field>
            <b-field expanded v-if="canSchedule">
              <b-button expanded @click="startCampaign" :loading="loading.campaigns" type="is-primary"
                icon-left="clock-start" data-cy="btn-schedule">
                {{ $t('campaigns.schedule') }}
              </b-button>
            </b-field>
            <b-field expanded v-if="canUnSchedule">
              <b-button expanded @click="$utils.confirm(null, unscheduleCampaign)" :loading="loading.campaigns"
                type="is-primary" icon-left="clock-start" data-cy="btn-unschedule">
                {{ $t('campaigns.unSchedule') }}
              </b-button>
            </b-field>
          </b-field>
        </div>
      </div>
    </header>

    <b-loading :active="loading.campaigns" />

    <b-tabs type="is-boxed" :animated="false" v-model="activeTab" @input="onTab">
      <b-tab-item :label="$tc('globals.terms.campaign')" label-position="on-border" value="campaign"
        icon="rocket-launch-outline">
        <section class="wrap">
          <div class="columns">
            <div class="column is-7">
              <form @submit.prevent="() => onSubmit(isNew ? 'create' : 'update')">
                <b-field :label="$t('globals.fields.name')" label-position="on-border">
                  <b-input :maxlength="200" :ref="'focus'" v-model="form.name" name="name" :disabled="!canEdit"
                    :placeholder="$t('globals.fields.name')" required autofocus />
                </b-field>

                <b-field :label="$t('campaigns.subject')" label-position="on-border">
                  <b-input :maxlength="5000" v-model="form.subject" name="subject" :disabled="!canEdit"
                    :placeholder="$t('campaigns.subject')" required />
                </b-field>

                <b-field :label="$t('campaigns.fromAddress')" label-position="on-border">
                  <b-input :maxlength="200" v-model="form.fromEmail" name="from_email" :disabled="!canEdit"
                    :placeholder="$t('campaigns.fromAddressPlaceholder')" required />
                </b-field>

                <list-selector v-model="form.lists" :selected="form.lists" :all="lists.results" :disabled="!canEdit"
                  :label="$t('globals.terms.lists')" :placeholder="$t('campaigns.sendToLists')" />

                <div class="columns">
                  <div class="column is-6">
                    <b-field :label="$tc('globals.terms.messenger')" label-position="on-border">
                      <b-select :placeholder="$tc('globals.terms.messenger')" v-model="form.messenger" name="messenger"
                        :disabled="!canEdit" @input="onMessengerChange" required expanded>
                        <optgroup label="email">
                          <option v-for="m in emailMessengers" :value="m" :key="m">{{ m.name }}</option>
                        </optgroup>
                        <option v-for="m in otherMessengers" :value="m" :key="m">{{ m.name }}</option>
                      </b-select>
                    </b-field>
                  </div>
                  <div class="column is-6">
                    <b-field :label="$t('campaigns.format')" label-position="on-border" class="mr-4 mb-0">
                      <b-select v-model="form.content.contentType" :disabled="!canEdit || isEditing" value="richtext"
                        expanded>
                        <option v-for="(name, f) in contentTypes" :key="f" name="format" :value="f"
                          :data-cy="`check-${f}`">
                          {{ name }}
                        </option>
                      </b-select>
                    </b-field>
                  </div>
                </div>

                <b-field :label="$t('globals.terms.tags')" label-position="on-border">
                  <b-taginput v-model="form.tags" name="tags" :disabled="!canEdit" ellipsis icon="tag-outline"
                    :placeholder="$t('globals.terms.tags')" />
                </b-field>
                <hr />

                <div class="columns">
                  <div class="column is-4">
                    <b-field :label="$t('campaigns.sendLater')" data-cy="btn-send-later">
                      <b-switch v-model="form.sendLater" :disabled="!canEdit" />
                    </b-field>
                  </div>
                  <div class="column">
                    <br />
                    <b-field v-if="form.sendLater" data-cy="send_at"
                      :message="form.sendAtDate ? $utils.duration(Date(), form.sendAtDate) : ''">
                      <b-datetimepicker v-model="form.sendAtDate" :disabled="!canEdit" required editable mobile-native
                        position="is-top-right" :placeholder="$t('campaigns.dateAndTime')" icon="calendar-clock"
                        :timepicker="{ hourFormat: '24' }" :datetime-formatter="formatDateTime"
                        horizontal-time-picker />
                    </b-field>
                  </div>
                </div>

                <div>
                  <p class="has-text-right">
                    <a href="#" @click.prevent="onShowHeaders" data-cy="btn-headers">
                      <b-icon icon="plus" />{{ $t('settings.smtp.setCustomHeaders') }}
                    </a>
                  </p>
                  <b-field v-if="form.headersStr !== '[]' || isHeadersVisible" label-position="on-border"
                    :message="$t('campaigns.customHeadersHelp')">
                    <b-input v-model="form.headersStr" name="headers" type="textarea"
                      placeholder="[{&quot;X-Custom&quot;: &quot;value&quot;}, {&quot;X-Custom2&quot;: &quot;value&quot;}]"
                      :disabled="!canEdit" />
                  </b-field>
                </div>
                <hr />

                <b-field v-if="isNew">
                  <b-button native-type="submit" type="is-primary" :loading="loading.campaigns" data-cy="btn-continue">
                    {{ $t('campaigns.continue') }}
                  </b-button>
                </b-field>
              </form>
            </div>
            <div v-if="canManage" class="column is-4 is-offset-1">
              <br />
              <div class="box">
                <h3 class="title is-size-6">
                  {{ $t('campaigns.sendTest') }}
                </h3>
                <b-field :message="$t('campaigns.sendTestHelp')">
                  <b-taginput v-model="form.testEmails" :before-adding="$utils.validateEmail" :disabled="isNew" ellipsis
                    icon="email-outline" :placeholder="$t('campaigns.testEmails')" />
                </b-field>
                <b-field>
                  <b-button @click="() => onSubmit('test')" :loading="loading.campaigns" :disabled="isNew"
                    type="is-primary" icon-left="email-outline">
                    {{ $t('campaigns.send') }}
                  </b-button>
                </b-field>
              </div>
            </div>
          </div>
        </section>
      </b-tab-item><!-- campaign -->

      <b-tab-item :label="$t('campaigns.content')" icon="text" :disabled="isNew" value="content">
        <editor v-if="data.id" v-model="form.content" :id="data.id" :title="data.name" :disabled="!canEdit"
          :templates="templates" :content-types="contentTypes" />

        <div class="columns">
          <div class="column is-6">
            <p v-if="!isAttachFieldVisible" class="is-size-6 has-text-grey">
              <a href="#" @click.prevent="onShowAttachField()" data-cy="btn-attach">
                <b-icon icon="file-upload-outline" size="is-small" />
                {{ $t('campaigns.addAttachments') }}
              </a>
            </p>

            <b-field v-if="isAttachFieldVisible" :label="$t('campaigns.attachments')" label-position="on-border"
              expanded data-cy="media">
              <b-taginput v-model="form.media" name="media" ellipsis icon="tag-outline" ref="media" field="filename"
                @focus="onOpenAttach" :disabled="!canEdit" />
            </b-field>
          </div>
          <div class="column has-text-right">
            <a href="https://listmonk.app/docs/templating/#template-expressions" target="_blank"
              rel="noopener noreferer">
              <b-icon icon="code" /> {{ $t('campaigns.templatingRef') }}</a>
            <span v-if="canEdit && form.content.contentType !== 'plain'" class="is-size-6 has-text-grey ml-6">
              <a v-if="form.altbody === null" href="#" @click.prevent="onAddAltBody">
                <b-icon icon="text" size="is-small" /> {{ $t('campaigns.addAltText') }}
              </a>
              <a v-else href="#" @click.prevent="$utils.confirm(null, onRemoveAltBody)">
                <b-icon icon="trash-can-outline" size="is-small" />
                {{ $t('campaigns.removeAltText') }}
              </a>
            </span>
          </div>
        </div>

        <div v-if="canEdit && form.content.contentType !== 'plain'" class="alt-body">
          <b-input v-if="form.altbody !== null" v-model="form.altbody" type="textarea" :disabled="!canEdit" />
        </div>
      </b-tab-item><!-- content -->

      <b-tab-item :label="$t('campaigns.archive')" icon="newspaper-variant-outline" value="archive" :disabled="isNew">
        <section class="wrap">
          <div class="columns">
            <div class="column is-4">
              <b-field :label="$t('campaigns.archiveEnable')" data-cy="btn-archive"
                :message="$t('campaigns.archiveHelp')">
                <div class="columns">
                  <div class="column">
                    <b-switch data-cy="btn-archive" v-model="form.archive" :disabled="!canArchive" />
                  </div>
                  <div class="column is-12">
                    <a :href="`${serverConfig.root_url}/archive/${data.uuid}`" target="_blank" rel="noopener noreferer"
                      :class="{ 'has-text-grey-light': !form.archive }" aria-label="$t('campaigns.archive')">
                      <b-icon icon="link-variant" />
                    </a>
                  </div>
                </div>
              </b-field>
            </div>
            <div class="column is-8">
              <b-field grouped position="is-right">
                <b-field v-if="!canEdit && canArchive">
                  <b-button @click="onUpdateCampaignArchive" :loading="loading.campaigns" type="is-primary"
                    icon-left="content-save-outline" data-cy="btn-save">
                    {{ $t('globals.buttons.saveChanges') }}
                  </b-button>
                </b-field>
              </b-field>
            </div>
          </div>

          <div class="columns">
            <div class="column is-6">
              <b-field :label="$tc('globals.terms.template')" label-position="on-border">
                <b-select :placeholder="$tc('globals.terms.template')" v-model="form.archiveTemplateId" name="template"
                  :disabled="!canArchive || !form.archive || form.content.contentType === 'visual'" required>
                  <template v-for="t in templates">
                    <option v-if="t.type === 'campaign'" :value="t.id" :key="t.id">
                      {{ t.name }}
                    </option>
                  </template>
                </b-select>
              </b-field>
            </div>

            <div class="column is-6">
              <b-field grouped position="is-right">
                <b-field v-if="form.archive && (!this.form.archiveMetaStr || this.form.archiveMetaStr === '{}')">
                  <a class="button is-primary" href="#" @click.prevent="onFillArchiveMeta" aria-label="{}"><b-icon
                      icon="code" /></a>
                </b-field>
                <b-field v-if="form.archive">
                  <b-button @click="onToggleArchivePreview" type="is-primary" icon-left="file-find-outline"
                    data-cy="btn-preview">
                    {{ $t('campaigns.preview') }}
                  </b-button>
                </b-field>
              </b-field>
            </div>
          </div>
          <b-field>
            <b-field :label="$t('campaigns.archiveSlug')" label-position="on-border"
              :message="$t('campaigns.archiveSlugHelp')">
              <b-input :maxlength="200" :ref="'focus'" v-model="form.archiveSlug" name="archive_slug"
                data-cy="archive-slug" :disabled="!canArchive || !form.archive" />
            </b-field>
          </b-field>
          <b-field :label="$t('campaigns.archiveMeta')" :message="$t('campaigns.archiveMetaHelp')"
            label-position="on-border">
            <b-input v-model="form.archiveMetaStr" name="archive_meta" type="textarea" data-cy="archive-meta"
              :disabled="!canArchive || !form.archive" rows="20" />
          </b-field>
        </section>
      </b-tab-item><!-- archive -->
    </b-tabs>

    <b-modal scroll="keep" :aria-modal="true" :active.sync="isAttachModalOpen" :width="900">
      <div class="modal-card content" style="width: auto">
        <section expanded class="modal-card-body">
          <media is-modal @selected="onAttachSelect" />
        </section>
      </div>
    </b-modal>

    <campaign-preview v-if="isPreviewingArchive" @close="onToggleArchivePreview" type="campaign" :id="data.id"
      :archive-meta="form.archiveMetaStr" :title="data.title" :content-type="data.contentType"
      :template-id="form.archiveTemplateId" is-post is-archive />
  </section>
</template>

<script>
import dayjs from 'dayjs';
import htmlToPlainText from 'textversionjs';
import Vue from 'vue';
import { mapState } from 'vuex';

import CopyText from '../components/CopyText.vue';
import Editor from '../components/Editor.vue';
import ListSelector from '../components/ListSelector.vue';
import Media from './Media.vue';
import CampaignPreview from '../components/CampaignPreview.vue';

export default Vue.extend({
  components: {
    ListSelector,
    Editor,
    Media,
    CopyText,
    CampaignPreview,
  },

  data() {
    return {
      contentTypes: Object.freeze({
        richtext: this.$t('campaigns.richText'),
        html: this.$t('campaigns.rawHTML'),
        markdown: this.$t('campaigns.markdown'),
        plain: this.$t('campaigns.plainText'),
        visual: this.$t('campaigns.visual'),
      }),

      isNew: false,
      isEditing: false,
      isHeadersVisible: false,
      isAttachFieldVisible: false,
      isAttachModalOpen: false,
      isPreviewingArchive: false,
      activeTab: 'campaign',
      prevMessenger: {},

      data: {},

      // IDs from ?list_id query param.
      selListIDs: [],

      // Binds form input values.
      form: {
        archiveSlug: null,
        name: '',
        subject: '',
        fromEmail: '',
        headersStr: '[]',
        headers: [],
        messenger: {},
        lists: [],
        tags: [],
        sendAt: null,
        content: {
          contentType: 'richtext',
          body: '',
          bodySource: null,
          templateId: null,
        },
        altbody: null,
        media: [],

        // Parsed Date() version of send_at from the API.
        sendAtDate: null,
        sendLater: false,
        archive: false,
        archiveMetaStr: '{}',
        archiveMeta: {},
        testEmails: [],
      },
    };
  },

  methods: {
    formatDateTime(s) {
      return dayjs(s).format('YYYY-MM-DD HH:mm');
    },

    onToggleArchivePreview() {
      this.isPreviewingArchive = !this.isPreviewingArchive;
    },

    onAddAltBody() {
      this.form.altbody = htmlToPlainText(this.form.content.body);
    },

    onRemoveAltBody() {
      this.form.altbody = null;
    },

    onShowHeaders() {
      this.isHeadersVisible = !this.isHeadersVisible;
    },

    onShowAttachField() {
      this.isAttachFieldVisible = true;
      this.$nextTick(() => {
        this.$refs.media.focus();
      });
    },

    onOpenAttach() {
      this.isAttachModalOpen = true;
    },

    onAttachSelect(o) {
      if (this.form.media.some((m) => m.id === o.id)) {
        return;
      }

      this.form.media.push(o);
    },

    isUnsaved() {
      return this.data.body !== this.form.content.body
        || this.data.contentType !== this.form.content.contentType;
    },

    onTab(tab) {
      if (tab === 'content' && window.tinymce && window.tinymce.editors.length > 0) {
        this.$nextTick(() => {
          window.tinymce.editors[0].focus();
        });
      }

      // this.$router.replace({ hash: `#${tab}` });
      window.history.replaceState({}, '', `#${tab}`);
    },

    onFillArchiveMeta() {
      const archiveStr = `{"email": "email@domain.com", "name": "${this.$t('globals.fields.name')}", "attribs": {}}`;
      this.form.archiveMetaStr = this.$utils.getPref('campaign.archiveMetaStr') || JSON.stringify(JSON.parse(archiveStr), null, 4);
    },

    onSubmit(typ) {
      // Validate custom JSON headers.
      if (this.form.headersStr && this.form.headersStr !== '[]') {
        try {
          this.form.headers = JSON.parse(this.form.headersStr);
        } catch (e) {
          this.$utils.toast(e.toString(), 'is-danger');
          return;
        }
      } else {
        this.form.headers = [];
      }

      // Validate archive JSON body.
      if (this.form.archive && this.form.archiveMetaStr) {
        try {
          this.form.archiveMeta = JSON.parse(this.form.archiveMetaStr);
        } catch (e) {
          this.$utils.toast(e.toString(), 'is-danger');
          return;
        }
      }

      switch (typ) {
        case 'create':
          this.createCampaign();
          break;
        case 'test':
          this.sendTest();
          break;
        default:
          this.updateCampaign();
          break;
      }
    },

    onMessengerChange() {
      // If sending from the default from address,
      // update from address to match updated messenger.
      const prevEmail = this.prevMessenger.from_email || '';
      const currEmail = this.form.fromEmail || '';
      const nextEmail = this.form.messenger.from_email;
      if (currEmail.trim() === prevEmail.trim() && nextEmail) {
        this.form.fromEmail = nextEmail;
      }
      this.prevMessenger = this.form.messenger;
    },

    getCampaign(id) {
      return this.$api.getCampaign(id).then((data) => {
        this.data = data;
        this.form = {
          ...this.form,
          ...data,
          messenger: this.emailMessengers.find(({ name }) => name === data.messenger),
          headersStr: JSON.stringify(data.headers, null, 4),
          archiveMetaStr: data.archiveMeta ? JSON.stringify(data.archiveMeta, null, 4) : '{}',

          // The structure that is populated by editor input event.
          content: {
            contentType: data.contentType,
            body: data.body,
            bodySource: data.bodySource,
            templateId: data.templateId,
          },
        };
        this.form.messenger ||= this.emailMessengers[0];
        this.form.from_email ||= this.form.messenger.from_email;
        this.isAttachFieldVisible = this.form.media.length > 0;

        this.form.media = this.form.media.map((f) => {
          if (!f.id) {
            return { ...f, filename: `❌ ${f.filename}` };
          }
          return f;
        });
      });
    },

    sendTest() {
      const data = {
        id: this.data.id,
        name: this.form.name,
        subject: this.form.subject,
        lists: this.form.lists.map((l) => l.id),
        from_email: this.form.fromEmail,
        messenger: this.form.messenger.name,
        type: 'regular',
        headers: this.form.headers,
        tags: this.form.tags,
        template_id: this.form.content.templateId,
        content_type: this.form.content.contentType,
        body: this.form.content.body,
        altbody: this.form.content.contentType !== 'plain' ? this.form.altbody : null,
        subscribers: this.form.testEmails,
        media: this.form.media.map((m) => m.id),
      };

      this.$api.testCampaign(data).then(() => {
        this.$utils.toast(this.$t('campaigns.testSent'));
      });
      return false;
    },

    createCampaign() {
      const data = {
        archiveSlug: this.form.subject,
        name: this.form.name,
        subject: this.form.subject,
        lists: this.form.lists.map((l) => l.id),
        from_email: this.form.fromEmail,
        content_type: this.form.content.contentType,
        messenger: this.form.messenger.name,
        type: 'regular',
        tags: this.form.tags,
        send_at: this.form.sendLater ? this.form.sendAtDate : null,
        headers: this.form.headers,
        media: this.form.media.map((m) => m.id),
      };

      this.$api.createCampaign(data).then((d) => {
        this.$router.push({ name: 'campaign', hash: '#content', params: { id: d.id } });
      });
      return false;
    },

    async updateCampaign(typ) {
      const data = {
        archive_slug: this.form.archiveSlug,
        name: this.form.name,
        subject: this.form.subject,
        lists: this.form.lists.map((l) => l.id),
        from_email: this.form.fromEmail,
        messenger: this.form.messenger.name,
        type: 'regular',
        tags: this.form.tags,
        send_at: this.form.sendLater ? this.form.sendAtDate : null,
        headers: this.form.headers,
        template_id: this.form.content.templateId,
        content_type: this.form.content.contentType,
        body: this.form.content.body,
        body_source: this.form.content.bodySource,
        altbody: this.form.content.contentType !== 'plain' ? this.form.altbody : null,
        archive: this.form.archive,
        archive_template_id: this.form.archiveTemplateId,
        archive_meta: this.form.archiveMeta,
        media: this.form.media.map((m) => m.id),
      };

      let typMsg = 'globals.messages.updated';
      if (typ === 'start') {
        typMsg = 'campaigns.started';
      }

      if (!this.form.sendAtDate) {
        this.form.sendLater = false;
      }

      // This promise is used by startCampaign to first save before starting.
      return new Promise((resolve) => {
        this.$api.updateCampaign(this.data.id, data).then((d) => {
          this.data = d;
          this.form.archiveSlug = d.archiveSlug;

          this.$utils.toast(this.$t(typMsg, { name: d.name }));
          resolve();
        });
      });
    },

    onUpdateCampaignArchive() {
      if (this.isEditing && this.canEdit) {
        return;
      }

      const data = {
        archive: this.form.archive,
        archive_template_id: this.form.archiveTemplateId,
        archive_meta: JSON.parse(this.form.archiveMetaStr),
        archive_slug: this.form.archiveSlug,
      };

      this.$api.updateCampaignArchive(this.data.id, data).then((d) => {
        this.form.archiveSlug = d.archiveSlug;
      });
    },

    // Starts or schedule a campaign.
    startCampaign() {
      if (!this.canStart && !this.canSchedule) {
        return;
      }

      this.$utils.confirm(
        null,
        () => {
          // First save the campaign.
          this.updateCampaign().then(() => {
            // Then start/schedule it.
            let status = '';
            if (this.canStart) {
              status = 'running';
            } else if (this.canSchedule) {
              status = 'scheduled';
            } else {
              return;
            }

            this.$api.changeCampaignStatus(this.data.id, status).then(() => {
              this.$router.push({ name: 'campaigns' });
            });
          });
        },
      );
    },

    unscheduleCampaign() {
      this.$api.changeCampaignStatus(this.data.id, 'draft').then((d) => {
        this.data = d;
      });
    },
  },

  computed: {
    ...mapState(['serverConfig', 'loading', 'lists', 'templates']),

    canManage() {
      return this.$can('campaigns:manage_all', 'campaigns:manage');
    },

    canEdit() {
      return this.isNew
        || this.data.status === 'draft' || this.data.status === 'scheduled' || this.data.status === 'paused';
    },

    canSchedule() {
      return (this.data.status === 'draft' || this.data.status === 'paused') && (this.form.sendLater && this.form.sendAtDate);
    },

    canUnSchedule() {
      return this.data.status === 'scheduled';
    },

    canStart() {
      return (this.data.status === 'draft' || this.data.status === 'paused') && !this.form.sendLater;
    },

    canArchive() {
      return this.data.status !== 'cancelled' && this.data.type !== 'optin';
    },

    selectedLists() {
      if (this.selListIDs.length === 0 || !this.lists.results) {
        return [];
      }

      return this.lists.results.filter((l) => this.selListIDs.indexOf(l.id) > -1);
    },

    emailMessengers() {
      return this.serverConfig.messengers.filter(({ name }) => name === 'email' || name.startsWith('email-'));
    },

    otherMessengers() {
      return this.serverConfig.messengers.filter(({ name }) => name !== 'email' && !name.startsWith('email-'));
    },
  },

  beforeRouteLeave(to, from, next) {
    if (this.isUnsaved()) {
      this.$utils.confirm(this.$t('globals.messages.confirmDiscard'), () => next(true));
      return;
    }
    next(true);
  },

  watch: {
    selectedLists() {
      this.form.lists = this.selectedLists;
    },

    // eslint-disable-next-line func-names
    'data.sendAt': function () {
      if (this.data.sendAt !== null) {
        this.form.sendLater = true;
        this.form.sendAtDate = dayjs(this.data.sendAt).toDate();
      } else {
        this.form.sendLater = false;
        this.form.sendAtDate = null;
      }
    },
  },

  mounted() {
    window.onbeforeunload = () => this.isUnsaved() || null;

    // New campaign.
    const { id } = this.$route.params;
    if (id === 'new') {
      this.isNew = true;

      if (this.$route.query.list_id) {
        // Multiple list_id query params.
        let strIds = [];
        if (typeof this.$route.query.list_id === 'object') {
          strIds = this.$route.query.list_id;
        } else {
          strIds = [this.$route.query.list_id];
        }

        this.selListIDs = strIds.map((v) => parseInt(v, 10));
      }
    } else {
      const intID = parseInt(id, 10);
      if (intID <= 0 || Number.isNaN(intID)) {
        this.$utils.toast(this.$t('campaigns.invalid'));
        return;
      }

      this.isEditing = true;
    }

    // Get templates list.
    this.$api.getTemplates().then((data) => {
      if (data.length > 0) {
        if (!this.form.templateId) {
          const tpl = data.find((i) => i.isDefault === true);
          this.form.templateId = tpl.id;
        }
      }
    });

    // Fetch campaign.
    if (this.isEditing) {
      this.getCampaign(id).then(() => {
        if (this.$route.hash !== '') {
          this.activeTab = this.$route.hash.replace('#', '');
        }
      });
    } else {
      [this.form.messenger] = this.emailMessengers;
      this.form.fromEmail = this.form.messenger.from_email;
    }
    this.prevMessenger = this.form.messenger;

    this.$nextTick(() => {
      this.$refs.focus.focus();
    });

    this.$events.$on('campaign.update', () => {
      this.onSubmit('update');
    });
  },

  beforeDestroy() {
    this.$events.$off('campaign.update');
  },
});
</script>
