<template>
  <li v-if="hasAttachments || data.content" :class="alignBubble">
    <div :class="wrapClass">
      <div v-tooltip.top-start="messageToolTip" :class="bubbleClass">
        <bubble-mail-head
          :email-attributes="contentAttributes.email"
          :cc="emailHeadAttributes.cc"
          :bcc="emailHeadAttributes.bcc"
          :is-incoming="isIncoming"
        />
        <bubble-text
          v-if="data.content"
          :message="message"
          :is-email="isEmailContentType"
          :readable-time="readableTime"
          :display-quoted-button="displayQuotedButton"
        />
        <span
          v-if="isPending && hasAttachments"
          class="chat-bubble has-attachment agent"
        >
          {{ $t('CONVERSATION.UPLOADING_ATTACHMENTS') }}
        </span>
        <div v-if="!isPending && hasAttachments">
          <div v-for="attachment in data.attachments" :key="attachment.id">
            <bubble-image
              v-if="attachment.file_type === 'image' && !hasImageError"
              :url="attachment.data_url"
              :readable-time="readableTime"
              @error="onImageLoadError"
            />
            <audio v-else-if="attachment.file_type === 'audio'" controls>
              <source :src="attachment.data_url" />
            </audio>
            <bubble-video
              v-else-if="attachment.file_type === 'video'"
              :url="attachment.data_url"
              :readable-time="readableTime"
            />
            <bubble-file
              v-else
              :url="attachment.data_url"
              :readable-time="readableTime"
            />
          </div>
        </div>
        <bubble-actions
          :id="data.id"
          :sender="data.sender"
          :is-a-tweet="isATweet"
          :is-email="isEmailContentType"
          :is-private="data.private"
          :message-type="data.message_type"
          :readable-time="readableTime"
          :source-id="data.source_id"
          :inbox-id="data.inbox_id"
        />
      </div>
      <spinner v-if="isPending" size="tiny" />
      <a
        v-if="isATweet && isIncoming && sender"
        class="sender--info"
        :href="twitterProfileLink"
        target="_blank"
        rel="noopener noreferrer nofollow"
      >
        <woot-thumbnail
          :src="sender.thumbnail"
          :username="sender.name"
          size="16px"
        />
        <div class="sender--available-name">
          {{ sender.name }}
        </div>
      </a>
      <div v-if="isFailed" class="message-failed--alert">
        <woot-button
          v-tooltip.top-end="$t('CONVERSATION.TRY_AGAIN')"
          size="tiny"
          color-scheme="alert"
          variant="clear"
          icon="arrow-clockwise"
          @click="retrySendMessage"
        />
      </div>
    </div>
    <div v-if="shouldShowContextMenu" class="context-menu-wrap">
      <context-menu
        v-if="isBubble && !isMessageDeleted"
        :is-open="showContextMenu"
        :show-copy="hasText"
        :menu-position="contextMenuPosition"
        @toggle="handleContextMenuClick"
        @delete="handleDelete"
        @copy="handleCopy"
      />
    </div>
  </li>
</template>
<script>
import copy from 'copy-text-to-clipboard';

import messageFormatterMixin from 'shared/mixins/messageFormatterMixin';
import timeMixin from '../../../mixins/time';

import BubbleMailHead from './bubble/MailHead';
import BubbleText from './bubble/Text';
import BubbleImage from './bubble/Image';
import BubbleFile from './bubble/File';
import BubbleVideo from './bubble/Video.vue';
import BubbleActions from './bubble/Actions';

import Spinner from 'shared/components/Spinner';
import ContextMenu from 'dashboard/modules/conversations/components/MessageContextMenu';

import { isEmptyObject } from 'dashboard/helper/commons';
import alertMixin from 'shared/mixins/alertMixin';
import contentTypeMixin from 'shared/mixins/contentTypeMixin';
import { MESSAGE_TYPE, MESSAGE_STATUS } from 'shared/constants/messages';
import { generateBotMessageContent } from './helpers/botMessageContentHelper';

