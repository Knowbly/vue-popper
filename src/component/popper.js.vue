<style>
  .popper {
    width: auto;
    background-color: #fafafa;
    color: #212121;
    text-align: center;
    padding: 2px;
    display: inline-block;
    border-radius: 3px;
    position: absolute;
    font-size: 14px;
    font-weight: normal;
    border: 1px #ebebeb solid;
    z-index: 200000;
    -moz-box-shadow: rgb(58, 58, 58) 0 0 6px 0;
    -webkit-box-shadow: rgb(58, 58, 58) 0 0 6px 0;
    box-shadow: rgb(58, 58, 58) 0 0 6px 0;
  }

  .popper .popper__arrow {
    width: 0;
    height: 0;
    border-style: solid;
    position: absolute;
    margin: 5px;
  }

  .popper[x-placement^="top"] {
    margin-bottom: 5px;
  }

  .popper[x-placement^="top"] .popper__arrow {
    border-width: 5px 5px 0 5px;
    border-color: #fafafa transparent transparent transparent;
    bottom: -5px;
    left: calc(50% - 5px);
    margin-top: 0;
    margin-bottom: 0;
  }

  .popper[x-placement^="bottom"] {
    margin-top: 5px;
  }

  .popper[x-placement^="bottom"] .popper__arrow {
    border-width: 0 5px 5px 5px;
    border-color: transparent transparent #fafafa transparent;
    top: -5px;
    left: calc(50% - 5px);
    margin-top: 0;
    margin-bottom: 0;
  }

  .popper[x-placement^="right"] {
    margin-left: 5px;
  }

  .popper[x-placement^="right"] .popper__arrow {
    border-width: 5px 5px 5px 0;
    border-color: transparent #fafafa transparent transparent;
    left: -5px;
    top: calc(50% - 5px);
    margin-left: 0;
    margin-right: 0;
  }

  .popper[x-placement^="left"] {
    margin-right: 5px;
  }

  .popper[x-placement^="left"] .popper__arrow {
    border-width: 5px 0 5px 5px;
    border-color: transparent transparent transparent #fafafa;
    right: -5px;
    top: calc(50% - 5px);
    margin-left: 0;
    margin-right: 0;
  }
</style>

<template>
  <component :is="tagName">
    <transition :name="transition" :enter-active-class="enterActiveClass" :leave-active-class="leaveActiveClass" @after-leave="doDestroy">
      <span
        ref="popper"
        v-show="!disabled && showPopper">
        <slot>{{ content }}</slot>
      </span>
    </transition>
    <slot name="reference"></slot>
  </component>
</template>

