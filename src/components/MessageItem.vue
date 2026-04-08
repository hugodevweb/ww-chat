<template>
    <div
        class="ww-message-item"
        :class="{
            'ww-message-item--own': isOwnMessage,
            'ww-message-item--continued': sameSenderAsPrevious,
            'ww-message-item--continue-next': sameSenderAsNext,
            'ww-message-item--has-avatar': !isOwnMessage && !sameSenderAsPrevious,
        }"
    >
        <!-- Avatar for other users (only shown on first message in group) -->
        <div 
            v-if="!isOwnMessage && !sameSenderAsPrevious" 
            class="ww-message-item__avatar"
        >
            <img 
                v-if="message.avatar" 
                :src="message.avatar" 
                :alt="message.userName"
                class="ww-message-item__avatar-img"
            />
            <span 
                v-else 
                class="ww-message-item__avatar-initials"
            >
                {{ userInitials }}
            </span>
        </div>
        <!-- Avatar spacer for continued messages (keeps alignment) -->
        <div 
            v-else-if="!isOwnMessage && sameSenderAsPrevious" 
            class="ww-message-item__avatar-spacer"
        ></div>

        <!-- Message content -->
        <div
            class="ww-message-item__content"
            :class="{ 'ww-message-item__content--own': isOwnMessage }"
            :style="messageStyles"
            @contextmenu.prevent="handleRightClick"
        >
            <!-- Sender name if first in group -->
            <div
                v-if="!sameSenderAsPrevious"
                class="ww-message-item__sender"
                :class="{ 'ww-message-item__sender--own': isOwnMessage }"
            >
                {{ message.userName }}
            </div>

            <!-- Reply preview if this is a reply -->
            <div 
                v-if="replyToMessage" 
                class="ww-message-item__reply-preview"
                :class="{ 'ww-message-item__reply-preview--own': isOwnMessage }"
                @click="handleReplyClick"
            >
                <div class="ww-message-item__reply-preview-bar"></div>
                <div class="ww-message-item__reply-preview-content">
                    <span class="ww-message-item__reply-preview-sender">{{ replyToMessage.userName }}</span>
                    <span class="ww-message-item__reply-preview-text">{{ truncateText(replyToMessage.text, 100) }}</span>
                </div>
            </div>

            <!-- Message text -->
            <div class="ww-message-item__text" v-html="highlightedMessageText"></div>

            <!-- Attachments if any -->
            <div v-if="formattedAttachments.length" class="ww-message-item__attachments">
                <div
                    v-for="(attachmentMeta, index) in formattedAttachments"
                    :key="attachmentMeta.attachment.id ?? index"
                    :ref="el => registerAttachmentRef(el, attachmentMeta.attachment, index)"
                    class="ww-message-item__attachment"
                    :class="{ 'ww-message-item__attachment--own': isOwnMessage }"
                    @click="handleAttachmentClick(attachmentMeta.attachment)"
                >
                    <!-- Skeleton: shown while waiting for a signed URL -->
                    <template v-if="!resolveAttachmentUrl(attachmentMeta.attachment)">
                        <!-- Image skeleton -->
                        <div
                            v-if="attachmentMeta.isImage"
                            class="ww-message-item__attachment-skeleton ww-message-item__attachment-skeleton--image"
                        />
                        <!-- File skeleton -->
                        <div
                            v-else
                            class="ww-message-item__attachment-skeleton ww-message-item__attachment-skeleton--file"
                        >
                            <div class="ww-message-item__attachment-skeleton__icon" />
                            <div class="ww-message-item__attachment-skeleton__info">
                                <div class="ww-message-item__attachment-skeleton__line ww-message-item__attachment-skeleton__line--name" />
                                <div class="ww-message-item__attachment-skeleton__line ww-message-item__attachment-skeleton__line--size" />
                            </div>
                        </div>
                    </template>

                    <!-- Image preview for image files -->
                    <template v-else>
                        <div
                            v-if="attachmentMeta.isImage"
                            class="ww-message-item__attachment-preview"
                            :class="{ 'ww-message-item__attachment-preview--own': isOwnMessage }"
                        >
                            <img :src="resolveAttachmentUrl(attachmentMeta.attachment)" :alt="attachmentMeta.attachment.name" />
                        </div>

                        <!-- File icon for non-image files -->
                        <div
                            v-else
                            class="ww-message-item__attachment-file"
                            :class="{ 'ww-message-item__attachment-file--own': isOwnMessage }"
                        >
                            <div class="ww-message-item__attachment-icon">
                                <svg
                                    xmlns="http://www.w3.org/2000/svg"
                                    width="24"
                                    height="24"
                                    viewBox="0 0 24 24"
                                    fill="none"
                                    stroke="currentColor"
                                    stroke-width="2"
                                    stroke-linecap="round"
                                    stroke-linejoin="round"
                                >
                                    <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path>
                                    <polyline points="14 2 14 8 20 8"></polyline>
                                    <line x1="16" y1="13" x2="8" y2="13"></line>
                                </svg>
                            </div>
                            <div class="ww-message-item__attachment-info">
                                <div class="ww-message-item__attachment-name">{{ attachmentMeta.attachment.name }}</div>
                                <div v-if="attachmentMeta.formattedSize" class="ww-message-item__attachment-size">
                                    {{ attachmentMeta.formattedSize }}
                                </div>
                            </div>
                        </div>
                    </template>
                </div>
            </div>

            <!-- Message timestamp and modified indicator -->
            <div class="ww-message-item__footer">
                <span 
                    v-if="isModified" 
                    class="ww-message-item__modified"
                    :title="modifiedTooltip"
                >
                    modifié
                </span>
                <span v-if="isModified" class="ww-message-item__footer-separator">·</span>
                <span class="ww-message-item__time">
                    {{ formatMessageTime(message.timestamp) }}
                </span>
            </div>
        </div>
    </div>
