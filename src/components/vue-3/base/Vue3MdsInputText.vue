<template>
  <div class="v-mds-input-text" :class="{ compact, invalid, warning }" ref="root">
    <div class="flex">
      <label v-if="label" :for="name">
        <div class="shaker">{{ label }}</div>
        <span v-if="required" class="required-mark">(Required)</span>
      </label>
      <Vue3MdsTooltip v-if="tooltip" :tip="tooltip" />
    </div>
    <div class="flex">
      <div v-if="prefix || $slots.prefix" class="prefix">
        <slot name="prefix">
          {{ prefix }}
        </slot>
      </div>
      <div class="v-autocomplete-container">
        <input
          v-if="!multiline"
          :type="type"
          :class="{ hasPrefix: prefix || $slots.prefix, hasSuffix: suffix || $slots.suffix, open: open && autocompleteResults.length, 'has-eye-icon': hasEyeIcon }"
          :name="name"
          :disabled="disabled"
          :value="displayText"
          :maxlength="maxLength"
          v-bind="$attrs"
          @focus="open = true"
          @keydown.enter.stop.prevent="didEnter($event.target.value)"
          @input="didInput($event.target.value)"
          @blur="didBlur"
          :style="inputStyles"
        />
        <textarea
          v-else
          :class="{ hasPrefix: prefix || $slots.prefix, hasSuffix: suffix || $slots.suffix, open: open && autocompleteResults.length }"
          :name="name"
          :disabled="disabled"
          :value="displayText"
          v-bind="$attrs"
          :maxlength="maxLength"
          @focus="open = true"
          @keydown="open = true"
          @keydown.enter="onInputEnter($event)"
          @input="didInput($event.target.value)"
          @blur="didBlur"
          :style="inputStyles"
        />
        <Transition :css="false" @enter="onAutocompleteEnter" @leave="onLeave">
          <div class="v-autocomplete-results" v-if="open && autocompleteResults.length">
            <div
              v-for="result in autocompleteResults"
              :key="result[valueKey]"
              class="result"
              @click="didSelectAutocompleteResult(result)"
            >
              <slot v-bind="result">
                {{ result[textKey] }}
              </slot>
            </div>
          </div>
        </Transition>
        <iconify-icon
          icon="mdi:close"
          :height="compact ? 12 : 16"
          class="clear"
          :class="{ 'v-hidden': !clearable || !modelValue || disabled }"
          @click="didClear"
        />
        <button
          v-if="showPassword"
          class="show-password-btn"
          aria-label="Show password"
          title="Show Password"
        >
          <!-- <img
            src="@icons/show-password.svg"
            width="24"
            height="24"
            alt="Show Password"
            class="show-password-icon"
            @click="showPassword = false; $emit('show-password-toggle', showPassword)"
          /> -->
        </button>
        <button
          v-else
          class="show-password-btn"
          aria-label="Hide password"
          title="Hide Password"
        >
          <!-- <img
            src="@icons/hide-password.svg"
            width="24"
            height="24"
            alt="Hide Password"
            class="show-password-icon"
            @click="showPassword = true; $emit('show-password-toggle', showPassword)"
          /> -->
        </button>
      </div>
      <div v-if="suffix || $slots.suffix" class="suffix">
        <slot name="suffix">
          {{ suffix }}
        </slot>
      </div>
      <div v-if="hint || $slots.hint" class="hint">
        <slot name="hint">
          {{ hint }}
        </slot>
      </div>
    </div>
    <div class="error-line" :class="{visible: errorMessage && invalid}">
      {{ errorMessage }}
    </div>
  </div>
</template>

