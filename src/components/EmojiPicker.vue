<template>
    <div class="ww-emoji-picker" ref="pickerRef">
        <button 
            type="button"
            class="ww-emoji-picker__trigger"
            :class="{ 'ww-emoji-picker__trigger--active': isOpen }"
            @click="togglePicker"
            :disabled="disabled"
            title="Ajouter un emoji"
        >
            <svg 
                xmlns="http://www.w3.org/2000/svg" 
                width="20" 
                height="20" 
                viewBox="0 0 24 24" 
                fill="none" 
                stroke="currentColor" 
                stroke-width="2" 
                stroke-linecap="round" 
                stroke-linejoin="round"
            >
                <circle cx="12" cy="12" r="10"></circle>
                <path d="M8 14s1.5 2 4 2 4-2 4-2"></path>
                <line x1="9" y1="9" x2="9.01" y2="9"></line>
                <line x1="15" y1="9" x2="15.01" y2="9"></line>
            </svg>
        </button>

        <teleport to="body">
            <div 
                v-if="isOpen" 
                ref="popupRef"
                class="ww-emoji-picker__popup"
                :style="popupStyle"
            >
                <!-- Search -->
                <div class="ww-emoji-picker__search">
                    <input 
                        v-model="searchQuery"
                        type="text"
                        placeholder="Rechercher un emoji..."
                        class="ww-emoji-picker__search-input"
                        ref="searchInputRef"
                    />
                </div>

                <!-- Categories -->
                <div class="ww-emoji-picker__categories">
                    <button 
                        v-for="category in categories" 
                        :key="category.id"
                        type="button"
                        class="ww-emoji-picker__category-btn"
                        :class="{ 'ww-emoji-picker__category-btn--active': activeCategory === category.id }"
                        @click="activeCategory = category.id"
                        :title="category.name"
                    >
                        {{ category.icon }}
                    </button>
                </div>

                <!-- Emoji Grid -->
                <div class="ww-emoji-picker__grid-container">
                    <div class="ww-emoji-picker__grid">
                        <button 
                            v-for="emoji in filteredEmojis" 
                            :key="emoji"
                            type="button"
                            class="ww-emoji-picker__emoji"
                            @click="selectEmoji(emoji)"
                        >
                            {{ emoji }}
                        </button>
                        <div v-if="filteredEmojis.length === 0" class="ww-emoji-picker__empty">
                            Aucun emoji trouvé
                        </div>
                    </div>
                </div>
            </div>

            <!-- Backdrop -->
            <div 
                v-if="isOpen" 
                class="ww-emoji-picker__backdrop"
                @click="closePicker"
            ></div>
        </teleport>
    </div>
</template>

<script>
import { ref, computed, watch, onMounted, onUnmounted, nextTick } from 'vue';