</template>

<script>
import { computed, inject, onMounted, onUnmounted } from 'vue';
import { formatTime, formatDateTime } from '../utils/dateTimeFormatter';

export default {
    name: 'MessageItem',
    props: {
        message: {
            type: Object,
            required: true,
        },
        isOwnMessage: {
            type: Boolean,
            default: false,
        },
        sameSenderAsPrevious: {
            type: Boolean,
            default: false,
        },
        sameSenderAsNext: {
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
        messageRadius: {
            type: String,
            default: '18px 18px 18px 18px',
        },
        ownMessageRadius: {
            type: String,
            default: '18px 18px 18px 18px',
        },
        mentionsColor: {
            type: String,
            default: '#3b82f6',
        },
        mentionsBgColor: {
            type: String,
            default: '#dbeafe',
        },
        replyToMessage: {
            type: Object,
            default: null,
        },
        signedUrlsMap: {
            type: Array,
            default: () => [],
        },
    },
    emits: ['attachment-click', 'right-click', 'reply-click', 'attachment-visible'],
    setup(props, { emit }) {
        // IntersectionObserver — fires 'attachment-visible' when an attachment enters
        // the viewport and has no resolved URL yet (not in signedUrlsMap).
        let attachmentObserver = null;
        const observedAttachments = new Map(); // element → { attachment, index }

        const registerAttachmentRef = (el, attachment, index) => {
            if (!el) return;
            observedAttachments.set(el, { attachment, index });
            if (attachmentObserver) attachmentObserver.observe(el);
        };

        onMounted(() => {
            attachmentObserver = new IntersectionObserver(
                entries => {
                    entries.forEach(entry => {
                        if (!entry.isIntersecting) return;
                        const data = observedAttachments.get(entry.target);
                        if (!data) return;
                        const { attachment } = data;
                        // Only fire if no URL is resolved yet (neither from the array nor the data)
                        const resolved = props.signedUrlsMap?.find?.(e => e.id === attachment.id);
                        if (!resolved?.url && !attachment.url) {
                            emit('attachment-visible', attachment);
                        }
                    });
                },
                { threshold: 0.1 }
            );
            // Observe elements registered before mount (fast renders)
            observedAttachments.forEach((_, el) => attachmentObserver.observe(el));
        });

        onUnmounted(() => {
            if (attachmentObserver) {
                attachmentObserver.disconnect();
                attachmentObserver = null;
            }
            observedAttachments.clear();
        });

        const isEditing = inject(
            'isEditing',
            computed(() => false)
        );
        const chatRootEl = inject('chatRootEl', null);

        const dateTimeOptions = inject(
            'dateTimeOptions',
            computed(() => ({}))
        );

        const messageStyles = computed(() => {
            if (props.isOwnMessage) {
                return {
                    backgroundColor: props.ownMessageBgColor,
                    color: props.ownMessageTextColor,
                    fontSize: props.ownMessageFontSize,
                    fontWeight: props.ownMessageFontWeight,
                    fontFamily: props.ownMessageFontFamily,
                    border: props.ownMessageBorder,
                    '--message-radius': props.ownMessageRadius,
                };
            } else {
                return {
                    backgroundColor: props.messageBgColor,
                    color: props.messageTextColor,
                    fontSize: props.messageFontSize,
                    fontWeight: props.messageFontWeight,
                    fontFamily: props.messageFontFamily,
                    border: props.messageBorder,
                    '--message-radius': props.messageRadius,
                };
            }
        });

        const isImageFile = attachment => {
            if (!attachment.type) return false;
            return attachment.type.startsWith('image/');
        };

        const formatFileSize = rawSize => {
            const bytes = Number(rawSize);
            if (!Number.isFinite(bytes) || bytes < 0) return '';
            if (bytes === 0) return '0 Bytes';

            const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB'];
            const index = Math.min(Math.floor(Math.log(bytes) / Math.log(1024)), sizes.length - 1);
            const value = bytes / Math.pow(1024, index);
            return `${parseFloat(value.toFixed(2))} ${sizes[index]}`;
        };

        const formattedAttachments = computed(() => {
            const attachments = Array.isArray(props.message.attachments) ? props.message.attachments : [];
            return attachments.map(attachment => ({
                attachment,
                isImage: isImageFile(attachment),
                formattedSize: formatFileSize(attachment.size),
            }));
        });

        const formatMessageTime = timestamp => {
            return formatTime(timestamp, dateTimeOptions.value);
        };

        const handleAttachmentClick = attachment => {
            if (isEditing.value) return;
            emit('attachment-click', attachment);
        };

        const handleRightClick = event => {
            // Get the message content element's bounding rect
            const messageContentEl = event.currentTarget;
            const messageRect = messageContentEl.getBoundingClientRect();
            
            // Coordinates relative to the chat root element
            let elementX = 0;
            let elementY = 0;
            const root = chatRootEl && chatRootEl.value ? chatRootEl.value : null;
            if (root && typeof root.getBoundingClientRect === 'function') {
                const chatRect = root.getBoundingClientRect();
                elementX = event.clientX - chatRect.left;
                elementY = event.clientY - chatRect.top;
            } else {
                // Fallback: treat client coords as element-relative if root not found
                elementX = event.clientX;
                elementY = event.clientY;
            }

            // Calculate position centered over the message
            // Position at the horizontal center of the message, vertically at the top
            const viewportX = messageRect.left + (messageRect.width / 2);
            const viewportY = messageRect.top + (messageRect.height / 2);

            emit('right-click', {
                message: props.message,
                elementX,
                elementY,
                viewportX,
                viewportY,
                messageRect: {
                    top: messageRect.top,
                    left: messageRect.left,
                    right: messageRect.right,
                    bottom: messageRect.bottom,
                    width: messageRect.width,
                    height: messageRect.height,
                },
            });
        };

        const handleReplyClick = () => {
            if (isEditing.value) return;
            if (props.replyToMessage?.id) {
                emit('reply-click', props.replyToMessage.id);
            }
        };

        // Helper function to escape HTML and convert newlines to <br>
        const escapeHtmlAndConvertNewlines = (str) => {
            return str
                .replace(/&/g, '&amp;')
                .replace(/</g, '&lt;')
                .replace(/>/g, '&gt;')
                .replace(/\n/g, '<br>');
        };

        const highlightedMessageText = computed(() => {
            const text = props.message?.text || '';
            if (!text) return '';
            
            // Get mentions from message data if available
            const mentions = props.message?.mentions || [];
            
            // If we have mentions data, use it for precise highlighting
            if (mentions.length > 0) {
                // Find all mention positions in the text
                const mentionOccurrences = [];
                
                mentions.forEach(mention => {
                    const mentionPattern = `@${mention.name}`;
                    let index = text.indexOf(mentionPattern);
                    
                    while (index !== -1) {
                        mentionOccurrences.push({
                            start: index,
                            end: index + mentionPattern.length,
                            text: mentionPattern,
                            mention: mention
                        });
                        index = text.indexOf(mentionPattern, index + 1);
                    }
                });
                
                // Sort by position (earliest first)
                mentionOccurrences.sort((a, b) => a.start - b.start);
                
                // Build the result string
                const parts = [];
                let lastIndex = 0;
                
                mentionOccurrences.forEach(occurrence => {
                    // Add text before this mention (escape HTML and convert newlines)
                    if (occurrence.start > lastIndex) {
                        const beforeText = text.substring(lastIndex, occurrence.start);
                        parts.push(escapeHtmlAndConvertNewlines(beforeText));
                    }
                    
                    // Add the highlighted mention (don't convert newlines in mentions)
                    const escapedMention = occurrence.text
                        .replace(/&/g, '&amp;')
                        .replace(/</g, '&lt;')
                        .replace(/>/g, '&gt;');
                    
                    parts.push(`<span class="ww-message-item__mention" style="background-color: ${props.mentionsBgColor}; color: ${props.mentionsColor};">${escapedMention}</span>`);
                    
                    lastIndex = occurrence.end;
                });
                
                // Add remaining text after last mention (escape HTML and convert newlines)
                if (lastIndex < text.length) {
                    const remainingText = text.substring(lastIndex);
                    parts.push(escapeHtmlAndConvertNewlines(remainingText));
                }
                
                return parts.join('');
            }
            
            // Fallback: use regex for generic @mention detection (up to 2 words)
            const mentionRegex = /@([a-zA-Z0-9_]+(?:\s+[a-zA-Z0-9_]+)?)(?=\s+[a-z]|\s*[,.:;!?)\n]|$)/g;
            const parts = [];
            let lastIndex = 0;
            let match;
            
            while ((match = mentionRegex.exec(text)) !== null) {
                // Add text before the mention (escape HTML and convert newlines)
                if (match.index > lastIndex) {
                    const beforeText = text.substring(lastIndex, match.index);
                    parts.push(escapeHtmlAndConvertNewlines(beforeText));
                }
                
                // Add the mention with styling (escape the mention text, don't convert newlines)
                const mentionText = match[0]
                    .replace(/&/g, '&amp;')
                    .replace(/</g, '&lt;')
                    .replace(/>/g, '&gt;');
                parts.push(`<span class="ww-message-item__mention" style="background-color: ${props.mentionsBgColor}; color: ${props.mentionsColor};">${mentionText}</span>`);
                
                lastIndex = match.index + match[0].length;
            }
            
            // Add remaining text after last mention (escape HTML and convert newlines)
            if (lastIndex < text.length) {
                const remainingText = text.substring(lastIndex);
                parts.push(escapeHtmlAndConvertNewlines(remainingText));
            }
            
            return parts.join('');
        });

        // Compute initials for avatar fallback
        const userInitials = computed(() => {
            const name = props.message?.userName || '';
            if (!name) return '?';
            
            const words = name.trim().split(/\s+/);
            if (words.length === 1) {
                return words[0].charAt(0).toUpperCase();
            }
            return (words[0].charAt(0) + words[words.length - 1].charAt(0)).toUpperCase();
        });

        const truncateText = (text, maxLength = 100) => {
            if (!text) return '';
            if (text.length <= maxLength) return text;
            return text.slice(0, maxLength) + '...';
        };

        // Check if message was modified (lastModifiedAt differs from timestamp)
        const isModified = computed(() => {
            const { timestamp, lastModifiedAt } = props.message;
            if (!lastModifiedAt || !timestamp) return false;
            
            const createdDate = new Date(timestamp);
            const modifiedDate = new Date(lastModifiedAt);
            
            // Check if dates are valid
            if (isNaN(createdDate.getTime()) || isNaN(modifiedDate.getTime())) return false;
            
            // Compare timestamps (allow 1 second tolerance for potential timing issues)
            return Math.abs(modifiedDate.getTime() - createdDate.getTime()) > 1000;
        });

        // Format the modified date for tooltip
        const modifiedTooltip = computed(() => {
            if (!props.message.lastModifiedAt) return '';
            return `Modifié le ${formatDateTime(props.message.lastModifiedAt, dateTimeOptions.value)}`;
        });

        // Returns the best available URL: signed (from array) → original → null
        const resolveAttachmentUrl = attachment => {
            const entry = props.signedUrlsMap?.find?.(e => e.id === attachment.id);
            return entry?.url || attachment.url || null;
        };

        return {
            messageStyles,
            isImageFile,
            formatFileSize,
            formattedAttachments,
            formatMessageTime,
            handleAttachmentClick,
            handleRightClick,
            handleReplyClick,
            highlightedMessageText,
            userInitials,
            isModified,
            modifiedTooltip,
            truncateText,
            registerAttachmentRef,
            resolveAttachmentUrl,
        };
    },
};
</script>

