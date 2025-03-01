<template>
  <div class="reply-box" :class="replyBoxClass">
    <banner
      v-if="showSelfAssignBanner"
      action-button-variant="clear"
      color-scheme="secondary"
      class="banner--self-assign"
      :banner-message="$t('CONVERSATION.NOT_ASSIGNED_TO_YOU')"
      :has-action-button="true"
      :action-button-label="$t('CONVERSATION.ASSIGN_TO_ME')"
      @click="onClickSelfAssign"
    />
    <reply-top-panel
      :mode="replyType"
      :set-reply-mode="setReplyMode"
      :is-message-length-reaching-threshold="isMessageLengthReachingThreshold"
      :characters-remaining="charactersRemaining"
      :popout-reply-box="popoutReplyBox"
      @click="$emit('click')"
    />
    <div class="reply-box__top">
      <canned-response
        v-if="showMentions && hasSlashCommand"
        v-on-clickaway="hideMentions"
        class="normal-editor__canned-box"
        :search-key="mentionSearchKey"
        @click="replaceText"
      />
      <emoji-input
        v-if="showEmojiPicker"
        v-on-clickaway="hideEmojiPicker"
        :class="emojiDialogClassOnExpandedLayoutAndRTLView"
        :on-click="emojiOnClick"
      />
      <reply-email-head
        v-if="showReplyHead"
        :cc-emails.sync="ccEmails"
        :bcc-emails.sync="bccEmails"
        :to-emails.sync="toEmails"
      />
      <woot-audio-recorder
        v-if="showAudioRecorderEditor"
        ref="audioRecorderInput"
        :audio-record-format="audioRecordFormat"
        @state-recorder-progress-changed="onStateProgressRecorderChanged"
        @state-recorder-changed="onStateRecorderChanged"
        @finish-record="onFinishRecorder"
      />
      <resizable-text-area
        v-else-if="!showRichContentEditor"
        ref="messageInput"
        v-model="message"
        class="input"
        :placeholder="messagePlaceHolder"
        :min-height="4"
        @typing-off="onTypingOff"
        @typing-on="onTypingOn"
        @focus="onFocus"
        @blur="onBlur"
      />
      <woot-message-editor
        v-else
        v-model="message"
        :editor-id="editorStateId"
        class="input"
        :is-private="isOnPrivateNote"
        :placeholder="messagePlaceHolder"
        :update-selection-with="updateEditorSelectionWith"
        :min-height="4"
        :enable-variables="true"
        :variables="messageVariables"
        @typing-off="onTypingOff"
        @typing-on="onTypingOn"
        @focus="onFocus"
        @blur="onBlur"
        @toggle-user-mention="toggleUserMention"
        @toggle-canned-menu="toggleCannedMenu"
        @toggle-variables-menu="toggleVariablesMenu"
        @clear-selection="clearEditorSelection"
      />
    </div>
    <div v-if="hasAttachments" class="attachment-preview-box" @paste="onPaste">
      <attachment-preview
        class="mt-4 flex-col"
        :attachments="attachedFiles"
        :remove-attachment="removeAttachment"
      />
    </div>
    <div
      v-if="isSignatureEnabledForInbox"
      v-tooltip="$t('CONVERSATION.FOOTER.MESSAGE_SIGN_TOOLTIP')"
      class="message-signature-wrap"
    >
      <p
        v-if="isSignatureAvailable"
        v-dompurify-html="formatMessage(messageSignature)"
        class="message-signature"
      />
      <p v-else class="message-signature">
        {{ $t('CONVERSATION.FOOTER.MESSAGE_SIGNATURE_NOT_CONFIGURED') }}
        <router-link :to="profilePath">
          {{ $t('CONVERSATION.FOOTER.CLICK_HERE') }}
        </router-link>
      </p>
    </div>
    <reply-bottom-panel
      :conversation-id="conversationId"
      :enable-multiple-file-upload="enableMultipleFileUpload"
      :has-whatsapp-templates="hasWhatsappTemplates"
      :inbox="inbox"
      :is-on-private-note="isOnPrivateNote"
      :is-recording-audio="isRecordingAudio"
      :is-send-disabled="isReplyButtonDisabled"
      :mode="replyType"
      :on-file-upload="onFileUpload"
      :on-send="onSendReply"
      :recording-audio-duration-text="recordingAudioDurationText"
      :recording-audio-state="recordingAudioState"
      :send-button-text="replyButtonLabel"
      :show-audio-recorder="showAudioRecorder"
      :show-editor-toggle="isAPIInbox && !isOnPrivateNote"
      :show-emoji-picker="showEmojiPicker"
      :show-file-upload="showFileUpload"
      :toggle-audio-recorder-play-pause="toggleAudioRecorderPlayPause"
      :toggle-audio-recorder="toggleAudioRecorder"
      :toggle-emoji-picker="toggleEmojiPicker"
      :message="message"
      :new-conversation-modal-active="newConversationModalActive"
      @selectWhatsappTemplate="openWhatsappTemplateModal"
      @toggle-editor="toggleRichContentEditor"
      @replace-text="replaceText"
    />
    <whatsapp-templates
      :inbox-id="inbox.id"
      :show="showWhatsAppTemplatesModal"
      @close="hideWhatsappTemplatesModal"
      @on-send="onSendWhatsAppReply"
      @cancel="hideWhatsappTemplatesModal"
    />

    <woot-confirm-modal
      ref="confirmDialog"
      :title="$t('CONVERSATION.REPLYBOX.UNDEFINED_VARIABLES.TITLE')"
      :description="undefinedVariableMessage"
    />
  </div>
