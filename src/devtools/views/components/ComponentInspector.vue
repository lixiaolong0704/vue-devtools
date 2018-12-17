<template>
  <scroll-pane>
    <action-header
      v-show="hasTarget"
      slot="header"
    >
      <span class="title">
        <span class="title-bracket">&lt;</span>
        <span>{{ targetName }}</span>
        <span class="title-bracket">&gt;</span>
      </span>
      <div class="search">
        <VueIcon icon="search" />
        <input
          v-model.trim="filter"
          placeholder="Filter inspected data"
        >
      </div>
      <a
        v-if="$isChrome"
        v-tooltip="'Inspect DOM'"
        class="button inspect"
        @click="inspectDOM"
      >
        <VueIcon icon="code" />
        <span>Inspect DOM</span>
      </a>
      <a
        v-if="fileIsPath"
        v-tooltip="$t('ComponentInspector.openInEditor.tooltip', { file: target.file })"
        class="button"
        @click="openInEditor"
      >
        <VueIcon icon="launch" />
        <span>Open in editor</span>
      </a>
        <a
                v-tooltip="'Copy'"
                class="button"
                @click="Copy"
        >
            <span>Copy</span>

        </a>
    </action-header>
    <template slot="scroll">
      <section
        v-if="!hasTarget"
        class="notice"
      >
        <div>Select a component instance to inspect.</div>
      </section>
      <div
        v-else-if="!target.state || !target.state.length"
        class="notice"
      >
        <div>This instance has no reactive state.</div>
      </div>
      <section
        v-else
        class="data"
      >
        <state-inspector
          :state="filteredState"
          class="component-state-inspector"
        />
      </section>
    </template>
  </scroll-pane>
</template>

<script>
import ScrollPane from 'components/ScrollPane.vue'
import ActionHeader from 'components/ActionHeader.vue'
import StateInspector from 'components/StateInspector.vue'
import { searchDeepInObject, sortByKey, classify, openInEditor } from 'src/util'
import groupBy from 'lodash.groupby'
import { mapState } from 'vuex'
// Copies a string to the clipboard. Must be called from within an
// event handler such as click. May return false if it failed, but
// this is not always possible. Browser support for Chrome 43+,
// Firefox 42+, Safari 10+, Edge and IE 10+.
// IE: The clipboard feature may be disabled by an administrator. By
// default a prompt is shown the first time the clipboard is
// used (per session).
function copyToClipboard(text) {
    if (window.clipboardData && window.clipboardData.setData) {
        // IE specific code path to prevent textarea being shown while dialog is visible.
        return clipboardData.setData("Text", text);

    } else if (document.queryCommandSupported && document.queryCommandSupported("copy")) {
        var textarea = document.createElement("textarea");
        textarea.textContent = text;
        textarea.style.position = "fixed";  // Prevent scrolling to bottom of page in MS Edge.
        document.body.appendChild(textarea);
        textarea.select();
        try {
            return document.execCommand("copy");  // Security exception may be thrown by some browsers.
        } catch (ex) {
            console.warn("Copy to clipboard failed.", ex);
            return false;
        } finally {
            document.body.removeChild(textarea);
        }
    }
}
export default {
  components: {
    ScrollPane,
    ActionHeader,
    StateInspector
  },

  props: {
    target: {
      type: Object,
      required: true
    }
  },

  data () {
    return {
      filter: ''
    }
  },

  computed: {
  ...mapState('components', [
      'inspectedInstance'
  ]),
    hasTarget () {
      return this.target.id != null
    },

    targetName () {
      return this.$shared.classifyComponents ? classify(this.target.name) : this.target.name
    },

    filteredState () {
      return groupBy(sortByKey(this.target.state.filter(el => {
        return searchDeepInObject({
          [el.key]: el.value
        }, this.filter)
      })), 'type')
    },

    // Checks if the file is actually a path (e.g. '/path/to/file.vue'), or
    // only the basename of a pre-compiled 3rd-party component (e.g. 'file.vue')
    fileIsPath () {
      return this.target.file && /[/\\]/.test(this.target.file)
    }
  },

  methods: {
    Copy(){
        console.log(this.target.state);
        console.dir(this.filteredState);
        //JSON.stringify(obj, null, 2);
        /* Get the text field */
        if(this.inspectedInstance && this.inspectedInstance.infos){
            copyToClipboard(JSON.stringify(this.inspectedInstance.infos, null, 2))
        }




    },
    inspectDOM () {
      if (!this.hasTarget) return
      if (this.$isChrome) {
        chrome.devtools.inspectedWindow.eval(
          `inspect(window.__VUE_DEVTOOLS_INSTANCE_MAP__.get("${this.target.id}").$el)`
        )
      } else {
        window.alert('DOM inspection is not supported in this shell.')
      }
    },

    openInEditor () {
      const file = this.target.file
      openInEditor(file)
    }
  }
}
</script>

<style lang="stylus" scoped>
.title
  white-space nowrap
  position relative
  top -1px
</style>
