<template>
    <div class="kiwi-autocomplete">

        <div v-for="(item, item_idx) in filteredAndLimitedItems"
            :class="{
                'kiwi-autocomplete-item': true,
                'kiwi-autocomplete-item--selected': item.idx === selected_idx}
            "
            @click="selected_idx = item.idx; selectCurrentItem()"
        >
            <template v-if="item.type === 'user'">
                <span class="kiwi-autocomplete-item-value">{{item.text}}</span>
                <span class="u-link kiwi-autocomplete-item-action" @click.stop="openQuery(item.text)">{{$t('send_message')}}</span>
            </template>
            <template v-else-if="item.type === 'command'">
                <span class="kiwi-autocomplete-item-value">{{item.text}}</span>
                <span class="u-link kiwi-autocomplete-item-description">{{item.description}}</span>
            </template>
            <template v-else>
                <span class="kiwi-autocomplete-item-value">{{item.text}}</span>
            </template>
        </div>
    </div>
</template>

<script>

import _ from 'lodash';
import state from '@/libs/state';

export default {
    data: function data() {
        return {
            items_: [
                { text: 'anick1', type: 'user' },
                { text: 'anick2', type: 'user' },
                { text: 'bnick3', type: 'user' },
                { text: 'cnick4' },
                { text: 'dnick5' },
            ],
            selected_idx: 0,
        };
    },
    props: ['filter', 'buffer', 'items'],
    computed: {
        filteredItems: function filteredItems() {
            let filterVal = (this.filter || '').toLowerCase();

            return _(this.items).filter(item => {
                let s = false;
                if (item.text.toLowerCase().indexOf(filterVal) === 0) {
                    s = true;
                }

                return s;
            })
                .sort((a, b) => a.text.localeCompare(b.text))
                .value();
        },
        filteredAndLimitedItems: function filteredAndLimitedItems() {
            return this.filteredItems.filter((item, itemIdx, items) => {
                let numItems = items.length - 1;
                let idxFrom = this.selected_idx - 3;
                let idxTo = this.selected_idx + 3;
                let isInRange = false;

                // Adjust the number of items before and after the selected item
                // when we reach either end of the list.
                // If we don't do this then this:
                // * Item 1
                // * item 2
                // * Item 3
                // * Item 4
                // * Item 5 < selected
                // Can turn into this:
                // * Item 4
                // * Item 5 < selected
                if (idxFrom < 0) {
                    idxTo += -idxFrom;
                    idxFrom = 0;
                } else if (idxTo > numItems) {
                    idxFrom -= (idxTo - numItems);
                    idxTo = numItems;
                }

                if (itemIdx >= idxFrom && itemIdx <= idxTo) {
                    isInRange = true;
                }

                // Keep track of the pre-limited index for item selection
                item.idx = itemIdx;

                return isInRange;
            });
        },
        selectedItem: function selectedItem() {
            let item = this.filteredItems[this.selected_idx];
            return item || null;
        },
    },
    methods: {
        handleOnKeyDown: function handleOnKeyDown(event) {
            let handled = false;

            let cancelKeyCodes = [
                13, // return
                188, // comma
                190, // period
            ];

            if (cancelKeyCodes.indexOf(event.keyCode) > -1) {
                // If no item is selected (ie. on an empty list), leave the return key
                // to do its default action as if the autocomplete box isn't active.
                if (!this.selectedItem) {
                    this.cancel();
                } else {
                    this.selectCurrentItem();
                    if (event.keyCode === 13) {
                        event.preventDefault();
                    }
                    handled = true;
                }
            } else if (event.keyCode === 38 || (event.keyCode === 9 && event.shiftKey)) {
                // Up or tab + shift
                if (this.selected_idx > 0) {
                    this.selected_idx--;
                } else {
                    // Wrap around to the end
                    this.selected_idx = this.filteredItems.length - 1;
                }

                event.preventDefault();
                handled = true;
            } else if ((event.keyCode === 40 && !event.altKey) || event.keyCode === 9) {
                // Down or tab
                if (this.selected_idx < this.filteredItems.length - 1) {
                    this.selected_idx++;
                } else {
                    // Wrap around to the start
                    this.selected_idx = 0;
                }

                event.preventDefault();
                handled = true;
            }

            return handled;
        },
        openQuery: function openQuery(nick) {
            let buffer = state.addBuffer(this.buffer.networkid, nick);
            state.setActiveBuffer(buffer.networkid, buffer.name);
            this.cancel();
        },
        tempCurrentItem: function tempCurrentItem() {
            let item = this.selectedItem;
            if (!item) {
                return;
            }
            this.$emit('temp', item.value || item.text, item);
        },
        selectCurrentItem: function selectCurrentItem() {
            let item = this.selectedItem;
            let value = '';

            if (item) {
                value = item.value || item.text;
            }

            this.$emit('selected', value, item);
        },
        cancel: function cancel() {
            this.$emit('cancel');
        },
    },
    mounted: function mounted() {
        this.tempCurrentItem();
    },
    watch: {
        selected_idx: function watchSelectedIdx() {
            // nextTick() as the DOM hasn't updated yet
            this.$nextTick(() => {
                let el = this.$el.querySelector('.kiwi-autocomplete-item--selected');
                if (!el) {
                    return;
                }

                this.$el.scrollTop = el.offsetTop - (el.getBoundingClientRect().height * 2);
            });

            this.tempCurrentItem();
        },
        filter: function watchFilter() {
            let numItems = this.filteredAndLimitedItems.length - 1;
            if (this.selected_idx > numItems) {
                this.selected_idx = numItems;
            }
        },
    },
};
</script>

<style>

.kiwi-autocomplete {
    box-sizing: border-box;
    overflow-y: auto;
    position: absolute;
    bottom: 100%;
    right: 0;
    left: 0;
    z-index: 1;
    max-height: 300px;
}

.kiwi-autocomplete-item {
    padding: 5px 2em;
}

.kiwi-autocomplete-item-value {
    font-weight: bold;
}

.kiwi-autocomplete-item-action {
    float: right;
    font-size: 0.9em;
}

</style>