<style lang="scss" scoped>
.ww-message-item {
    display: flex;
    margin-bottom: 4px;
    align-items: flex-start;

    &--own {
        justify-content: flex-end;
    }

    &__avatar {
        width: 32px;
        height: 32px;
        min-width: 32px;
        border-radius: 50%;
        margin-right: 8px;
        margin-top: 8px;
        overflow: hidden;
        display: flex;
        align-items: center;
        justify-content: center;
        background-color: #e2e8f0;
    }

    &__avatar-img {
        width: 100%;
        height: 100%;
        object-fit: cover;
    }

    &__avatar-initials {
        font-size: 0.75rem;
        font-weight: 600;
        color: #64748b;
        text-transform: uppercase;
    }

    &__avatar-spacer {
        width: 32px;
        min-width: 32px;
        margin-right: 8px;
    }

    &__content {
        max-width: 70%;
        padding: 10px 12px;
        border-radius: var(--message-radius, 18px 18px 18px 18px);
        position: relative;
        box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);

        &:not(.ww-message-item--continued) {
            margin-top: 8px;
        }

        .ww-message-item--continued & {
            margin-top: 2px;
        }

        .ww-message-item--continue-next & {
            margin-bottom: 2px;
        }
    }

    // Adjust max-width when avatar is present
    &--has-avatar &__content,
    &:not(&--own) &__content {
        max-width: calc(70% - 40px);
    }

    &__sender {
        font-weight: 600;
        font-size: 0.75rem;
        margin-bottom: 2px;
        opacity: 0.8;

        &--own {
            text-align: right;
        }
    }

    &__reply-preview {
        display: flex;
        gap: 8px;
        padding: 8px 10px;
        margin-bottom: 6px;
        background: rgba(0, 0, 0, 0.04);
        border-radius: 6px;
        cursor: pointer;
        transition: background-color 0.15s ease;

        &:hover {
            background: rgba(0, 0, 0, 0.06);
        }

        &--own {
            background: rgba(255, 255, 255, 0.2);

            &:hover {
                background: rgba(255, 255, 255, 0.3);
            }
        }
    }

    &__reply-preview-bar {
        width: 3px;
        background: #64748b;
        border-radius: 2px;
        flex-shrink: 0;
    }

    .ww-message-item__reply-preview--own &__reply-preview-bar {
        background: rgba(255, 255, 255, 0.6);
    }

    &__reply-preview-content {
        display: flex;
        flex-direction: column;
        gap: 2px;
        min-width: 0;
        flex: 1;
    }

    &__reply-preview-sender {
        font-size: 0.6875rem;
        font-weight: 600;
        color: #64748b;
    }

    .ww-message-item__reply-preview--own &__reply-preview-sender {
        color: rgba(255, 255, 255, 0.8);
    }

    &__reply-preview-text {
        font-size: 0.75rem;
        color: #64748b;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }

    .ww-message-item__reply-preview--own &__reply-preview-text {
        color: rgba(255, 255, 255, 0.7);
    }

    &__text {
        line-height: 1.4;
        word-break: break-word;

        :deep(.ww-message-item__mention) {
            border-radius: 4px;
            padding: 2px 4px;
            font-weight: 600;
            display: inline;
        }
    }

    &__footer {
        display: flex;
        align-items: center;
        justify-content: flex-end;
        gap: 4px;
        margin-top: 4px;
        font-size: 0.6875rem;
        opacity: 0.7;
        user-select: none;
    }

    &__modified {
        font-style: italic;
        cursor: help;
        position: relative;
        
        &:hover {
            opacity: 1;
        }
    }

    &__footer-separator {
        color: inherit;
    }

    &__time {
        // No additional styles needed, inherits from footer
    }

    &__attachments {
        margin-top: 8px;
        display: flex;
        flex-direction: column;
        gap: 8px;
        align-items: flex-start;

        .ww-message-item--own & {
            align-items: flex-end;
        }
    }

    &__attachment {
        border-radius: 8px;
        overflow: hidden;
        background-color: rgba(255, 255, 255, 0.1);
        cursor: pointer;

        &--own {
            align-self: flex-end;
        }
    }

    &__attachment-preview {
        max-width: var(--ww-chat-attachment-thumb-max-width, 250px);
        max-height: var(--ww-chat-attachment-thumb-max-height, 200px);
        display: flex;
        align-items: center;
        justify-content: center;

        &--own {
            justify-content: flex-end;
        }

        img {
            max-width: 100%;
            max-height: var(--ww-chat-attachment-thumb-max-height, 200px);
            object-fit: contain;
            border-radius: var(--ww-chat-attachment-thumb-radius, 6px);
        }
    }

    &__attachment-file {
        display: flex;
        align-items: center;
        padding: 8px;
        background-color: rgba(255, 255, 255, 0.15);
        border-radius: 6px;
        max-width: 200px;

        &:hover {
            opacity: 0.9;
        }

        &--own {
            flex-direction: row-reverse;

            .ww-message-item__attachment-icon {
                margin-right: 0;
                margin-left: 8px;
            }

            .ww-message-item__attachment-info {
                text-align: right;
            }
        }
    }

    &__attachment-icon {
        margin-right: 8px;
        display: flex;
        align-items: center;
        justify-content: center;
    }

    &__attachment-info {
        display: flex;
        flex-direction: column;
        overflow: hidden;
    }

    &__attachment-name {
        font-size: 0.8125rem;
        font-weight: 500;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
        max-width: 150px;
    }

    &__attachment-size {
        font-size: 0.75rem;
        opacity: 0.7;
    }

    // Skeleton shimmer
    &__attachment-skeleton {
        @keyframes ww-skeleton-shimmer {
            0%   { background-position: -400px 0; }
            100% { background-position:  400px 0; }
        }

        $shimmer: linear-gradient(90deg, rgba(255,255,255,0.06) 25%, rgba(255,255,255,0.18) 50%, rgba(255,255,255,0.06) 75%);

        border-radius: 6px;
        background: rgba(255, 255, 255, 0.1) $shimmer;
        background-size: 800px 100%;
        animation: ww-skeleton-shimmer 1.4s ease-in-out infinite;

        // Image skeleton — matches attachment-preview dimensions
        &--image {
            width: var(--ww-chat-attachment-thumb-max-width, 250px);
            height: 150px;
            border-radius: var(--ww-chat-attachment-thumb-radius, 6px);
        }

        // File skeleton — matches attachment-file pill
        &--file {
            display: flex;
            align-items: center;
            padding: 8px;
            width: 200px;
            background: rgba(255, 255, 255, 0.1) $shimmer;
            background-size: 800px 100%;
            animation: ww-skeleton-shimmer 1.4s ease-in-out infinite;
            border-radius: 6px;
        }

        &__icon {
            width: 24px;
            height: 24px;
            min-width: 24px;
            border-radius: 4px;
            margin-right: 8px;
            background: rgba(255, 255, 255, 0.15);
        }

        &__info {
            display: flex;
            flex-direction: column;
            gap: 6px;
            flex: 1;
            overflow: hidden;
        }

        &__line {
            border-radius: 3px;
            background: rgba(255, 255, 255, 0.15);

            &--name {
                height: 10px;
                width: 80%;
            }

            &--size {
                height: 8px;
                width: 40%;
            }
        }
    }
}
</style>
