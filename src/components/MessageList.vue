<template>
    <div class="ww-message-list">
        <!-- Loading Skeleton -->
        <div v-if="isLoading" class="ww-message-list__skeleton">
            <!-- Date separator skeleton -->
            <div class="ww-message-list__skeleton-date" :style="dateSeparatorStyle">
                <span class="ww-message-list__skeleton-date-text"></span>
            </div>

            <!-- Other user message - short -->
            <div class="ww-message-list__skeleton-message ww-message-list__skeleton-message--other">
                <div class="ww-message-list__skeleton-bubble ww-message-list__skeleton-bubble--other ww-message-list__skeleton-bubble--short" :style="skeletonOtherStyle"></div>
            </div>

            <!-- Other user message - medium -->
            <div class="ww-message-list__skeleton-message ww-message-list__skeleton-message--other">
                <div class="ww-message-list__skeleton-bubble ww-message-list__skeleton-bubble--other ww-message-list__skeleton-bubble--medium" :style="skeletonOtherStyle"></div>
            </div>

            <!-- Own message - long -->
            <div class="ww-message-list__skeleton-message ww-message-list__skeleton-message--own">
                <div class="ww-message-list__skeleton-bubble ww-message-list__skeleton-bubble--own ww-message-list__skeleton-bubble--long" :style="skeletonOwnStyle"></div>
            </div>

            <!-- Other user message - medium -->
            <div class="ww-message-list__skeleton-message ww-message-list__skeleton-message--other">
                <div class="ww-message-list__skeleton-bubble ww-message-list__skeleton-bubble--other ww-message-list__skeleton-bubble--medium" :style="skeletonOtherStyle"></div>
            </div>

            <!-- Own message - short -->
            <div class="ww-message-list__skeleton-message ww-message-list__skeleton-message--own">
                <div class="ww-message-list__skeleton-bubble ww-message-list__skeleton-bubble--own ww-message-list__skeleton-bubble--short" :style="skeletonOwnStyle"></div>
            </div>

            <!-- Other user message - long -->
            <div class="ww-message-list__skeleton-message ww-message-list__skeleton-message--other">
                <div class="ww-message-list__skeleton-bubble ww-message-list__skeleton-bubble--other ww-message-list__skeleton-bubble--long" :style="skeletonOtherStyle"></div>
            </div>

            <!-- Own message - medium -->
            <div class="ww-message-list__skeleton-message ww-message-list__skeleton-message--own">
                <div class="ww-message-list__skeleton-bubble ww-message-list__skeleton-bubble--own ww-message-list__skeleton-bubble--medium" :style="skeletonOwnStyle"></div>
            </div>
        </div>

        <!-- Empty state -->
        <div v-else-if="messages.length === 0" class="ww-message-list__empty">
            <div class="ww-message-list__empty-message" :style="emptyMessageStyle">{{ emptyMessageText }}</div>
        </div>

        <!-- Messages -->
        <transition-group v-else name="message-transition" tag="div" ref="messagesContainerRef">
            <div 
                v-for="(message, index) in groupedMessages" 
                :key="message.key"
                :data-message-id="message.type !== 'date-separator' ? message.id : undefined"
            >
                <!-- Date separator -->
                <div
                    v-if="message.type === 'date-separator'"
                    class="ww-message-list__date-separator"
                    :style="dateSeparatorStyle"
                >
                    <span>{{ message.date }}</span>
                </div>

                <!-- Message item -->
                <message-item
                    v-else
                    :message="message"
                    :is-own-message="message.senderId === currentUserId"
                    :same-sender-as-previous="isSameSenderAsPrevious(index)"
                    :same-sender-as-next="isSameSenderAsNext(index)"
                    :message-bg-color="messageBgColor"
                    :message-text-color="messageTextColor"
                    :message-font-size="messageFontSize"
                    :message-font-weight="messageFontWeight"
                    :message-font-family="messageFontFamily"
                    :message-border="messageBorder"
                    :message-radius="messageRadius"
                    :own-message-bg-color="ownMessageBgColor"
                    :own-message-text-color="ownMessageTextColor"
                    :own-message-font-size="ownMessageFontSize"
                    :own-message-font-weight="ownMessageFontWeight"
                    :own-message-font-family="ownMessageFontFamily"
                    :own-message-border="ownMessageBorder"
                    :own-message-radius="ownMessageRadius"
                    :mentions-color="mentionsColor"
                    :mentions-bg-color="mentionsBgColor"
                    :reply-to-message="findReplyToMessage(message.replyToMessageId)"
                    @attachment-click="handleAttachmentClick"
                    @right-click="handleRightClick"
                    @reply-click="handleReplyClick"
                />
            </div>
        </transition-group>
    </div>
