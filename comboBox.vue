<template>
    <div class="combo js-select" :class="{ open: open }">
        <div
            id="combo1"
            ref="combobox"
            aria-controls="listbox1"
            aria-expanded="false"
            aria-haspopup="listbox"
            aria-labelledby="combo1-label"
            class="combo-input"
            role="combobox"
            tabindex="0"
            @click="onComboClick"
            @blur="onComboBlur"
            @keydown="onComboKeyDown"
        >
            {{ selecetedItem }}
        </div>
        <div id="listbox1" ref="listbox" class="combo-menu" role="listbox" aria-labelledby="combo1-label" tabindex="-1">
            <div
                v-for="(option, index) in options"
                :id="`option-${index}`"
                ref="optionList"
                :class="index === 0 ? 'combo-option option-current' : 'combo-option'"
                role="option"
                :aria-selected="`${index === 0}`"
                @click="onOptionClick(index)"
                @mousedown="onOptionMouseDown"
            >
                {{ option.label }}
            </div>
        </div>
    </div>
</template>
<script>
    export default {
        props: {
            options: []
        },

        data() {
            return {
                selectedItem: 'not selected',
                SelectActions: {
                    Close: 0,
                    CloseSelect: 1,
                    First: 2,
                    Last: 3,
                    Next: 4,
                    Open: 5,
                    PageDown: 6,
                    PageUp: 7,
                    Previous: 8,
                    Select: 9,
                    Type: 10
                },
                comboEl: this.$refs.combobox,
                listboxEl: this.$refs.listbox,
                activeIndex: 0,
                open: false,
                searchString: '',
                searchTimeout: null,
                selecetedItem: 'not Selected'
            };
        },
        mounted() {
            this.init();
        },
        methods: {
            /*
             *   This content is licensed according to the W3C Software License at
             *   https://www.w3.org/Consortium/Legal/2015/copyright-software-and-document
             */

            // filter an array of options against an input string
            // returns an array of options that begin with the filter string, case-independent
            filterOptions(options = [], filter, exclude = []) {
                return options.filter((option) => {
                    const matches = option.toLowerCase().indexOf(filter.toLowerCase()) === 0;
                    return matches && !exclude.includes(option);
                });
            },

            // map a key press to an action
            getActionFromKey(event, menuOpen) {
                const { key, altKey, ctrlKey, metaKey } = event;
                const openKeys = ['ArrowDown', 'ArrowUp', 'Enter', ' ']; // all keys that will do the default open action
                // handle opening when closed
                if (!menuOpen && openKeys.includes(key)) {
                    return this.SelectActions.Open;
                }

                // home and end move the selected option when open or closed
                if (key === 'Home') {
                    return this.SelectActions.First;
                }
                if (key === 'End') {
                    return this.SelectActions.Last;
                }

                // handle typing characters when open or closed
                if (
                    key === 'Backspace' ||
                    key === 'Clear' ||
                    (key.length === 1 && key !== ' ' && !altKey && !ctrlKey && !metaKey)
                ) {
                    return this.SelectActions.Type;
                }

                // handle keys when open
                if (menuOpen) {
                    if (key === 'ArrowUp' && altKey) {
                        return this.SelectActions.CloseSelect;
                    } else if (key === 'ArrowDown' && !altKey) {
                        return this.SelectActions.Next;
                    } else if (key === 'ArrowUp') {
                        return this.SelectActions.Previous;
                    } else if (key === 'PageUp') {
                        return this.SelectActions.PageUp;
                    } else if (key === 'PageDown') {
                        return this.SelectActions.PageDown;
                    } else if (key === 'Escape') {
                        return this.SelectActions.Close;
                    } else if (key === 'Enter' || key === ' ') {
                        return this.SelectActions.CloseSelect;
                    }
                }
            },

            // return the index of an option from an array of options, based on a search string
            // if the filter is multiple iterations of the same letter (e.g "aaa"), then cycle through first-letter matches
            getIndexByLetter(options, filter, startIndex = 0) {
                const orderedOptions = [...options.slice(startIndex), ...options.slice(0, startIndex)];
                const firstMatch = this.filterOptions(orderedOptions, filter)[0];
                const allSameLetter = (array) => array.every((letter) => letter === array[0]);

                // first check if there is an exact match for the typed string
                if (firstMatch) {
                    return options.indexOf(firstMatch);
                }

                // if the same letter is being repeated, cycle through first-letter matches
                else if (allSameLetter(filter.split(''))) {
                    const matches = this.filterOptions(orderedOptions, filter[0]);
                    return options.indexOf(matches[0]);
                }

                // if no matches, return -1
                else {
                    return -1;
                }
            },

            // get an updated option index after performing an action
            getUpdatedIndex(currentIndex, maxIndex, action) {
                const pageSize = 10; // used for pageup/pagedown

                switch (action) {
                    case this.SelectActions.First:
                        return 0;
                    case this.SelectActions.Last:
                        return maxIndex;
                    case this.SelectActions.Previous:
                        return Math.max(0, currentIndex - 1);
                    case this.SelectActions.Next:
                        return Math.min(maxIndex, currentIndex + 1);
                    case this.SelectActions.PageUp:
                        return Math.max(0, currentIndex - pageSize);
                    case this.SelectActions.PageDown:
                        return Math.min(maxIndex, currentIndex + pageSize);
                    default:
                        return currentIndex;
                }
            },

            // check if element is visible in browser view port
            isElementInView(element) {
                const bounding = element.getBoundingClientRect();

                return (
                    bounding.top >= 0 &&
                    bounding.left >= 0 &&
                    bounding.bottom <= (window.innerHeight || document.documentElement.clientHeight) &&
                    bounding.right <= (window.innerWidth || document.documentElement.clientWidth)
                );
            },

            // check if an element is currently scrollable
            isScrollable(element) {
                return element && element.clientHeight < element.scrollHeight;
            },

            // ensure a given child element is within the parent's visible scroll area
            // if the child is not visible, scroll the parent
            maintainScrollVisibility(activeElement, scrollParent) {
                const { offsetHeight, offsetTop } = activeElement;
                const { offsetHeight: parentOffsetHeight, scrollTop } = scrollParent;

                const isAbove = offsetTop < scrollTop;
                const isBelow = offsetTop + offsetHeight > scrollTop + parentOffsetHeight;

                if (isAbove) {
                    scrollParent.scrollTo(0, offsetTop);
                } else if (isBelow) {
                    scrollParent.scrollTo(0, offsetTop - parentOffsetHeight + offsetHeight);
                }
            },

            init() {
                // element refs
                this.comboEl = this.$refs.combobox;
                this.listboxEl = this.$refs.listbox;

                // data
                this.options = this.options;

                // state
                this.activeIndex = 0;
                this.open = false;
                this.searchString = '';
                this.searchTimeout = null;

                // // init
                // if (this.comboEl && this.listboxEl) {
                //     this.init();
                // }
                // select first option by default
                this.selecetedItem = this.options[0].label;
            },

            getSearchString(char) {
                // reset typing timeout and start new timeout
                // this allows us to make multiple-letter matches, like a native select
                if (typeof this.searchTimeout === 'number') {
                    window.clearTimeout(this.searchTimeout);
                }

                this.searchTimeout = window.setTimeout(() => {
                    this.searchString = '';
                }, 500);

                // add most recent letter to saved search string
                this.searchString += char;
                return this.searchString;
            },

            onComboBlur() {
                // do not do blur action if ignoreBlur flag has been set
                if (this.ignoreBlur) {
                    this.ignoreBlur = false;
                    return;
                }

                // select current option and close
                if (this.open) {
                    this.selectOption(this.activeIndex);
                    this.updateMenuState(false, false);
                }
            },

            onComboClick() {
                this.updateMenuState(!this.open, false);
            },

            onComboKeyDown(event) {
                const { key } = event;
                const max = this.options.length - 1;

                const action = this.getActionFromKey(event, this.open);

                switch (action) {
                    case this.SelectActions.Last:
                    case this.SelectActions.First:
                        this.updateMenuState(true);
                    // intentional fallthrough
                    case this.SelectActions.Next:
                    case this.SelectActions.Previous:
                    case this.SelectActions.PageUp:
                    case this.SelectActions.PageDown:
                        event.preventDefault();
                        return this.onOptionChange(this.getUpdatedIndex(this.activeIndex, max, action));
                    case this.SelectActions.CloseSelect:
                        event.preventDefault();
                        this.selectOption(this.activeIndex);
                    // intentional fallthrough
                    case this.SelectActions.Close:
                        event.preventDefault();
                        return this.updateMenuState(false);
                    case this.SelectActions.Type:
                        return this.onComboType(key);
                    case this.SelectActions.Open:
                        event.preventDefault();
                        return this.updateMenuState(true);
                }
            },

            onComboType(letter) {
                // open the listbox if it is closed
                this.updateMenuState(true);

                // find the index of the first matching option
                const searchString = this.getSearchString(letter);
                const searchIndex = this.getIndexByLetter(this.options, searchString, this.activeIndex + 1);

                // if a match was found, go to it
                if (searchIndex >= 0) {
                    this.onOptionChange(searchIndex);
                }
                // if no matches, clear the timeout and search string
                else {
                    window.clearTimeout(this.searchTimeout);
                    this.searchString = '';
                }
            },

            onOptionChange(index) {
                // update state
                this.activeIndex = index;

                // update aria-activedescendant
                this.comboEl.setAttribute('aria-activedescendant', `option-${index}`);

                // update active option styles
                const options = this.$refs.optionList;
                [...options].forEach((optionEl) => {
                    optionEl.classList.remove('option-current');
                });
                options[index].classList.add('option-current');

                // ensure the new option is in view
                if (this.isScrollable(this.listboxEl)) {
                    this.maintainScrollVisibility(options[index], this.listboxEl);
                }

                // ensure the new option is visible on screen
                // ensure the new option is in view
                if (!this.isElementInView(options[index])) {
                    options[index].scrollIntoView({ behavior: 'smooth', block: 'nearest' });
                }
            },

            onOptionClick(index) {
                this.onOptionChange(index);
                this.selectOption(index);
                this.updateMenuState(false);
            },

            onOptionMouseDown() {
                // Clicking an option will cause a blur event,
                // but we don't want to perform the default keyboard blur action
                this.ignoreBlur = true;
            },

            selectOption(index) {
                // update state
                this.activeIndex = index;
                // update displayed value
                const selected = this.options[index];
                this.selecetedItem = selected.label;
                // update aria-selected
                const options = this.$refs.optionList;
                if (options.length > 0) {
                    [...options].forEach((optionEl) => {
                        optionEl.setAttribute('aria-selected', 'false');
                    });
                    options[index].setAttribute('aria-selected', 'true');
                }
                this.$emit('selected', selected);
            },

            updateMenuState(open, callFocus = true) {
                if (this.open === open) {
                    return;
                }

                // update state
                this.open = open;

                // update aria-expanded and styles
                this.comboEl.setAttribute('aria-expanded', `${open}`);
                open ? this.comboEl.classList.add('open') : this.comboEl.classList.remove('open');

                // update activedescendant
                const activeID = open ? `option-${this.activeIndex}` : '';
                this.comboEl.setAttribute('aria-activedescendant', activeID);

                if (activeID === '' && !this.isElementInView(this.comboEl)) {
                    this.comboEl.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
                }

                // move focus back to the combobox, if needed
                callFocus && this.comboEl.focus();
            }
        }
    };
