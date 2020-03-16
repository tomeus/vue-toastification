<template>
  <div>
    <div v-for="pos in positions" :key="pos">
      <Transition
        :transition="defaults.transition"
        :transition-duration="defaults.transitionDuration"
        :class="getClasses(pos)"
      >
        <Toast
          v-for="toast in getPositionToasts(pos)"
          :key="toast.id"
          v-bind="toast"
        />
      </Transition>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from "vue";

import Toast from "./VtToast.vue";
import Transition from "./VtTransition.vue";
import events from "../ts/events";
import { EVENTS, POSITION, VT_NAMESPACE } from "../ts/constants";
import PROPS from "../ts/propValidators";
import { PluginOptions, ToastOptions, ToastID } from "../types";
import { removeElement, isUndefined } from "../ts/utils";

export default Vue.extend({
  components: { Toast, Transition },

  props: Object.assign({}, PROPS.CORE_TOAST, PROPS.CONTAINER),

  data() {
    const data: {
      count: number;
      positions: POSITION[];
      toasts: {
        [toastId: number]: ToastOptions;
        [toastId: string]: ToastOptions;
      };
      defaults: PluginOptions;
    } = {
      count: 0,
      positions: Object.values(POSITION),
      toasts: {},
      defaults: {}
    };
    return data;
  },

  computed: {
    toastArray(): ToastOptions[] {
      return Object.values(this.toasts);
    },
    filteredToasts(): ToastOptions[] {
      if (!isUndefined(this.defaults.filterToasts)) {
        return this.defaults.filterToasts(this.toastArray);
      }
      return this.toastArray;
    }
  },

  beforeMount() {
    this.setup(this.container);
    events.$on(EVENTS.ADD, this.addToast);
    events.$on(EVENTS.CLEAR, this.clearToasts);
    events.$on(EVENTS.DISMISS, this.dismissToast);
    events.$on(EVENTS.UPDATE, this.updateToast);
    events.$on(EVENTS.UPDATE_DEFAULTS, this.updateDefaults);
    this.defaults = this.$props;
  },

  methods: {
    setup(container: HTMLElement) {
      removeElement(this.$el);
      container.appendChild(this.$el);
    },
    setToast(props: ToastOptions) {
      if (!isUndefined(props.id)) {
        this.$set(this.toasts, props.id, props);
      }
    },
    addToast(params: ToastOptions) {
      const props = Object.assign({}, this.defaults, params);
      const filterBeforeCreate = isUndefined(this.defaults.filterBeforeCreate)
        ? (toast: ToastOptions) => toast
        : this.defaults.filterBeforeCreate;
      const toast = filterBeforeCreate(props, this.toastArray);
      toast && this.setToast(toast);
    },
    dismissToast(id: ToastID) {
      const toast = this.toasts[id];
      if (!isUndefined(toast) && !isUndefined(toast.onClose)) {
        toast.onClose();
      }
      this.$delete(this.toasts, id);
    },
    clearToasts() {
      Object.keys(this.toasts).forEach((id: ToastID) => this.dismissToast(id));
    },
    getPositionToasts(position: POSITION) {
      const toasts = this.filteredToasts
        .filter(toast => toast.position === position)
        .slice(0, this.defaults.maxToasts);
      return this.defaults.newestOnTop ? toasts.reverse() : toasts;
    },
    updateDefaults(update: PluginOptions) {
      // Update container if changed
      if (!isUndefined(update.container)) {
        this.setup(update.container);
      }
      this.defaults = Object.assign({}, this.defaults, update);
    },
    updateToast({
      id,
      options,
      create
    }: {
      id: ToastID;
      options: ToastOptions;
      create: boolean;
    }) {
      if (this.toasts[id]) {
        // If a timeout is defined, and is equal to the one before, change it
        // a little so the progressBar is reset
        if (options.timeout && options.timeout === this.toasts[id].timeout) {
          options.timeout++;
        }
        this.setToast(Object.assign({}, this.toasts[id], options));
      } else if (create) this.addToast(Object.assign({}, { id }, options));
    },
    getClasses(position: string) {
      const classes = [`${VT_NAMESPACE}__container`, position];
      return classes.concat(this.containerClassName);
    }
  }
});
</script>