</template>

<script>
import { computed, inject, ref } from 'vue';
import MessageItem from './MessageItem.vue';
import { formatDate } from '../utils/dateTimeFormatter';

export default {
    name: 'MessageList',
    components: {
        MessageItem,
    },
    props: {
        messages: {
            type: Array,
            default: () => [],
        },
        currentUserId: {
            type: String,
            required: true,
        },
        isLoading: {
            type: Boolean,
            default: false,
        },
        messageBgColor: {
            type: String,
            default: '#f1f5f9',
        },
        messageTextColor: {
            type: String,
            default: '#334155',
        },
        messageFontSize: {
            type: String,
            default: '0.875rem',
        },
        messageFontWeight: {
            type: String,
            default: '400',
        },
        messageFontFamily: {
            type: String,
            default: 'inherit',
        },
        messageBorder: {
            type: String,
            default: '1px solid #e2e8f0',
        },
        messageRadius: {
            type: String,
            default: '18px 18px 18px 18px',
        },
        ownMessageBgColor: {
            type: String,
            default: '#dbeafe',
        },
        ownMessageTextColor: {
            type: String,
            default: '#1e40af',
        },
        ownMessageFontSize: {
            type: String,
            default: '0.875rem',
        },
        ownMessageFontWeight: {
            type: String,
            default: '400',
        },
        ownMessageFontFamily: {
            type: String,
            default: 'inherit',
        },
        ownMessageBorder: {
            type: String,
            default: '1px solid #bfdbfe',
        },
        ownMessageRadius: {
            type: String,
            default: '18px 18px 18px 18px',
        },
        emptyMessageText: {
            type: String,
            default: 'No messages yet',
        },
        emptyMessageColor: {
            type: String,
            default: '#64748b',
        },
        dateSeparatorTextColor: {
            type: String,
            default: '#64748b',
        },
        dateSeparatorLineColor: {
            type: String,
            default: '#e2e8f0',
        },
        dateSeparatorBgColor: {
            type: String,
            default: '#ffffff',
        },
        dateSeparatorBorderRadius: {
            type: String,
            default: '4px',
        },
        mentionsColor: {
            type: String,
            default: '#3b82f6',
        },
        mentionsBgColor: {
            type: String,
            default: '#dbeafe',
        },
    },
    emits: ['attachment-click', 'message-right-click'],
    setup(props, { emit }) {
        const isEditing = inject(
            'isEditing',
            computed(() => false)
        );

        const dateTimeOptions = inject(
            'dateTimeOptions',
            computed(() => ({}))
        );

        const messagesContainerRef = ref(null);

        const emptyMessageStyle = computed(() => ({
            color: props.emptyMessageColor,
        }));

        const dateSeparatorStyle = computed(() => ({
            '--date-separator-text-color': props.dateSeparatorTextColor,
            '--date-separator-line-color': props.dateSeparatorLineColor,
            '--date-separator-bg-color': props.dateSeparatorBgColor,
            '--date-separator-border-radius': props.dateSeparatorBorderRadius,
        }));

        const skeletonOtherStyle = computed(() => ({
            '--skeleton-bg': props.messageBgColor,
            '--skeleton-border': props.messageBorder,
            '--skeleton-radius': props.messageRadius,
        }));

        const skeletonOwnStyle = computed(() => ({
            '--skeleton-bg': props.ownMessageBgColor,
            '--skeleton-border': props.ownMessageBorder,
            '--skeleton-radius': props.ownMessageRadius,
        }));

        const groupedMessages = computed(() => {
            if (!props.messages || props.messages.length === 0) return [];

            const result = [];
            let currentDate = null;

            props.messages.forEach(message => {
                const messageDate = message.timestamp
                    ? new Date(message.timestamp).toDateString()
                    : new Date().toDateString();

                if (messageDate !== currentDate) {
                    currentDate = messageDate;
                    result.push({
                        type: 'date-separator',
                        date: formatDate(message.timestamp, dateTimeOptions.value),
                        key: `date-${messageDate}`,
                    });
                }

                result.push({
                    ...message,
                    key: message.id || `msg-${wwLib.wwUtils.getUid()}`,
                });
            });

            return result;
        });

        const isSameSenderAsPrevious = index => {
            if (index === 0) return false;

            const currentMessage = groupedMessages.value[index];
            if (currentMessage.type === 'date-separator') return false;

            let prevIndex = index - 1;
            while (prevIndex >= 0) {
                const prevMessage = groupedMessages.value[prevIndex];
                if (prevMessage.type !== 'date-separator') {
                    return currentMessage.senderId === prevMessage.senderId;
                }
                prevIndex--;
            }

            return false;
        };

        const isSameSenderAsNext = index => {
            if (index === groupedMessages.value.length - 1) return false;

            const currentMessage = groupedMessages.value[index];
            if (currentMessage.type === 'date-separator') return false;

            let nextIndex = index + 1;
            while (nextIndex < groupedMessages.value.length) {
                const nextMessage = groupedMessages.value[nextIndex];
                if (nextMessage.type !== 'date-separator') {
                    return currentMessage.senderId === nextMessage.senderId;
                }
                nextIndex++;
            }

            return false;
        };

        const handleAttachmentClick = attachment => {
            if (isEditing.value) return;
            emit('attachment-click', attachment);
        };

        const handleRightClick = ({ message, elementX, elementY, viewportX, viewportY }) => {
            if (isEditing.value) return;
            emit('message-right-click', {
                message,
                position: {
                    elementX,
                    elementY,
                    viewportX,
                    viewportY,
                },
            });
        };

        // Find the message being replied to by its ID
        const findReplyToMessage = (replyToMessageId) => {
            if (!replyToMessageId || !props.messages || props.messages.length === 0) return null;
            return props.messages.find(msg => msg.id === replyToMessageId) || null;
        };

        // Handle click on reply preview - scroll to the original message
        const handleReplyClick = (messageId) => {
            if (isEditing.value || !messageId) return;
            
            // Find the message element by data-message-id
            const container = messagesContainerRef.value?.$el || messagesContainerRef.value;
            if (!container) return;
            
            const messageEl = container.querySelector(`[data-message-id="${messageId}"]`);
            if (!messageEl) return;
            
            // Get the scrollable parent (messages container in wwElement.vue)
            const scrollableParent = messageEl.closest('.ww-chat__messages');
            if (!scrollableParent) return;
            
            // Calculate scroll position to center the message in the viewport
            const containerRect = scrollableParent.getBoundingClientRect();
            const messageRect = messageEl.getBoundingClientRect();
            const scrollTop = scrollableParent.scrollTop + messageRect.top - containerRect.top - (containerRect.height / 2) + (messageRect.height / 2);
            
            // Scroll to the message
            scrollableParent.scrollTo({
                top: Math.max(0, scrollTop),
                behavior: 'smooth',
            });
            
            // Add highlight animation
            messageEl.classList.add('ww-message-highlight');
            setTimeout(() => {
                messageEl.classList.remove('ww-message-highlight');
            }, 1500);
        };

        return {
            groupedMessages,
            findReplyToMessage,
            isSameSenderAsPrevious,
            isSameSenderAsNext,
            handleAttachmentClick,
            handleRightClick,
            handleReplyClick,
            emptyMessageStyle,
            dateSeparatorStyle,
            skeletonOtherStyle,
            skeletonOwnStyle,
            dateTimeOptions,
            messagesContainerRef,
        };
    },
};
</script>