export default {
    name: 'EmojiPicker',
    props: {
        disabled: {
            type: Boolean,
            default: false,
        },
    },
    emits: ['select'],
    setup(props, { emit }) {
        const isOpen = ref(false);
        const searchQuery = ref('');
        const activeCategory = ref('smileys');
        const pickerRef = ref(null);
        const popupRef = ref(null);
        const searchInputRef = ref(null);
        const popupStyle = ref({});

        // Get the correct window/document for iframe support
        const getFrontWindow = () => {
            return typeof wwLib !== 'undefined' && wwLib.getFrontWindow 
                ? wwLib.getFrontWindow() 
                : window;
        };

        const getFrontDocument = () => {
            return typeof wwLib !== 'undefined' && wwLib.getFrontDocument 
                ? wwLib.getFrontDocument() 
                : document;
        };

        const categories = [
            { id: 'smileys', name: 'Smileys', icon: '😀' },
            { id: 'people', name: 'Personnes', icon: '👋' },
            { id: 'animals', name: 'Animaux', icon: '🐶' },
            { id: 'food', name: 'Nourriture', icon: '🍕' },
            { id: 'activities', name: 'Activités', icon: '⚽' },
            { id: 'travel', name: 'Voyage', icon: '✈️' },
            { id: 'objects', name: 'Objets', icon: '💡' },
            { id: 'symbols', name: 'Symboles', icon: '❤️' },
        ];

        const emojiData = {
            smileys: ['😀', '😃', '😄', '😁', '😆', '😅', '🤣', '😂', '🙂', '🙃', '😉', '😊', '😇', '🥰', '😍', '🤩', '😘', '😗', '😚', '😙', '🥲', '😋', '😛', '😜', '🤪', '😝', '🤑', '🤗', '🤭', '🤫', '🤔', '🤐', '🤨', '😐', '😑', '😶', '😏', '😒', '🙄', '😬', '🤥', '😌', '😔', '😪', '🤤', '😴', '😷', '🤒', '🤕', '🤢', '🤮', '🤧', '🥵', '🥶', '🥴', '😵', '🤯', '🤠', '🥳', '🥸', '😎', '🤓', '🧐'],
            people: ['👋', '🤚', '🖐️', '✋', '🖖', '👌', '🤌', '🤏', '✌️', '🤞', '🤟', '🤘', '🤙', '👈', '👉', '👆', '🖕', '👇', '☝️', '👍', '👎', '✊', '👊', '🤛', '🤜', '👏', '🙌', '👐', '🤲', '🤝', '🙏', '✍️', '💅', '🤳', '💪', '🦾', '🦿', '🦵', '🦶', '👂', '🦻', '👃', '🧠', '🫀', '🫁', '🦷', '🦴', '👀', '👁️', '👅', '👄', '👶', '🧒', '👦', '👧', '🧑', '👱', '👨', '🧔', '👩', '🧓', '👴', '👵'],
            animals: ['🐶', '🐱', '🐭', '🐹', '🐰', '🦊', '🐻', '🐼', '🐻‍❄️', '🐨', '🐯', '🦁', '🐮', '🐷', '🐽', '🐸', '🐵', '🙈', '🙉', '🙊', '🐒', '🐔', '🐧', '🐦', '🐤', '🐣', '🐥', '🦆', '🦅', '🦉', '🦇', '🐺', '🐗', '🐴', '🦄', '🐝', '🪱', '🐛', '🦋', '🐌', '🐞', '🐜', '🪰', '🪲', '🪳', '🦟', '🦗', '🕷️', '🦂', '🐢', '🐍', '🦎', '🦖', '🦕', '🐙', '🦑', '🦐', '🦞', '🦀', '🐡', '🐠', '🐟', '🐬', '🐳'],
            food: ['🍕', '🍔', '🍟', '🌭', '🍿', '🧂', '🥓', '🥚', '🍳', '🧇', '🥞', '🧈', '🍞', '🥐', '🥖', '🥨', '🧀', '🥗', '🥙', '🥪', '🌮', '🌯', '🫔', '🥫', '🍝', '🍜', '🍲', '🍛', '🍣', '🍱', '🥟', '🦪', '🍤', '🍙', '🍚', '🍘', '🍥', '🥠', '🥮', '🍢', '🍡', '🍧', '🍨', '🍦', '🥧', '🧁', '🍰', '🎂', '🍮', '🍭', '🍬', '🍫', '🍿', '🍩', '🍪', '🌰', '🥜', '🍯', '🥛', '🍼', '☕', '🍵', '🧃'],
            activities: ['⚽', '🏀', '🏈', '⚾', '🥎', '🎾', '🏐', '🏉', '🥏', '🎱', '🪀', '🏓', '🏸', '🏒', '🏑', '🥍', '🏏', '🪃', '🥅', '⛳', '🪁', '🏹', '🎣', '🤿', '🥊', '🥋', '🎽', '🛹', '🛼', '🛷', '⛸️', '🥌', '🎿', '⛷️', '🏂', '🪂', '🏋️', '🤼', '🤸', '⛹️', '🤺', '🤾', '🏌️', '🏇', '🧘', '🏄', '🏊', '🤽', '🚣', '🧗', '🚵', '🚴', '🏆', '🥇', '🥈', '🥉', '🏅', '🎖️', '🏵️', '🎗️', '🎫', '🎟️', '🎪', '🎭'],
            travel: ['✈️', '🛫', '🛬', '🛩️', '💺', '🛰️', '🚀', '🛸', '🚁', '🛶', '⛵', '🚤', '🛥️', '🛳️', '⛴️', '🚢', '⚓', '🪝', '⛽', '🚧', '🚦', '🚥', '🚏', '🗺️', '🗿', '🗽', '🗼', '🏰', '🏯', '🏟️', '🎡', '🎢', '🎠', '⛲', '⛱️', '🏖️', '🏝️', '🏜️', '🌋', '⛰️', '🏔️', '🗻', '🏕️', '⛺', '🛖', '🏠', '🏡', '🏘️', '🏚️', '🏗️', '🏭', '🏢', '🏬', '🏣', '🏤', '🏥', '🏦', '🏨', '🏪', '🏫', '🏩', '💒', '🏛️', '⛪'],
            objects: ['💡', '🔦', '🏮', '🪔', '📱', '📲', '💻', '🖥️', '🖨️', '⌨️', '🖱️', '🖲️', '💽', '💾', '💿', '📀', '🧮', '🎥', '🎞️', '📽️', '🎬', '📺', '📷', '📸', '📹', '📼', '🔍', '🔎', '🕯️', '💰', '💴', '💵', '💶', '💷', '💎', '⚖️', '🪜', '🧰', '🪛', '🔧', '🔨', '⚒️', '🛠️', '⛏️', '🪚', '🔩', '⚙️', '🪤', '🧱', '⛓️', '🧲', '🔫', '💣', '🧨', '🪓', '🔪', '🗡️', '⚔️', '🛡️', '🚬', '⚰️', '🪦', '⚱️', '🏺'],
            symbols: ['❤️', '🧡', '💛', '💚', '💙', '💜', '🖤', '🤍', '🤎', '💔', '❣️', '💕', '💞', '💓', '💗', '💖', '💘', '💝', '💟', '☮️', '✝️', '☪️', '🕉️', '☸️', '✡️', '🔯', '🕎', '☯️', '☦️', '🛐', '⛎', '♈', '♉', '♊', '♋', '♌', '♍', '♎', '♏', '♐', '♑', '♒', '♓', '🆔', '⚛️', '🉑', '☢️', '☣️', '📴', '📳', '🈶', '🈚', '🈸', '🈺', '🈷️', '✴️', '🆚', '💮', '🉐', '㊙️', '㊗️', '🈴', '🈵', '🈹'],
        };

        const allEmojis = computed(() => {
            return Object.values(emojiData).flat();
        });

        const filteredEmojis = computed(() => {
            if (searchQuery.value.trim()) {
                // When searching, search all emojis
                return allEmojis.value.filter(emoji => 
                    emoji.includes(searchQuery.value)
                );
            }
            // When not searching, show category emojis
            return emojiData[activeCategory.value] || [];
        });

        const togglePicker = () => {
            if (props.disabled) return;
            isOpen.value = !isOpen.value;
            if (isOpen.value) {
                nextTick(() => {
                    updatePopupPosition();
                    searchInputRef.value?.focus();
                });
            }
        };

        const closePicker = () => {
            isOpen.value = false;
            searchQuery.value = '';
        };

        const selectEmoji = (emoji) => {
            emit('select', emoji);
            closePicker();
        };

        const updatePopupPosition = () => {
            if (!pickerRef.value || !popupRef.value) return;

            const triggerRect = pickerRef.value.getBoundingClientRect();
            const popupHeight = 350;
            const popupWidth = 320;
            const margin = 8;

            // Get viewport dimensions from the correct window (iframe-aware)
            const frontWindow = getFrontWindow();
            const viewportWidth = frontWindow.innerWidth;
            const viewportHeight = frontWindow.innerHeight;

            let top = triggerRect.top - popupHeight - margin;
            let left = triggerRect.left - popupWidth + triggerRect.width;

            // If popup would go above viewport, show below
            if (top < margin) {
                top = triggerRect.bottom + margin;
            }

            // Keep within viewport horizontally
            if (left < margin) {
                left = margin;
            }
            if (left + popupWidth > viewportWidth - margin) {
                left = viewportWidth - popupWidth - margin;
            }

            // Ensure popup doesn't go below viewport
            if (top + popupHeight > viewportHeight - margin) {
                top = viewportHeight - popupHeight - margin;
            }

            popupStyle.value = {
                top: `${top}px`,
                left: `${left}px`,
            };
        };

        const handleKeyDown = (event) => {
            if (event.key === 'Escape' && isOpen.value) {
                closePicker();
            }
        };

        const handleResize = () => {
            if (isOpen.value) {
                updatePopupPosition();
            }
        };

        onMounted(() => {
            const frontDoc = getFrontDocument();
            const frontWin = getFrontWindow();
            frontDoc.addEventListener('keydown', handleKeyDown);
            frontWin.addEventListener('resize', handleResize);
            frontWin.addEventListener('scroll', handleResize, true);
        });

        onUnmounted(() => {
            const frontDoc = getFrontDocument();
            const frontWin = getFrontWindow();
            frontDoc.removeEventListener('keydown', handleKeyDown);
            frontWin.removeEventListener('resize', handleResize);
            frontWin.removeEventListener('scroll', handleResize, true);
        });

        return {
            isOpen,
            searchQuery,
            activeCategory,
            categories,
            filteredEmojis,
            pickerRef,
            popupRef,
            searchInputRef,
            popupStyle,
            togglePicker,
            closePicker,
            selectEmoji,
        };
    },
};
</script>