<script>
export default {
  name: 'Vue3MdsInputText',
  inheritAttrs: false,
  emits: ['update:modelValue', 'autocomplete-result-selected', 'enter', 'blur', 'show-password-toggle'],
  props: {
    autocompleteResults: { type: Array, default() { return [] } },
    clearable: Boolean,
    clearDisplayOnSelection: Boolean,
    compact: Boolean,
    disabled: Boolean,
    invalid: Boolean,
    label: String,
    multiline: Boolean,
    name: String,
    hint: String,
    errorMessage: String,
    modelValue: String,
    prefix: [String, Number],
    returnObject: Boolean,
    suffix: [String, Number],
    textKey: { type: String, default: 'text' },
    tooltip: [String, Object],
    type: {type: String, default: 'text'},
    valueKey: { type: String, default: 'value' },
    warning: Boolean,
    maxLength: { type: Number, default: null },
    inputStyles: Object,
    allowEnter: { type: Boolean, default: false },
    hasEyeIcon: { type: Boolean, default: false },
    required: { type: Boolean, default: false }
  },
  data() {
    return {
      open: false,
      autocompleteDisplayText: '',
      showPassword: true
    }
  },
  computed: {
    displayText() {
      return this.autocompleteDisplayText ? this.autocompleteDisplayText : this.modelValue
    },
    showAutocompleteResults() {
      return this.open && this.autocompleteResults?.length > 0
    }
  },
  methods: {
    didEnter(val) {
      this.$emit('enter', val)
    },
    didInput(val) {
      this.autocompleteDisplayText = ''
      this.$emit('update:modelValue', val)
    },
    didBlur() {
      this.$emit('blur', this.modelValue)
    },
    didClear() {
      this.autocompleteDisplayText = ''
      this.$emit('update:modelValue', '')
    },
    didSelectAutocompleteResult(result) {
      if (this.clearDisplayOnSelection) {
        this.autocompleteDisplayText = ''
      } else if (this.textKey) {
        this.autocompleteDisplayText = result[this.textKey]
      }
      if (this.returnObject) {
        this.$emit('autocomplete-result-selected', result)
      } else {
        this.$emit('autocomplete-result-selected', result[this.valueKey])
      }
    },
    // TRANSITIONS
    onAutocompleteEnter(el, done) {
      let finalHeight = el.getBoundingClientRect().height
      el.animate([{ height: '0px' }, { height: finalHeight + 'px' }], 150).onfinish = done
    },
    onLeave(el, done) {
      let initialHeight = el.getBoundingClientRect().height
      el.animate([{ height: initialHeight + 'px' }, { height: '0px' }], 150).onfinish = done
    },
    // KEYBOARD HANDLING
    onInputEnter(evt) {
      if (this.multiline && this.allowEnter) return
      evt.stopPropagation()
      evt.preventDefault()
      const val = evt && evt.target ? evt.target.value : this.modelValue
      this.didEnter(val)
    },
    closeMenuOnOutsideClick(e) {
      const root = this.$refs.root
      if (!root) {
        window.removeEventListener('click', this.closeMenuOnOutsideClick)
        return
      }

      if (!root.contains(e.target)) this.open = false
    }
  },
  mounted() {
    window.addEventListener('click', this.closeMenuOnOutsideClick)
  },
  beforeUnmount() {
    window.removeEventListener('click', this.closeMenuOnOutsideClick)
  }
}
</script>

<style lang="scss" scoped>
// @use '@styles/tokens/_animations.scss' as animations;
@use '../_colours.scss' as colours;