</template>

<script>
import { mapGetters } from 'vuex';
import { mixin as clickaway } from 'vue-clickaway';
import alertMixin from 'shared/mixins/alertMixin';

import CannedResponse from './CannedResponse';
import ResizableTextArea from 'shared/components/ResizableTextArea';
import AttachmentPreview from 'dashboard/components/widgets/AttachmentsPreview';
import ReplyTopPanel from 'dashboard/components/widgets/WootWriter/ReplyTopPanel';
import ReplyEmailHead from './ReplyEmailHead';
import ReplyBottomPanel from 'dashboard/components/widgets/WootWriter/ReplyBottomPanel';
import Banner from 'dashboard/components/ui/Banner.vue';
import { REPLY_EDITOR_MODES } from 'dashboard/components/widgets/WootWriter/constants';
import WootMessageEditor from 'dashboard/components/widgets/WootWriter/Editor';
import WootAudioRecorder from 'dashboard/components/widgets/WootWriter/AudioRecorder';
import messageFormatterMixin from 'shared/mixins/messageFormatterMixin';
import { AUDIO_FORMATS } from 'shared/constants/messages';
import { BUS_EVENTS } from 'shared/constants/busEvents';
import {
  getMessageVariables,
  getUndefinedVariablesInMessage,
  replaceVariablesInMessage,
} from '@chatwoot/utils';
import WhatsappTemplates from './WhatsappTemplates/Modal.vue';
import { buildHotKeys } from 'shared/helpers/KeyboardHelpers';
import { MESSAGE_MAX_LENGTH } from 'shared/helpers/MessageTypeHelper';
import inboxMixin from 'shared/mixins/inboxMixin';
import uiSettingsMixin from 'dashboard/mixins/uiSettings';
import { frontendURL } from '../../../helper/URLHelper';
import { trimContent, debounce } from '@chatwoot/utils';
import wootConstants from 'dashboard/constants/globals';
import { isEditorHotKeyEnabled } from 'dashboard/mixins/uiSettings';
import { CONVERSATION_EVENTS } from '../../../helper/AnalyticsHelper/events';
import rtlMixin from 'shared/mixins/rtlMixin';
import fileUploadMixin from 'dashboard/mixins/fileUploadMixin';

const EmojiInput = () => import('shared/components/emoji/EmojiInput');