<style lang="scss" scoped>
.ww-emoji-picker {
    position: relative;
    display: inline-flex;

    &__trigger {
        display: flex;
        align-items: center;
        justify-content: center;
        width: 28px;
        height: 28px;
        padding: 0;
        background: transparent;
        border: none;
        border-radius: 6px;
        color: #94a3b8;
        cursor: pointer;
        transition: all 0.15s ease;

        &:hover:not(:disabled) {
            background: #f1f5f9;
            color: #64748b;
        }

        &--active {
            background: #e2e8f0;
            color: #475569;
        }

        &:disabled {
            opacity: 0.4;
            cursor: not-allowed;
        }

        svg {
            width: 18px;
            height: 18px;
        }
    }

    &__popup {
        position: fixed;
        z-index: 10001;
        width: 320px;
        height: 350px;
        background: #ffffff;
        border: 1px solid #e2e8f0;
        border-radius: 12px;
        box-shadow: 0 10px 40px rgba(0, 0, 0, 0.15), 0 2px 10px rgba(0, 0, 0, 0.1);
        display: flex;
        flex-direction: column;
        animation: emojiPickerFadeIn 0.15s ease-out;
        overflow: hidden;
    }

    &__backdrop {
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        z-index: 10000;
        background: transparent;
    }

    &__search {
        padding: 10px 12px;
        border-bottom: 1px solid #f1f5f9;
    }

    &__search-input {
        width: 100%;
        padding: 8px 12px;
        border: 1px solid #e2e8f0;
        border-radius: 8px;
        font-size: 13px;
        outline: none;
        transition: all 0.15s ease;

        &:focus {
            border-color: #3b82f6;
            box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
        }

        &::placeholder {
            color: #94a3b8;
        }
    }

    &__categories {
        display: flex;
        gap: 2px;
        padding: 8px 10px;
        border-bottom: 1px solid #f1f5f9;
        overflow-x: auto;

        &::-webkit-scrollbar {
            display: none;
        }
    }

    &__category-btn {
        display: flex;
        align-items: center;
        justify-content: center;
        width: 32px;
        height: 32px;
        min-width: 32px;
        padding: 0;
        background: transparent;
        border: none;
        border-radius: 6px;
        font-size: 18px;
        cursor: pointer;
        transition: all 0.15s ease;

        &:hover {
            background: #f1f5f9;
        }

        &--active {
            background: #dbeafe;
        }
    }

    &__grid-container {
        flex: 1;
        overflow-y: auto;
        padding: 8px;

        &::-webkit-scrollbar {
            width: 6px;
        }

        &::-webkit-scrollbar-thumb {
            background-color: #cbd5e1;
            border-radius: 3px;
        }

        &::-webkit-scrollbar-track {
            background-color: transparent;
        }
    }

    &__grid {
        display: grid;
        grid-template-columns: repeat(8, 1fr);
        gap: 2px;
    }

    &__emoji {
        display: flex;
        align-items: center;
        justify-content: center;
        width: 100%;
        aspect-ratio: 1;
        padding: 0;
        background: transparent;
        border: none;
        border-radius: 6px;
        font-size: 22px;
        cursor: pointer;
        transition: all 0.1s ease;

        &:hover {
            background: #f1f5f9;
            transform: scale(1.15);
        }

        &:active {
            transform: scale(1);
        }
    }

    &__empty {
        grid-column: 1 / -1;
        padding: 20px;
        text-align: center;
        color: #94a3b8;
        font-size: 13px;
    }
}

@keyframes emojiPickerFadeIn {
    from {
        opacity: 0;
        transform: translateY(8px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}
</style>