.v-mds-input-text {
  position: relative;
  .flex {
    display: flex;
    justify-content: space-between;
    :deep(.v-tooltip) {
      margin-right: 2px;
    }
  }
  label {
    display: flex;
    align-items: center;
    gap: 5px;
    color: colours.$inputLabel;
    font-family: 'Libre Franklin', sans-serif;
    font-size: 16px;
    font-weight: 400;
    line-height: 140%;
    margin-bottom: 5px;

    .required-mark {
      font-size: 14px;
    }
  }
  .v-autocomplete-container {
    position: relative;
    flex: 1;
    .v-autocomplete-results {
      position: absolute;
      top: 100%;
      left: 0px;
      right: 0px;
      background: colours.$inputBackground;
      border: 1px solid #80bdff;
      border-top: none;
      border-radius: 0px 0px colours.$inputBorderRadius colours.$inputBorderRadius;
      box-shadow: 0px 0px 3px rgba(#000, 0.24);
      overflow: hidden;
      z-index: 1000;
      max-height: calc(27px * 12.8);
      overflow: auto;
      .result {
        cursor: pointer;
        display: flex;
        align-items: baseline;
        font-size: 13px;
        background: colours.$inputBackground;
        padding: 4px 8px;
        user-select: none;
        transition: 0.15s;
        &:hover {
          color: #fff;
          background: #188ae2;
        }
      }
    }
    .show-password-btn {
      display: none;
    }
    &:has(.has-eye-icon) {
      .show-password-btn {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        position: absolute;
        top: 50%;
        transform: translateY(-50%);
        right: 10px;
        cursor: pointer;
        width: 24px;
        height: 24px;
        background: transparent;
        border: 1px solid transparent;
        &:focus {
          border: 1px solid colours.$messagingFocus;
        }
      }
      .show-password-icon {
        max-width: unset;
      }
    }
  }
  input, textarea {
    width: 100%;
    font-size: 400;
    color: colours.$inputText;
    padding: 5px 10px;
    border: colours.$inputBorder;
    border-radius: colours.$inputBorderRadius;
    transition: 0.15s;
    flex: 1;
    font-family: 'Libre Franklin', sans-serif;
    &[type="password"] {
      padding-right: 40px;
    }
    &.hasPrefix {
      border-top-left-radius: 0px;
      border-bottom-left-radius: 0px;
    }
    &.hasSuffix {
      border-top-right-radius: 0px;
      border-bottom-right-radius: 0px;
    }
    &:focus {
      color: colours.$inputTextFocus;
      outline: none;
      border: colours.$inputBorderFocus;
    }
    &:disabled {
      background: colours.$inputBackgroundDisabled;
    }
    &.open {
      border-bottom-left-radius: 0px;
      border-bottom-right-radius: 0px;
    }
    &.open {
      border-bottom-left-radius: 0px;
      border-bottom-right-radius: 0px;
    }
  }
  input {
    min-height: 40px;
  }
  textarea {
    display: block;
    height: 100%;
    min-height: 80px;
    resize: vertical;
    overflow: auto;
  }
  .prefix,
  .suffix {
    color: colours.$inputPrefixText;
    background: colours.$inputPrefixBackground;
    padding: 8px 16px;
    border: colours.$inputPrefixBorder;
    &.prefix {
      border-right: none;
      border-radius: colours.$inputBorderRadius 0px 0px colours.$inputBorderRadius;
    }
    &.suffix {
      border-left: none;
      border-radius: 0px colours.$inputBorderRadius colours.$inputBorderRadius 0px;
    }
  }
  .hint {
    color: #6C757D;
    margin-top: 5px;
    font-size: 12px;
  }
  ::placeholder {
    color: colours.$inputPlaceholder;
  }
  .clear {
    cursor: pointer;
    position: absolute;
    display: grid;
    place-items: center;
    top: 9px;
    right: 16px;
    width: 20px;
    height: 20px;
    color: colours.$inputIconText;
    background: colours.$inputIconBackground;
    border-radius: 50%;
    transition: 0.2s;
    &.v-hidden {
      transform: scale(0);
    }
  }
  &.invalid {
    label {
      .shaker {
        // @include animations.invalidShake();
      }
    }
    input, textarea {
      border: colours.$inputBorderInvalid;
    }
    .clear {
      color: colours.$inputIconTextInvalid;
      background: colours.$inputIconBackgroundInvalid;
    }
  }
  &.warning input {
    border: colours.$inputBorderWarning;
  }
  &.compact {
    label {
      font-size: 12px;
      line-height: 18px;
      margin-bottom: 0px;
    }
    input, textarea {
      height: 20px;
      min-height: 20px;
      font-size: 12px;
      padding: 1px 8px;
    }
    .clear {
      height: 14px;
      width: 14px;
      top: 5px;
      right: 8px;
    }
    .v-autocomplete-results {
      .result {
        font-size: 12px;
        padding: 1px 8px;
      }
    }
  }
  .error-line {
    color: colours.$inputErrorColour;
    font-family: 'Libre Franklin', sans-serif;
    font-size: 14px;
    font-weight: 400;
    margin-top: 5px;
    opacity: 0;
    min-height: 20px;
    transition: all ease 0.2s;
    &.visible {
      opacity: 1;
    }
  }
}
</style>