<style lang="scss" scoped>
.ww-message-list {
    display: flex;
    flex-direction: column;

    &__empty {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        opacity: 0.5;
    }

    &__empty-message {
        font-size: 0.875rem;
        font-style: italic;
    }

    &__date-separator {
        display: flex;
        align-items: center;
        justify-content: center;
        margin: 16px 0;
        position: relative;

        &::before,
        &::after {
            content: '';
            flex: 1;
            height: 1px;
            background-color: var(--date-separator-line-color, #e2e8f0);
        }

        span {
            padding: 0 12px;
            font-size: 0.75rem;
            color: var(--date-separator-text-color, #64748b);
            background-color: var(--date-separator-bg-color, #ffffff);
            border-radius: var(--date-separator-border-radius, 4px);
        }
    }

    // Skeleton styles
    &__skeleton {
        display: flex;
        flex-direction: column;
        gap: 8px;
    }

    &__skeleton-date {
        display: flex;
        align-items: center;
        justify-content: center;
        margin: 16px 0;
        position: relative;

        &::before,
        &::after {
            content: '';
            flex: 1;
            height: 1px;
            background-color: var(--date-separator-line-color, #e2e8f0);
        }
    }

    &__skeleton-date-text {
        display: inline-block;
        width: 80px;
        height: 16px;
        margin: 0 12px;
        background: linear-gradient(90deg, var(--date-separator-bg-color, #f1f5f9) 25%, #e2e8f0 50%, var(--date-separator-bg-color, #f1f5f9) 75%);
        background-size: 200% 100%;
        animation: skeleton-shimmer 1.5s infinite;
        border-radius: var(--date-separator-border-radius, 4px);
    }

    &__skeleton-message {
        display: flex;
        margin-bottom: 4px;

        &--own {
            justify-content: flex-end;
        }

        &--other {
            justify-content: flex-start;
        }
    }

    &__skeleton-bubble {
        padding: 10px 12px;
        position: relative;
        box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
        background: var(--skeleton-bg, #f1f5f9);
        border: var(--skeleton-border, 1px solid #e2e8f0);
        border-radius: var(--skeleton-radius, 18px);
        overflow: hidden;

        &::after {
            content: '';
            position: absolute;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            background: linear-gradient(90deg, transparent 25%, rgba(255, 255, 255, 0.4) 50%, transparent 75%);
            background-size: 200% 100%;
            animation: skeleton-shimmer 1.5s infinite;
        }

        &--short {
            width: 80px;
            height: 36px;
        }

        &--medium {
            width: 160px;
            height: 36px;
        }

        &--long {
            width: 220px;
            height: 52px;
        }

        &--other {
            margin-top: 8px;

            &:first-child {
                margin-top: 0;
            }
        }

        &--own {
            margin-top: 8px;

            &:first-child {
                margin-top: 0;
            }
        }
    }

    @keyframes skeleton-shimmer {
        0% {
            background-position: 200% 0;
        }
        100% {
            background-position: -200% 0;
        }
    }
}

// Message transition animations
.message-transition-enter-active,
.message-transition-leave-active {
    transition: all 0.3s ease;
}

.message-transition-enter-from,
.message-transition-leave-to {
    opacity: 0;
    transform: translateY(10px);
}

.message-transition-move {
    transition: transform 0.3s;
}

// Highlight animation when scrolling to a message via reply click
:deep(.ww-message-highlight) {
    animation: message-highlight-pulse 1.5s ease-out;
}

@keyframes message-highlight-pulse {
    0% {
        background-color: transparent;
    }
    20% {
        background-color: rgba(59, 130, 246, 0.15);
    }
    100% {
        background-color: transparent;
    }
}
</style>