export default {
  components: {
    EmojiInput,
    CannedResponse,
    ResizableTextArea,
    AttachmentPreview,
    ReplyTopPanel,
    ReplyEmailHead,
    ReplyBottomPanel,
    WootMessageEditor,
    WootAudioRecorder,
    Banner,
    WhatsappTemplates,
  },
  mixins: [
    clickaway,
    inboxMixin,
    uiSettingsMixin,
    alertMixin,
    messageFormatterMixin,
    rtlMixin,
    fileUploadMixin,
  ],
  props: {
    selectedTweet: {
      type: [Object, String],
      default: () => ({}),
    },
    isATweet: {
      type: Boolean,
      default: false,
    },
    popoutReplyBox: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      message: '',
      isFocused: false,
      showEmojiPicker: false,
      attachedFiles: [],
      isRecordingAudio: false,
      recordingAudioState: '',
      recordingAudioDurationText: '',
      isUploading: false,
      replyType: REPLY_EDITOR_MODES.REPLY,
      mentionSearchKey: '',
      hasSlashCommand: false,
      bccEmails: '',
      ccEmails: '',
      toEmails: '',
      doAutoSaveDraft: () => {},
      showWhatsAppTemplatesModal: false,
      updateEditorSelectionWith: '',
      undefinedVariableMessage: '',
      showMentions: false,
      showUserMentions: false,
      showCannedMenu: false,
      showVariablesMenu: false,
      newConversationModalActive: false,
    };
  },
  computed: {
    ...mapGetters({
      currentChat: 'getSelectedChat',
      messageSignature: 'getMessageSignature',
      currentUser: 'getCurrentUser',
      lastEmail: 'getLastEmailInSelectedChat',
      globalConfig: 'globalConfig/get',
      accountId: 'getCurrentAccountId',
    }),
    showRichContentEditor() {
      if (this.isOnPrivateNote || this.isRichEditorEnabled) {
        return true;
      }

      if (this.isAPIInbox) {
        const {
          display_rich_content_editor: displayRichContentEditor = false,
        } = this.uiSettings;
        return displayRichContentEditor;
      }

      return false;
    },
    assignedAgent: {
      get() {
        return this.currentChat.meta.assignee;
      },
      set(agent) {
        const agentId = agent ? agent.id : 0;
        this.$store.dispatch('setCurrentChatAssignee', agent);
        this.$store
          .dispatch('assignAgent', {
            conversationId: this.currentChat.id,
            agentId,
          })
          .then(() => {
            this.showAlert(this.$t('CONVERSATION.CHANGE_AGENT'));
          });
      },
    },
    showSelfAssignBanner() {
      if (this.message !== '' && !this.isOnPrivateNote) {
        if (!this.assignedAgent) {
          return true;
        }
        if (this.assignedAgent.id !== this.currentUser.id) {
          return true;
        }
      }

      return false;
    },
    hasWhatsappTemplates() {
      return !!this.$store.getters['inboxes/getWhatsAppTemplates'](this.inboxId)
        .length;
    },
    isPrivate() {
      if (this.currentChat.can_reply || this.isAWhatsAppChannel) {
        return this.isOnPrivateNote;
      }
      return true;
    },
    inboxId() {
      return this.currentChat.inbox_id;
    },
    inbox() {
      return this.$store.getters['inboxes/getInbox'](this.inboxId);
    },
    messagePlaceHolder() {
      return this.isPrivate
        ? this.$t('CONVERSATION.FOOTER.PRIVATE_MSG_INPUT')
        : this.$t('CONVERSATION.FOOTER.MSG_INPUT');
    },
    isMessageLengthReachingThreshold() {
      return this.message.length > this.maxLength - 50;
    },
    charactersRemaining() {
      return this.maxLength - this.message.length;
    },
    isReplyButtonDisabled() {
      if (this.isATweet && !this.inReplyTo && !this.isOnPrivateNote) {
        return true;
      }

      if (this.hasAttachments || this.hasRecordedAudio) return false;

      return (
        this.isMessageEmpty ||
        this.message.length === 0 ||
        this.message.length > this.maxLength
      );
    },
    sender() {
      return {
        name: this.currentUser.name,
        thumbnail: this.currentUser.avatar_url,
      };
    },
    conversationType() {
      const { additional_attributes: additionalAttributes } = this.currentChat;
      const type = additionalAttributes ? additionalAttributes.type : '';
      return type || '';
    },
    maxLength() {
      if (this.isPrivate) {
        return MESSAGE_MAX_LENGTH.GENERAL;
      }
      if (this.isAFacebookInbox) {
        return MESSAGE_MAX_LENGTH.FACEBOOK;
      }
      if (this.isAWhatsAppChannel) {
        return MESSAGE_MAX_LENGTH.TWILIO_WHATSAPP;
      }
      if (this.isASmsInbox) {
        return MESSAGE_MAX_LENGTH.TWILIO_SMS;
      }
      if (this.isATwitterInbox) {
        if (this.conversationType === 'tweet') {
          return MESSAGE_MAX_LENGTH.TWEET - this.replyToUserLength - 2;
        }
      }
      return MESSAGE_MAX_LENGTH.GENERAL;
    },
    showFileUpload() {
      return (
        this.isAWebWidgetInbox ||
        this.isAFacebookInbox ||
        this.isAWhatsAppChannel ||
        this.isAPIInbox ||
        this.isAnEmailChannel ||
        this.isASmsInbox ||
        this.isATelegramChannel
      );
    },
    replyButtonLabel() {
      let sendMessageText = this.$t('CONVERSATION.REPLYBOX.SEND');
      if (this.isPrivate) {
        sendMessageText = this.$t('CONVERSATION.REPLYBOX.CREATE');
      } else if (this.conversationType === 'tweet') {
        sendMessageText = this.$t('CONVERSATION.REPLYBOX.TWEET');
      }
      const keyLabel = isEditorHotKeyEnabled(this.uiSettings, 'cmd_enter')
        ? '(⌘ + ↵)'
        : '(↵)';
      return `${sendMessageText} ${keyLabel}`;
    },
    replyBoxClass() {
      return {
        'is-private': this.isPrivate,
        'is-focused': this.isFocused || this.hasAttachments,
      };
    },
    hasAttachments() {
      return this.attachedFiles.length;
    },
    hasRecordedAudio() {
      return (
        this.$refs.audioRecorderInput &&
        this.$refs.audioRecorderInput.hasAudio()
      );
    },
    isRichEditorEnabled() {
      return this.isAWebWidgetInbox || this.isAnEmailChannel;
    },
    showAudioRecorder() {
      return !this.isOnPrivateNote && this.showFileUpload;
    },
    showAudioRecorderEditor() {
      return this.showAudioRecorder && this.isRecordingAudio;
    },
    isOnPrivateNote() {
      return this.replyType === REPLY_EDITOR_MODES.NOTE;
    },
    inReplyTo() {
      const selectedTweet = this.selectedTweet || {};
      return selectedTweet.id;
    },
    isOnExpandedLayout() {
      const {
        LAYOUT_TYPES: { CONDENSED },
      } = wootConstants;
      const {
        conversation_display_type: conversationDisplayType = CONDENSED,
      } = this.uiSettings;
      return conversationDisplayType !== CONDENSED;
    },
    emojiDialogClassOnExpandedLayoutAndRTLView() {
      if (this.isOnExpandedLayout || this.popoutReplyBox) {
        return 'emoji-dialog--expanded';
      }
      if (this.isRTLView) {
        return 'emoji-dialog--rtl';
      }
      return '';
    },
    replyToUserLength() {
      const selectedTweet = this.selectedTweet || {};
      const {
        sender: {
          additional_attributes: { screen_name: screenName = '' } = {},
        } = {},
      } = selectedTweet;
      return screenName ? screenName.length : 0;
    },
    isMessageEmpty() {
      if (!this.message) {
        return true;
      }
      return !this.message.trim().replace(/\n/g, '').length;
    },
    showReplyHead() {
      return !this.isOnPrivateNote && this.isAnEmailChannel;
    },
    enableMultipleFileUpload() {
      return (
        this.isAnEmailChannel ||
        this.isAWebWidgetInbox ||
        this.isAPIInbox ||
        this.isAWhatsAppChannel
      );
    },
    isSignatureEnabledForInbox() {
      return !this.isPrivate && this.isAnEmailChannel && this.sendWithSignature;
    },
    isSignatureAvailable() {
      return !!this.messageSignature;
    },
    sendWithSignature() {
      const { send_with_signature: isEnabled } = this.uiSettings;
      return isEnabled;
    },
    profilePath() {
      return frontendURL(`accounts/${this.accountId}/profile/settings`);
    },
    editorMessageKey() {
      const { editor_message_key: isEnabled } = this.uiSettings;
      return isEnabled;
    },
    commandPlusEnterToSendEnabled() {
      return this.editorMessageKey === 'cmd_enter';
    },
    enterToSendEnabled() {
      return this.editorMessageKey === 'enter';
    },
    conversationId() {
      return this.currentChat.id;
    },
    conversationIdByRoute() {
      return this.conversationId;
    },
    editorStateId() {
      return `draft-${this.conversationIdByRoute}-${this.replyType}`;
    },
    audioRecordFormat() {
      if (this.isAWhatsAppChannel || this.isAPIInbox) {
        return AUDIO_FORMATS.OGG;
      }
      return AUDIO_FORMATS.WAV;
    },
    messageVariables() {
      const variables = getMessageVariables({
        conversation: this.currentChat,
      });
      return variables;
    },
  },
  watch: {
    currentChat(conversation) {
      const { can_reply: canReply } = conversation;

      if (this.isOnPrivateNote) {
        return;
      }

      if (canReply || this.isAWhatsAppChannel) {
        this.replyType = REPLY_EDITOR_MODES.REPLY;
      } else {
        this.replyType = REPLY_EDITOR_MODES.NOTE;
      }

      this.setCCAndToEmailsFromLastChat();
    },
    conversationIdByRoute(conversationId, oldConversationId) {
      if (conversationId !== oldConversationId) {
        this.setToDraft(oldConversationId, this.replyType);
        this.getFromDraft();
      }
    },
    message(updatedMessage) {
      this.hasSlashCommand =
        updatedMessage[0] === '/' && !this.showRichContentEditor;
      const hasNextWord = updatedMessage.includes(' ');
      const isShortCodeActive = this.hasSlashCommand && !hasNextWord;
      if (isShortCodeActive) {
        this.mentionSearchKey = updatedMessage.substring(1);
        this.showMentions = true;
      } else {
        this.mentionSearchKey = '';
        this.showMentions = false;
      }
      this.doAutoSaveDraft();
    },
    replyType(updatedReplyType, oldReplyType) {
      this.setToDraft(this.conversationIdByRoute, oldReplyType);
      this.getFromDraft();
    },
  },

  mounted() {
    this.getFromDraft();
    // Donot use the keyboard listener mixin here as the events here are supposed to be
    // working even if input/textarea is focussed.
    document.addEventListener('paste', this.onPaste);
    document.addEventListener('keydown', this.handleKeyEvents);
    this.setCCAndToEmailsFromLastChat();
    this.doAutoSaveDraft = debounce(
      () => {
        this.saveDraft(this.conversationIdByRoute, this.replyType);
      },
      500,
      true
    );

    // A hacky fix to solve the drag and drop
    // Is showing on top of new conversation modal drag and drop
    // TODO need to find a better solution
    bus.$on(
      BUS_EVENTS.NEW_CONVERSATION_MODAL,
      this.onNewConversationModalActive
    );
  },
  destroyed() {
    document.removeEventListener('paste', this.onPaste);
    document.removeEventListener('keydown', this.handleKeyEvents);
  },
  beforeDestroy() {
    bus.$off(
      BUS_EVENTS.NEW_CONVERSATION_MODAL,
      this.onNewConversationModalActive
    );
  },
  methods: {
    toggleRichContentEditor() {
      this.updateUISettings({
        display_rich_content_editor: !this.showRichContentEditor,
      });
    },
    saveDraft(conversationId, replyType) {
      if (this.message || this.message === '') {
        const key = `draft-${conversationId}-${replyType}`;
        const draftToSave = trimContent(this.message || '');

        this.$store.dispatch('draftMessages/set', {
          key,
          message: draftToSave,
        });
      }
    },
    setToDraft(conversationId, replyType) {
      this.saveDraft(conversationId, replyType);
      this.message = '';
    },
    getFromDraft() {
      if (this.conversationIdByRoute) {
        const key = `draft-${this.conversationIdByRoute}-${this.replyType}`;
        this.message = this.$store.getters['draftMessages/get'](key) || '';
      }
    },
    removeFromDraft() {
      if (this.conversationIdByRoute) {
        const key = `draft-${this.conversationIdByRoute}-${this.replyType}`;
        this.$store.dispatch('draftMessages/delete', { key });
      }
    },
    handleKeyEvents(e) {
      const keyCode = buildHotKeys(e);
      if (keyCode === 'escape') {
        this.hideEmojiPicker();
        this.hideMentions();
      } else if (keyCode === 'meta+k') {
        const ninja = document.querySelector('ninja-keys');
        ninja.open();
        e.preventDefault();
      } else if (keyCode === 'enter' && this.isAValidEvent('enter')) {
        this.onSendReply();
        e.preventDefault();
      } else if (
        ['meta+enter', 'ctrl+enter'].includes(keyCode) &&
        this.isAValidEvent('cmd_enter')
      ) {
        this.onSendReply();
      }
    },
    isAValidEvent(selectedKey) {
      return (
        !this.showUserMentions &&
        !this.showMentions &&
        !this.showCannedMenu &&
        !this.showVariablesMenu &&
        this.isFocused &&
        isEditorHotKeyEnabled(this.uiSettings, selectedKey)
      );
    },
    onPaste(e) {
      const data = e.clipboardData.files;
      if (!this.showRichContentEditor && data.length !== 0) {
        this.$refs.messageInput.$el.blur();
      }
      if (!data.length || !data[0]) {
        return;
      }
      data.forEach(file => {
        const { name, type, size } = file;
        this.onFileUpload({ name, type, size, file: file });
      });
    },
    toggleUserMention(currentMentionState) {
      this.showUserMentions = currentMentionState;
    },
    toggleCannedMenu(value) {
      this.showCannedMenu = value;
    },
    toggleVariablesMenu(value) {
      this.showVariablesMenu = value;
    },
    openWhatsappTemplateModal() {
      this.showWhatsAppTemplatesModal = true;
    },
    hideWhatsappTemplatesModal() {
      this.showWhatsAppTemplatesModal = false;
    },
    onClickSelfAssign() {
      const {
        account_id,
        availability_status,
        available_name,
        email,
        id,
        name,
        role,
        avatar_url,
      } = this.currentUser;
      const selfAssign = {
        account_id,
        availability_status,
        available_name,
        email,
        id,
        name,
        role,
        thumbnail: avatar_url,
      };
      this.assignedAgent = selfAssign;
    },
    confirmOnSendReply() {
      if (this.isReplyButtonDisabled) {
        return;
      }
      if (!this.showMentions) {
        let newMessage = this.message;
        if (this.isSignatureEnabledForInbox && this.messageSignature) {
          newMessage += '\n\n' + this.messageSignature;
        }

        const isOnWhatsApp =
          this.isATwilioWhatsAppChannel ||
          this.isAWhatsAppCloudChannel ||
          this.is360DialogWhatsAppChannel;
        if (isOnWhatsApp && !this.isPrivate) {
          this.sendMessageAsMultipleMessages(newMessage);
        } else {
          const messagePayload = this.getMessagePayload(newMessage);
          this.sendMessage(messagePayload);
        }

        if (!this.isPrivate) {
          this.clearEmailField();
        }

        this.clearMessage();
        this.hideEmojiPicker();
        this.$emit('update:popoutReplyBox', false);
      }
    },
    sendMessageAsMultipleMessages(message) {
      const messages = this.getMessagePayloadForWhatsapp(message);
      messages.forEach(messagePayload => {
        this.sendMessage(messagePayload);
      });
    },
    sendMessageAnalyticsData(isPrivate) {
      // Analytics data for message signature is enabled or not in channels
      return isPrivate
        ? this.$track(CONVERSATION_EVENTS.SENT_PRIVATE_NOTE)
        : this.$track(CONVERSATION_EVENTS.SENT_MESSAGE, {
            channelType: this.channelType,
            signatureEnabled: this.sendWithSignature,
          });
    },
    async onSendReply() {
      const undefinedVariables = getUndefinedVariablesInMessage({
        message: this.message,
        variables: this.messageVariables,
      });
      if (undefinedVariables.length > 0) {
        const undefinedVariablesCount =
          undefinedVariables.length > 1 ? undefinedVariables.length : 1;
        this.undefinedVariableMessage = this.$t(
          'CONVERSATION.REPLYBOX.UNDEFINED_VARIABLES.MESSAGE',
          {
            undefinedVariablesCount,
            undefinedVariables: undefinedVariables.join(', '),
          }
        );

        const ok = await this.$refs.confirmDialog.showConfirmation();
        if (ok) {
          this.confirmOnSendReply();
        }
      } else {
        this.confirmOnSendReply();
      }
    },
    async sendMessage(messagePayload) {
      try {
        await this.$store.dispatch(
          'createPendingMessageAndSend',
          messagePayload
        );
        bus.$emit(BUS_EVENTS.SCROLL_TO_MESSAGE);
        bus.$emit(BUS_EVENTS.MESSAGE_SENT);
        this.removeFromDraft();
        this.sendMessageAnalyticsData(messagePayload.private);
      } catch (error) {
        const errorMessage =
          error?.response?.data?.error || this.$t('CONVERSATION.MESSAGE_ERROR');
        this.showAlert(errorMessage);
      }
    },
    async onSendWhatsAppReply(messagePayload) {
      this.sendMessage({
        conversationId: this.currentChat.id,
        ...messagePayload,
      });
      this.hideWhatsappTemplatesModal();
    },
    replaceText(message) {
      const updatedMessage = replaceVariablesInMessage({
        message,
        variables: this.messageVariables,
      });
      setTimeout(() => {
        this.$track(CONVERSATION_EVENTS.INSERTED_A_CANNED_RESPONSE);
        this.message = updatedMessage;
      }, 100);
    },
    setReplyMode(mode = REPLY_EDITOR_MODES.REPLY) {
      const { can_reply: canReply } = this.currentChat;
      this.$store.dispatch('draftMessages/setReplyEditorMode', {
        mode,
      });
      if (canReply || this.isAWhatsAppChannel) this.replyType = mode;
      if (this.showRichContentEditor) {
        if (this.isRecordingAudio) {
          this.toggleAudioRecorder();
        }
        return;
      }
      this.$nextTick(() => this.$refs.messageInput.focus());
    },
    clearEditorSelection() {
      this.updateEditorSelectionWith = '';
    },
    insertEmoji(emoji, selectionStart, selectionEnd) {
      const { message } = this;
      const newMessage =
        message.slice(0, selectionStart) +
        emoji +
        message.slice(selectionEnd, message.length);
      this.message = newMessage;
    },
    emojiOnClick(emoji) {
      if (this.showRichContentEditor) {
        this.updateEditorSelectionWith = emoji;
        this.onFocus();
      }
      if (!this.showRichContentEditor) {
        const { selectionStart, selectionEnd } = this.$refs.messageInput.$el;
        this.insertEmoji(emoji, selectionStart, selectionEnd);
      }
    },
    clearMessage() {
      this.message = '';
      this.attachedFiles = [];
      this.isRecordingAudio = false;
    },
    clearEmailField() {
      this.ccEmails = '';
      this.bccEmails = '';
      this.toEmails = '';
    },
    toggleEmojiPicker() {
      this.showEmojiPicker = !this.showEmojiPicker;
    },
    toggleAudioRecorder() {
      this.isRecordingAudio = !this.isRecordingAudio;
      this.isRecorderAudioStopped = !this.isRecordingAudio;
      if (!this.isRecordingAudio) {
        this.clearMessage();
        this.clearEmailField();
      }
    },
    toggleAudioRecorderPlayPause() {
      if (this.isRecordingAudio) {
        if (!this.isRecorderAudioStopped) {
          this.isRecorderAudioStopped = true;
          this.$refs.audioRecorderInput.stopAudioRecording();
        } else if (this.isRecorderAudioStopped) {
          this.$refs.audioRecorderInput.playPause();
        }
      }
    },
    hideEmojiPicker() {
      if (this.showEmojiPicker) {
        this.toggleEmojiPicker();
      }
    },
    hideMentions() {
      this.showMentions = false;
    },
    onTypingOn() {
      this.toggleTyping('on');
    },
    onTypingOff() {
      this.toggleTyping('off');
    },
    onBlur() {
      this.isFocused = false;
      this.saveDraft(this.conversationIdByRoute, this.replyType);
    },
    onFocus() {
      this.isFocused = true;
    },
    onStateProgressRecorderChanged(duration) {
      this.recordingAudioDurationText = duration;
    },
    onStateRecorderChanged(state) {
      this.recordingAudioState = state;
      if (state && 'notallowederror'.includes(state)) {
        this.toggleAudioRecorder();
      }
    },
    onFinishRecorder(file) {
      return file && this.onFileUpload(file);
    },
    toggleTyping(status) {
      const conversationId = this.currentChat.id;
      const isPrivate = this.isPrivate;

      if (!conversationId) {
        return;
      }

      this.$store.dispatch('conversationTypingStatus/toggleTyping', {
        status,
        conversationId,
        isPrivate,
      });
    },
    attachFile({ blob, file }) {
      const reader = new FileReader();
      reader.readAsDataURL(file.file);
      reader.onloadend = () => {
        this.attachedFiles.push({
          currentChatId: this.currentChat.id,
          resource: blob || file,
          isPrivate: this.isPrivate,
          thumb: reader.result,
          blobSignedId: blob ? blob.signed_id : undefined,
        });
      };
    },
    removeAttachment(itemIndex) {
      this.attachedFiles = this.attachedFiles.filter(
        (item, index) => itemIndex !== index
      );
    },
    getMessagePayloadForWhatsapp(message) {
      const multipleMessagePayload = [];

      if (this.attachedFiles && this.attachedFiles.length) {
        let caption = message;
        this.attachedFiles.forEach(attachment => {
          const attachedFile = this.globalConfig.directUploadsEnabled
            ? attachment.blobSignedId
            : attachment.resource.file;
          const attachmentPayload = {
            conversationId: this.currentChat.id,
            files: [attachedFile],
            private: false,
            message: caption,
            sender: this.sender,
          };
          multipleMessagePayload.push(attachmentPayload);
          caption = '';
        });
      } else {
        const messagePayload = {
          conversationId: this.currentChat.id,
          message,
          private: false,
          sender: this.sender,
        };
        multipleMessagePayload.push(messagePayload);
      }

      return multipleMessagePayload;
    },
    getMessagePayload(message) {
      const messagePayload = {
        conversationId: this.currentChat.id,
        message,
        private: this.isPrivate,
        sender: this.sender,
      };

      if (this.inReplyTo) {
        messagePayload.contentAttributes = { in_reply_to: this.inReplyTo };
      }

      if (this.attachedFiles && this.attachedFiles.length) {
        messagePayload.files = [];
        this.attachedFiles.forEach(attachment => {
          if (this.globalConfig.directUploadsEnabled) {
            messagePayload.files.push(attachment.blobSignedId);
          } else {
            messagePayload.files.push(attachment.resource.file);
          }
        });
      }

      if (this.ccEmails && !this.isOnPrivateNote) {
        messagePayload.ccEmails = this.ccEmails;
      }

      if (this.bccEmails && !this.isOnPrivateNote) {
        messagePayload.bccEmails = this.bccEmails;
      }

      if (this.toEmails && !this.isOnPrivateNote) {
        messagePayload.toEmails = this.toEmails;
      }

      return messagePayload;
    },
    setCcEmails(value) {
      this.bccEmails = value.bccEmails;
      this.ccEmails = value.ccEmails;
    },
    setCCAndToEmailsFromLastChat() {
      if (!this.lastEmail) return;

      const {
        content_attributes: { email: emailAttributes = {} },
      } = this.lastEmail;

      // Retrieve the email of the current conversation's sender
      const conversationContact = this.currentChat?.meta?.sender?.email || '';
      let cc = emailAttributes.cc ? [...emailAttributes.cc] : [];
      let to = [];

      // there might be a situation where the current conversation will include a message from a third person,
      // and the current conversation contact is in CC.
      // This is an edge-case, reported here: CW-1511 [ONLY FOR INTERNAL REFERENCE]
      // So we remove the current conversation contact's email from the CC list if present
      if (cc.includes(conversationContact)) {
        cc = cc.filter(email => email !== conversationContact);
      }

      // If the last incoming message sender is different from the conversation contact, add them to the "to"
      // and add the conversation contact to the CC
      if (!emailAttributes.from.includes(conversationContact)) {
        to.push(...emailAttributes.from);
        cc.push(conversationContact);
      }

      // Remove the conversation contact's email from the BCC list if present
      let bcc = (emailAttributes.bcc || []).filter(
        email => email !== conversationContact
      );

      // Ensure only unique email addresses are in the CC list
      bcc = [...new Set(bcc)];
      cc = [...new Set(cc)];
      to = [...new Set(to)];

      this.ccEmails = cc.join(', ');
      this.bccEmails = bcc.join(', ');
      this.toEmails = to.join(', ');
    },
    onNewConversationModalActive(isActive) {
      // Issue is if the new conversation modal is open and we drag and drop the file
      // then the file is not getting attached to the new conversation modal
      // and it is getting attached to the current conversation reply box
      // so to fix this we are removing the drag and drop event listener from the current conversation reply box
      // When new conversation modal is open
      this.newConversationModalActive = isActive;
    },
  },
};
</script>