<script>
  import Popper from 'popper.js';
  import { throttle } from "lodash-es";

  function on(element, event, handler) {
    if (element && event && handler) {
      document.addEventListener ? element.addEventListener(event, handler, false) : element.attachEvent('on' + event, handler);
    }
  }

  function off(element, event, handler) {
    if (element && event) {
      document.removeEventListener ? element.removeEventListener(event, handler, false) : element.detachEvent('on' + event, handler)
    }
  }

  export default {
    props: {
      tagName: {
        type: String,
        default: 'span',
      },
      trigger: {
        type: String,
        default: 'hover',
        validator: value => ['click', 'hover'].indexOf(value) > -1
      },
      delayOnMouseOver: {
        type: Number,
        default: 10,
      },
      delayOnMouseOut: {
        type: Number,
        default: 10,
      },
      disabled: {
        type: Boolean,
        default: false
      },
      content: String,
      enterActiveClass: String,
      leaveActiveClass: String,
      boundariesSelector: String,
      reference: {},
      forceShow: {
        type: Boolean,
        default: false
      },
      dataValue: {
        default: null,
      },
      appendToBody: {
        type: Boolean,
        default: false
      },
      visibleArrow: {
        type: Boolean,
        default: true
      },
      transition: {
        type: String,
        default: ''
      },
      stopPropagation: {
        type: Boolean,
        default: false
      },
      preventDefault: {
        type: Boolean,
        default: false
      },
      options: {
        type: Object,
        default() {
          return {};
        }
      },
      closeOnScroll: {
        type: Boolean,
        default: false,
      }
    },

    data() {
      return {
        lastParentScrollY: null,
        referenceElm: null,
        popperJS: null,
        showPopper: false,
        currentPlacement: '',
        popperOptions: {
          placement: 'bottom',
          computeStyle: {
            gpuAcceleration: false
          }
        }
      };
    },

    watch: {
      showPopper(value) {
        if (value) {
          this.$emit('show', this);
          if (this.popperJS) {
            this.popperJS.enableEventListeners();
          }
          this.updatePopper();
        } else {
          if (this.popperJS) {
            this.stopListenParentScrollEvent();
            this.popperJS.disableEventListeners();
          }
          this.$emit('hide', this);
        }
      },

      forceShow: {
        handler(value) {
          this[value ? 'doShow' : 'doClose']();
        },
        immediate: true
      },

      disabled(value) {
        if (value) {
          this.showPopper = false;
        }
      }
    },

    created() {
      this.appendedArrow = false;
      this.appendedToBody = false;
      this.popperOptions = Object.assign(this.popperOptions, this.options);
      this.handleParentScrollThrottled = throttle(this.handleParentScroll, 20);
    },

    mounted() {
      this.referenceElm = this.reference || this.$slots.reference[0].elm;
      this.popper = this.$slots.default[0].elm;

      switch (this.trigger) {
        case 'click':
          on(this.referenceElm, 'click', this.doToggle);
          on(document, 'click', this.handleDocumentClick);
          break;
        case 'hover':
          on(this.referenceElm, 'mouseover', this.onMouseOver);
          on(this.referenceElm, 'focus', this.onMouseOver);
          on(this.popper, 'mouseover', this.onMouseOver);
          on(this.popper, 'focus', this.onMouseOver);
          on(this.referenceElm, 'mouseout', this.onMouseOut);
          on(this.referenceElm, 'blur', this.onMouseOut);
          on(this.popper, 'mouseout', this.onMouseOut);
          on(this.popper, 'blur', this.onMouseOut);
          break;
      }
    },

    methods: {
      doToggle(event) {
        if(this.stopPropagation) {
          event.stopPropagation();
        }

        if(this.preventDefault) {
          event.preventDefault();
        }

        if (!this.forceShow) {
          this.showPopper = !this.showPopper;
        }
      },

      doShow() {
        this.showPopper = true;
      },

      doClose() {
        this.showPopper = false;
      },

      doDestroy() {
        if (this.showPopper) {
          return;
        }

        if (this.popperJS) {
          this.popperJS.destroy();
          this.popperJS = null;
        }

        if (this.appendedToBody) {
          this.appendedToBody = false;
          document.body.removeChild(this.popper.parentElement);
        }
      },

      createPopper() {
        this.$nextTick(() => {
          if (this.visibleArrow) {
            this.appendArrow(this.popper);
          }

          if (this.appendToBody && !this.appendedToBody) {
            this.appendedToBody = true;
            document.body.appendChild(this.popper.parentElement);
          }

          if (this.popperJS && this.popperJS.destroy) {
            this.popperJS.destroy();
          }

          if (this.boundariesSelector) {
            const boundariesElement = document.querySelector(this.boundariesSelector);

            if (boundariesElement) {
              this.popperOptions.modifiers = Object.assign({}, this.popperOptions.modifiers);
              this.popperOptions.modifiers.preventOverflow = Object.assign({}, this.popperOptions.modifiers.preventOverflow);
              this.popperOptions.modifiers.preventOverflow.boundariesElement = boundariesElement;
            }
          }

          this.popperOptions.onCreate = () => {
            this.$emit('created', this);
            this.$nextTick(this.updatePopper);
          };

          this.popperJS = new Popper(this.referenceElm, this.popper, this.popperOptions);
          this.listenParentScrollEvent();
        });
      },

      destroyPopper() {
        off(this.referenceElm, 'click', this.doToggle);
        off(this.referenceElm, 'mouseup', this.doClose);
        off(this.referenceElm, 'mousedown', this.doShow);
        off(this.referenceElm, 'focus', this.doShow);
        off(this.referenceElm, 'blur', this.doClose);
        off(this.referenceElm, 'mouseout', this.onMouseOut);
        off(this.referenceElm, 'mouseover', this.onMouseOver);
        this.stopListenParentScrollEvent();
        off(document, 'click', this.handleDocumentClick);

        this.showPopper = false;
        this.doDestroy();
      },

      appendArrow(element) {
        if (this.appendedArrow) {
          return;
        }

        this.appendedArrow = true;

        const arrow = document.createElement('div');
        arrow.setAttribute('x-arrow', '');
        arrow.className = 'popper__arrow';
        element.appendChild(arrow);
      },

      updatePopper() {
        this.popperJS ? this.popperJS.scheduleUpdate() : this.createPopper();
      },

      onMouseOver() {
        clearTimeout(this._timer);
        this._timer = setTimeout(() => {
          this.showPopper = true;
        }, this.delayOnMouseOver);
      },

      onMouseOut() {
        clearTimeout(this._timer);
        this._timer = setTimeout(() => {
          this.showPopper = false;
        }, this.delayOnMouseOut);
      },

      handleDocumentClick(e) {
        if (!this.$el || !this.referenceElm ||
          this.elementContains(this.$el, e.target) ||
          this.elementContains(this.referenceElm, e.target) ||
          !this.popper || this.elementContains(this.popper, e.target)
        ) {
          return;
        }

        this.$emit('documentClick', this);

        if (this.forceShow) {
          return;
        }

        this.showPopper = false;
      },

      elementContains(elm, otherElm) {
        if (typeof elm.contains === 'function') {
          return elm.contains(otherElm);
        }

        return false;
      },

      listenParentScrollEvent() {
        const scrollElement = this.popperJS.state.scrollElement;
        if (this.closeOnScroll && scrollElement) {
          on(scrollElement, 'scroll', this.handleParentScrollThrottled);
        }
      },

      stopListenParentScrollEvent() {
        const scrollElement = this.popperJS && this.popperJS.state && this.popperJS.state.scrollElement;
        if (this.closeOnScroll && scrollElement) {
          off(scrollElement, 'scroll', this.handleParentScrollThrottled);
          this.lastParentScrollY = null;
        }
      },

      handleParentScroll(event) {
        if (event.currentTarget === null) {
          // this.popperJS.state.scrollElement is null
          // popper has been destroyed
          return;
        }
        const parentScrollY = event.currentTarget.scrollTop;
        if (this.lastParentScrollY !== null && parentScrollY !== this.lastParentScrollY) {
          this.doClose();
        }
        this.lastParentScrollY = parentScrollY;
      },
    },

    destroyed() {
      this.destroyPopper();
    }
  }
</script>