export default {
  components: {
    BubbleActions,
    BubbleText,
    BubbleImage,
    BubbleFile,
    BubbleVideo,
    BubbleMailHead,
    ContextMenu,
    Spinner,
  },
  mixins: [alertMixin, timeMixin, messageFormatterMixin, contentTypeMixin],
  props: {
    data: {
      type: Object,
      required: true,
    },
    isATweet: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      showContextMenu: false,
      hasImageError: false,
    };
  },
  computed: {
    contentToBeParsed() {
      const {
        html_content: { full: fullHTMLContent } = {},
        text_content: { full: fullTextContent } = {},
      } = this.contentAttributes.email || {};
      return fullHTMLContent || fullTextContent || '';
    },
    displayQuotedButton() {
      if (!this.isIncoming) {
        return false;
      }

      if (this.contentToBeParsed.includes('<blockquote')) {
        return true;
      }

      return false;
    },
    message() {
      const botMessageContent = generateBotMessageContent(
        this.contentType,
        this.contentAttributes,
        {
          noResponseText: this.$t('CONVERSATION.NO_RESPONSE'),
          csat: {
            ratingTitle: this.$t('CONVERSATION.RATING_TITLE'),
            feedbackTitle: this.$t('CONVERSATION.FEEDBACK_TITLE'),
          },
        }
      );

      const {
        email: { content_type: contentType = '' } = {},
      } = this.contentAttributes;
      if (this.contentToBeParsed && this.isIncoming) {
        const parsedContent = this.stripStyleCharacters(this.contentToBeParsed);
        if (parsedContent) {
          // This is a temporary fix for line-breaks in text/plain emails
          // Now, It is not rendered properly in the email preview.
          // FIXME: Remove this once we have a better solution for rendering text/plain emails
          return contentType.includes('text/plain')
            ? parsedContent.replace(/\n/g, '<br />')
            : parsedContent;
        }
      }
      return (
        this.formatMessage(this.data.content, this.isATweet) + botMessageContent
      );
    },
    contentAttributes() {
      return this.data.content_attributes || {};
    },
    sender() {
      return this.data.sender || {};
    },
    contentType() {
      const {
        data: { content_type: contentType },
      } = this;
      return contentType;
    },
    twitterProfileLink() {
      const additionalAttributes = this.sender.additional_attributes || {};
      const { screen_name: screenName } = additionalAttributes;
      return `https://twitter.com/${screenName}`;
    },
    alignBubble() {
      const { message_type: messageType } = this.data;
      const isCentered = messageType === MESSAGE_TYPE.ACTIVITY;
      const isLeftAligned = messageType === MESSAGE_TYPE.INCOMING;
      const isRightAligned =
        messageType === MESSAGE_TYPE.OUTGOING ||
        messageType === MESSAGE_TYPE.TEMPLATE;

      return {
        center: isCentered,
        left: isLeftAligned,
        right: isRightAligned,
        'has-context-menu': this.showContextMenu,
        'has-tweet-menu': this.isATweet,
      };
    },
    readableTime() {
      return this.messageStamp(
        this.contentAttributes.external_created_at || this.data.created_at,
        'LLL d, h:mm a'
      );
    },
    isBubble() {
      return [0, 1, 3].includes(this.data.message_type);
    },
    isIncoming() {
      return this.data.message_type === MESSAGE_TYPE.INCOMING;
    },
    emailHeadAttributes() {
      return {
        email: this.contentAttributes.email,
        cc: this.contentAttributes.cc_emails,
        bcc: this.contentAttributes.bcc_emails,
      };
    },
    hasAttachments() {
      return !!(this.data.attachments && this.data.attachments.length > 0);
    },
    isMessageDeleted() {
      return this.contentAttributes.deleted;
    },
    hasText() {
      return !!this.data.content;
    },
    messageToolTip() {
      if (this.isMessageDeleted) {
        return false;
      }
      if (this.isFailed) {
        return this.$t(`CONVERSATION.SEND_FAILED`);
      }
      const { sender } = this;
      return this.data.message_type === 1 && !isEmptyObject(sender)
        ? {
            content: `${this.$t('CONVERSATION.SENT_BY')} ${sender.name}`,
            classes: 'top',
          }
        : false;
    },
    wrapClass() {
      return {
        wrap: this.isBubble,
        'activity-wrap': !this.isBubble,
        'is-pending': this.isPending,
        'is-failed': this.isFailed,
      };
    },
    bubbleClass() {
      return {
        bubble: this.isBubble,
        'is-private': this.data.private,
        'is-image': this.hasMediaAttachment('image'),
        'is-video': this.hasMediaAttachment('video'),
        'is-text': this.hasText,
        'is-from-bot': this.isSentByBot,
        'is-failed': this.isFailed,
      };
    },
    isPending() {
      return this.data.status === MESSAGE_STATUS.PROGRESS;
    },
    isFailed() {
      return this.data.status === MESSAGE_STATUS.FAILED;
    },
    isSentByBot() {
      if (this.isPending || this.isFailed) return false;
      return !this.sender.type || this.sender.type === 'agent_bot';
    },
    contextMenuPosition() {
      const { message_type: messageType } = this.data;
      return messageType ? 'right' : 'left';
    },
    shouldShowContextMenu() {
      return !(this.isFailed || this.isPending);
    },
    errorMessage() {
      const { meta } = this.data;
      return meta ? meta.error : '';
    },
  },
  watch: {
    data() {
      this.hasImageError = false;
    },
  },
  mounted() {
    this.hasImageError = false;
  },
  methods: {
    hasMediaAttachment(type) {
      if (this.hasAttachments && this.data.attachments.length > 0) {
        const { attachments = [{}] } = this.data;
        const { file_type: fileType } = attachments[0];
        return fileType === type && !this.hasImageError;
      }
      return false;
    },
    handleContextMenuClick() {
      this.showContextMenu = !this.showContextMenu;
    },
    async handleDelete() {
      const { conversation_id: conversationId, id: messageId } = this.data;
      try {
        await this.$store.dispatch('deleteMessage', {
          conversationId,
          messageId,
        });
        this.showAlert(this.$t('CONVERSATION.SUCCESS_DELETE_MESSAGE'));
        this.showContextMenu = false;
      } catch (error) {
        this.showAlert(this.$t('CONVERSATION.FAIL_DELETE_MESSSAGE'));
      }
    },
    handleCopy() {
      copy(this.data.content);
      this.showAlert(this.$t('CONTACT_PANEL.COPY_SUCCESSFUL'));
      this.showContextMenu = false;
    },
    async retrySendMessage() {
      await this.$store.dispatch('sendMessageWithData', this.data);
    },
    onImageLoadError() {
      this.hasImageError = true;
    },
  },
};
</script>
<style lang="scss">
.wrap {
  > .bubble {
    &.is-image,
    &.is-video {
      padding: 0;
      overflow: hidden;

      .image,
      .video {
        max-width: 32rem;
        padding: var(--space-micro);

        > img,
        > video {
          border-radius: var(--border-radius-medium);
        }
        > video {
          height: 100%;
          object-fit: cover;
          width: 100%;
        }
      }
      .video {
        height: 18rem;
      }
    }

    &.is-image.is-text > .message-text__wrap {
      max-width: 32rem;
      padding: var(--space-small) var(--space-normal);
    }

    &.is-private .file.message-text__wrap {
      .file--icon {
        color: var(--w-400);
      }
      .text-block-title {
        color: #3c4858;
      }
      .download.button {
        color: var(--w-400);
      }
    }

    &.is-private.is-text > .message-text__wrap .link {
      color: var(--w-700);
    }
    &.is-private.is-text > .message-text__wrap .prosemirror-mention-node {
      font-weight: var(--font-weight-black);
      background: none;
      border-radius: var(--border-radius-small);
      padding: 0;
      color: var(--color-body);
      text-decoration: underline;
    }

    &.is-from-bot {
      background: var(--v-400);
      .message-text--metadata .time {
        color: var(--v-50);
      }
      &.is-private .message-text--metadata .time {
        color: var(--s-400);
      }
    }

    &.is-failed {
      background: var(--r-200);

      .message-text--metadata .time {
        color: var(--r-50);
      }
    }
  }

  &.is-pending {
    position: relative;
    opacity: 0.8;

    .spinner {
      position: absolute;
      bottom: var(--space-smaller);
      right: var(--space-smaller);
    }

    > .is-image.is-text.bubble > .message-text__wrap {
      padding: 0;
    }
  }
}

.sender--info {
  align-items: center;
  color: var(--b-700);
  display: inline-flex;
  padding: var(--space-smaller) 0;

  .sender--available-name {
    font-size: var(--font-size-mini);
    margin-left: var(--space-smaller);
  }
}

.message-failed--alert {
  color: var(--r-900);
  flex-grow: 1;
  text-align: right;
  margin-top: var(--space-smaller) var(--space-smaller) 0 0;
}

.button--delete-message {
  visibility: hidden;
}

li.left,
li.right {
  display: flex;
  align-items: flex-end;

  &:hover .button--delete-message {
    visibility: visible;
  }
}

li.left.has-tweet-menu .context-menu {
  margin-bottom: var(--space-medium);
}

li.right .context-menu-wrap {
  margin-left: auto;
}

li.right {
  flex-direction: row-reverse;
  justify-content: flex-end;

  .wrap.is-pending {
    margin-left: auto;
  }

  .wrap.is-failed {
    display: flex;
    flex-direction: row-reverse;
    align-items: flex-end;
    margin-left: auto;
  }
}

.has-context-menu {
  background: var(--color-background);
  .button--delete-message {
    visibility: visible;
  }
}

.context-menu {
  position: relative;
}
</style>