<style lang="scss" scoped>
.send-button {
  @apply mb-0;
}

.banner--self-assign {
  @apply py-2;
}

.message-signature-wrap {
  @apply my-0 mx-4 px-1 flex max-h-[8vh] items-baseline justify-between hover:bg-slate-25 dark:hover:bg-slate-800 border border-dashed border-slate-100 dark:border-slate-700 rounded-sm overflow-auto;
}

.message-signature {
  @apply w-fit m-0;
}

.attachment-preview-box {
  @apply bg-transparent py-0 px-4;
}

.reply-box {
  @apply border-t border-slate-50 dark:border-slate-700 bg-white dark:bg-slate-900;

  &.is-private {
    @apply bg-yellow-50 dark:bg-yellow-200;
  }
}
.send-button {
  @apply mb-0;
}

.reply-box__top {
  @apply relative py-0 px-4 -mt-px border-t border-solid border-slate-50 dark:border-slate-700;
}

.emoji-dialog {
  @apply top-[unset] -bottom-10 -left-80 right-[unset];

  &::before {
    transform: rotate(270deg);
    filter: drop-shadow(0px 4px 4px rgba(0, 0, 0, 0.08));
    @apply -right-4 bottom-2 rtl:right-0 rtl:-left-4;
  }
}

.emoji-dialog--rtl {
  @apply left-[unset] -right-80;
  &::before {
    transform: rotate(90deg);
    filter: drop-shadow(0px 4px 4px rgba(0, 0, 0, 0.08));
  }
}

.emoji-dialog--expanded {
  @apply left-[unset] bottom-0 absolute z-[100];

  &::before {
    transform: rotate(0deg);
    @apply left-1 -bottom-2;
  }
}
.message-signature {
  @apply mb-0;

  ::v-deep p:last-child {
    @apply mb-0;
  }
}

.normal-editor__canned-box {
  width: calc(100% - 2 * var(--space-normal));
  left: var(--space-normal);
}
</style>
