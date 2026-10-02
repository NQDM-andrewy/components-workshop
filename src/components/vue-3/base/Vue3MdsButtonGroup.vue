<template>
  <div class="v-mds-button-group" :class="{ center, right }">
    <div class="button-group-flex" :class="{ bar, block, column, spaced, spaceAround, spaceBetween }">
      <slot />
    </div>
  </div>
</template>

<script>
export default {
  name: 'Vue3MdsButtonGroup',
  props: {
    bar: Boolean,
    block: Boolean,
    center: Boolean,
    column: Boolean,
    right: Boolean,
    spaced: Boolean,
    spaceAround: Boolean,
    spaceBetween: Boolean,
  }
}
</script>

<style lang="scss" scoped>
.v-mds-button-group {
  display: flex;
  align-items: center;
  &.center {
    justify-content: center;
    width: 100%;
  }
  &.right {
    justify-content: end;
    width: 100%;
  }
  .button-group-flex {
    display: flex;
    align-items: center;
    &:not(.column) {
      &:not(.bar):not(.spaceAround):not(.spaceBetween) {
        >*:not(:last-child) {
          margin-right: 8px;
        }
      }
      &.spaceAround {
        justify-content: space-around;
        width: 100%;
      }
      &.spaceBetween {
        justify-content: space-between;
        width: 100%;
      }
      &.bar {
        >*:first-child {
          border-top-right-radius: 0px;
          border-bottom-right-radius: 0px;
        }
        >*:last-child {
          border-top-left-radius: 0px;
          border-bottom-left-radius: 0px;
        }
        >*:not(:first-child):not(:last-child) {
          border-radius: 0px;
        }
        >*:not(:first-child):not(.active) {
          border-left: none;
        }
        >*:focus {
          z-index: 1;
        }
      }
    }
    &.column {
      flex-direction: column;
      &:not(.bar) {
        >*:not(:last-child) {
          margin-bottom: 4px;
        }
      }
      &.bar {
        >*:first-child {
          border-bottom-left-radius: 0px;
          border-bottom-right-radius: 0px;
        }
        >*:last-child {
          border-top-left-radius: 0px;
          border-top-right-radius: 0px;
        }
        >*:not(:first-child):not(:last-child) {
          border-radius: 0px;
        }
        >*:not(:first-child):not(.active) {
          border-top: none;
        }
        >*:focus {
          z-index: 1;
        }
      }
      &.block {
        width: 100%;
      }
    }
  }
}
</style>