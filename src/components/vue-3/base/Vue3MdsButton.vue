<template>
  <button
    class="v-mds-button"
    :class="classObject"
    :type="behavior"
    @click="clicked"
    :disabled="disabled"
  >
    <div class="inner" :style="{ opacity: loading ? 0 : 1 }">
      <slot>
        <img
          v-if="leftIcon"
          class="icon"
          :src="leftIcon"
          :height="iconSize"
        />
        <span v-if="label" v-html="label" class="label" />
        <img
          v-if="rightIcon"
          class="icon"
          :src="rightIcon"
          :height="iconSize"
        />
      </slot>
    </div>
  </button>
</template>

<script>
export default {
  name: 'Vue3MdsButton',
  props: {
    block: Boolean,
    disabled: Boolean,
    href: String,
    leftIcon: String,
    rightIcon: String,
    label: String,
    large: Boolean,
    link: Boolean,
    loading: Boolean,
    outline: Boolean,
    preventDefault: Boolean,
    rounded: Boolean,
    small: Boolean,
    target: String,
    toggleState: Boolean,
    inline: Boolean,
    type: {
      type: String,
      default: 'tiertary',
      validator(value) {
        return ['primary', 'secondary', 'tiertary'].includes(
          value,
        )
      },
    },
    behavior: {
      type: String,
      default: 'button',
    },
    modelValue: {
      type: Boolean,
      default: undefined,
    },
    wide: Boolean,
  },
  emits: ['click'],
  computed: {
    classObject() {
      return [
        this.type,
        {
          active: this.toggleState,
          block: this.block,
          large: this.large,
          link: this.link,
          outline: this.outline,
          rounded: this.rounded,
          small: this.small,
          wide: this.wide,
          inline: this.inline,
        },
      ]
    },
    iconSize() {
      return this.small ? 12 : this.large ? 20 : 14
    },
  },
  methods: {
    clicked(e) {
      if (this.loading) return
      if (this.preventDefault) e.preventDefault()
      if (this.disabled) {
        e.preventDefault()
        this.$emit('click-while-disabled')
      } else {
        if (this.href) {
          if (this.target) {
            window.open(this.href, this.target)
          } else {
            window.location.href = this.href
          }
        } else if (this.modelValue != undefined) {
          this.$emit('update:modelValue', !this.modelValue)
        } else {
          this.$emit('click')
        }
      }
    },
  },
}
</script>

<style lang="scss" scoped>
@use '../_colours.scss' as colours;
.v-mds-button {
  display: inline-flex;
  min-height: 30px;
  padding: 5px 10px;
  align-items: center;
  border-radius: 100px;
  border: 1px solid colours.$monoscale-400;
  background: colours.$monoscale-80;
  transition: all ease 0.2s;
  cursor: pointer;
  .inner {
    display: flex;
    align-items: center;
    gap: 5px;
  }
  .label {
    color: colours.$monoscale-800;
    font-family: 'Libre Franklin', sans-serif;
    font-size: 400;
    font-weight: 700;
    line-height: 100%;
  }
  &.small {
    .label {
      font-size: 14px;
    }
  }
  &.primary {
    background: #FDDAB9;
  }
  &.secondary {
    background: #2C1D40;
    .label {
      color: colours.$monoscale-100;
    }
  }
  &.tiertary {
    background: colours.$monoscale-80;
    .label {
      color: colours.$monoscale-800;
    }
  }
  &:hover {
    box-shadow: 0 1px 2px 1px rgba(0, 0, 0, 0.3);
  }
  &:disabled {
    opacity: 0.5;
  }
  &:focus {
    border: 1px solid colours.$messagingFocus;
  }
  &:active {
    box-shadow: 0 1px 2px 0 rgba(0, 0, 0, 0.15) inset;
  }
}
</style>