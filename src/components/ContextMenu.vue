<template>
    <teleport to="body">
        <div 
            v-if="visible" 
            ref="menuRef"
            class="ww-context-menu"
            :style="positionStyle"
            @click.stop
        >
            <div 
                v-for="item in items" 
                :key="item.id"
                class="ww-context-menu__item"
                :class="{ 
                    'ww-context-menu__item--disabled': item.disabled,
                    'ww-context-menu__item--danger': item.danger 
                }"
                @click="handleItemClick(item)"
            >
                <span 
                    v-if="item.icon" 
                    class="ww-context-menu__icon"
                    v-html="item.icon"
                ></span>
                <span class="ww-context-menu__label">{{ item.label }}</span>
            </div>
            <div v-if="items.length === 0" class="ww-context-menu__empty">
                Aucune action disponible
            </div>
        </div>
        <!-- Backdrop to catch outside clicks -->
        <div 
            v-if="visible" 
            class="ww-context-menu__backdrop"
            @click="close"
            @contextmenu.prevent="close"
        ></div>
    </teleport>
</template>

<script>
import { ref, computed, watch, onMounted, onUnmounted } from 'vue';

export default {
    name: 'ContextMenu',
    props: {
        visible: {
            type: Boolean,
            default: false,
        },
        x: {
            type: Number,
            default: 0,
        },
        y: {
            type: Number,
            default: 0,
        },
        items: {
            type: Array,
            default: () => [],
            // Each item: { id: string, label: string, icon?: string, disabled?: boolean, danger?: boolean }
        },
    },
    emits: ['select', 'close'],
    setup(props, { emit }) {
        const menuRef = ref(null);

        const positionStyle = computed(() => {
            // Basic positioning - will be adjusted after mount if needed
            let left = props.x;
            let top = props.y;

            // Get viewport dimensions
            const viewportWidth = window.innerWidth;
            const viewportHeight = window.innerHeight;

            // Estimate menu dimensions (will refine after mount)
            const menuWidth = 180;
            const menuHeight = props.items.length * 40 + 16; // Rough estimate

            // Adjust if menu would overflow right edge
            if (left + menuWidth > viewportWidth - 10) {
                left = viewportWidth - menuWidth - 10;
            }

            // Adjust if menu would overflow bottom edge
            if (top + menuHeight > viewportHeight - 10) {
                top = viewportHeight - menuHeight - 10;
            }

            // Ensure menu doesn't go off left or top
            left = Math.max(10, left);
            top = Math.max(10, top);

            return {
                left: `${left}px`,
                top: `${top}px`,
            };
        });

        const handleItemClick = (item) => {
            if (item.disabled) return;
            emit('select', item);
            close();
        };

        const close = () => {
            emit('close');
        };

        const handleKeyDown = (event) => {
            if (event.key === 'Escape' && props.visible) {
                close();
            }
        };

        onMounted(() => {
            document.addEventListener('keydown', handleKeyDown);
        });

        onUnmounted(() => {
            document.removeEventListener('keydown', handleKeyDown);
        });

        return {
            menuRef,
            positionStyle,
            handleItemClick,
            close,
        };
    },
};
</script>

<style lang="scss" scoped>
.ww-context-menu {
    position: fixed;
    z-index: 10000;
    min-width: 160px;
    max-width: 280px;
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 10px;
    box-shadow: 0 10px 40px rgba(0, 0, 0, 0.15), 0 2px 10px rgba(0, 0, 0, 0.1);
    padding: 6px;
    animation: contextMenuFadeIn 0.15s ease-out;

    &__backdrop {
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        z-index: 9999;
        background: transparent;
    }

    &__item {
        display: flex;
        align-items: center;
        gap: 10px;
        padding: 10px 14px;
        border-radius: 6px;
        cursor: pointer;
        font-size: 14px;
        font-weight: 500;
        color: #334155;
        transition: all 0.15s ease;
        user-select: none;

        &:hover:not(.ww-context-menu__item--disabled) {
            background-color: #f1f5f9;
        }

        &:active:not(.ww-context-menu__item--disabled) {
            background-color: #e2e8f0;
            transform: scale(0.98);
        }

        &--disabled {
            opacity: 0.4;
            cursor: not-allowed;
        }

        &--danger {
            color: #ef4444;

            &:hover:not(.ww-context-menu__item--disabled) {
                background-color: #fef2f2;
            }
        }
    }

    &__icon {
        display: flex;
        align-items: center;
        justify-content: center;
        width: 18px;
        height: 18px;
        flex-shrink: 0;

        :deep(svg) {
            width: 100%;
            height: 100%;
        }
    }

    &__label {
        flex: 1;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
    }

    &__empty {
        padding: 12px 14px;
        font-size: 13px;
        color: #94a3b8;
        text-align: center;
        font-style: italic;
    }
}

@keyframes contextMenuFadeIn {
    from {
        opacity: 0;
        transform: scale(0.95) translateY(-5px);
    }
    to {
        opacity: 1;
        transform: scale(1) translateY(0);
    }
}
</style>