</script>
<style scoped>
    .combo *,
    .combo *::before,
    .combo *::after {
        box-sizing: border-box;
    }

    .combo {
        display: block;
        margin-bottom: 1.5em;
        max-width: 400px;
        position: relative;
    }

    .combo::after {
        border-bottom: 2px solid rgb(0 0 0 / 75%);
        border-right: 2px solid rgb(0 0 0 / 75%);
        content: '';
        display: block;
        height: 12px;
        pointer-events: none;
        position: absolute;
        right: 16px;
        top: 50%;
        transform: translate(0, -65%) rotate(45deg);
        width: 12px;
    }

    .combo-input {
        background-color: #f5f5f5;
        border: 2px solid rgb(0 0 0 / 75%);
        border-radius: 4px;
        display: block;
        font-size: 1em;
        min-height: calc(1.4em + 26px);
        padding: 12px 16px 14px;
        text-align: left;
        width: 100%;
    }

    .open .combo-input {
        border-radius: 4px 4px 0 0;
    }

    .combo-input:focus {
        border-color: black;
        box-shadow: 0 0 4px 2px black;
        outline: 4px solid transparent;
    }

    .combo-label {
        display: block;
        font-size: 20px;
        font-weight: 100;
        margin-bottom: 0.25em;
    }

    .combo-menu {
        background-color: #f5f5f5;
        border: 1px solid rgb(0 0 0 / 75%);
        border-radius: 0 0 4px 4px;
        display: none;
        max-height: 300px;
        overflow-y: scroll;
        left: 0;
        position: absolute;
        top: 100%;
        width: 100%;
        z-index: 100;
    }

    .open .combo-menu {
        display: block;
    }

    .combo-option {
        padding: 10px 12px 12px;
    }

    .combo-option:hover {
        background-color: rgb(0 0 0 / 10%);
    }

    .combo-option.option-current {
        outline: 3px solid #0067b8;
        outline-offset: -3px;
    }

    .combo-option[aria-selected='true'] {
        padding-right: 30px;
        position: relative;
    }

    .combo-option[aria-selected='true']::after {
        border-bottom: 2px solid #000;
        border-right: 2px solid #000;
        content: '';
        height: 16px;
        position: absolute;
        right: 15px;
        top: 50%;
        transform: translate(0, -50%) rotate(45deg);
        width: 8px;
    }
</style>