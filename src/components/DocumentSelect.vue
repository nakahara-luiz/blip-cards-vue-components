<template>
  <div
    v-if="!isEditing"
    class="blip-container document-select"
    :class="isFailedMessage(status, position)"
  >
    <div :class="'bubble ' + position">
      <bds-button-icon v-if="deletable && !isEditing"
        class="editIco trashIco icon-margin"
        icon="trash"
        variant="delete"
        size="short"
        v-on:click="trash(document)"
      ></bds-button-icon>
      <bds-button-icon v-if="editable && !isEditing"
        class="editIco icon-margin"
        icon="edit"
        variant="primary"
        size="short"
        v-on:click="toggleEdit"
      ></bds-button-icon> 
      <div class="header">
        <div
          v-if="this.document.header.value.uri"
          :class="'ratio ratio' + aspect"
          :style="'background-image: url(&quot;' + this.document.header.value.uri + '&quot;)'"
        ></div>

        <div
          class="title"
          v-if="document.header.value.title || document.header.value.text || document.header.value"
        >
          <strong
            class="hide-overflow"
            v-if="document.header.value.title"
            v-html="sanitize(document.header.value.title)"
          ></strong>
          <span
            class="hide-overflow"
            v-if="previewContent && !showContent"
            v-html="sanitize(previewContent)"
          ></span>
          <transition name="slide-fade">
            <div
              v-if="showContent && hasPreview"
              v-html="sanitize(document.header.value.text || document.header.value)"
            ></div>
          </transition>
          <a
            style="display: block;"
            v-if="!showContent && hasPreview"
            @click="showContent = true"
          >{{ showMoreMsg }}</a>
        </div>
      </div>
      <div class="fixed-options" v-if="options">
        <ul>
          <li
            v-for="(item, index) in options"
            v-bind:key="index"
            @click="disableOptions || editable ? null : select(item, index)"
            :class="`${disableOptions ? ' disabled-options pointer-default' : ''} ${selectedIndex != undefined && selectedIndex == index ? ' clicked-option' : ''} ${editable ? '' : ' pointer'}`"
          >
            <span class="disable-selection" v-html="sanitize(item.previewText)"></span>
          </li>
        </ul>
      </div>
    </div>

    <blip-card-date
      :status="status"
      :position="position"
      :date="date"
      :failed-to-send-msg="failedToSendMsg"
      :is-external-message="isExternalMessage"
      :external-message-text="externalMessageText"
    />
  </div>

  <div v-else class="blip-container document-select">
    <form
      v-if="!showOptionDialog"
      :class="'editing bubble ' + position"
      novalidate
      v-on:submit.prevent
    >
    <bds-button-icon 
      class="btn saveIco closeIco"
      icon="close"
      variant="ghost"
      size="short"
      v-on:click="cancel()"
    ></bds-button-icon>
    <bds-button-icon 
      class="btn saveIco"
      icon="check"
      variant="primary"
      size="short"
      :disabled="errors.any()"
      v-on:click="documentSelectSave()"
    ></bds-button-icon>
      <div class="header">
        <div class="form-group">
          <input
            type="text"
            name="image"
            :class="{'input-error': errors.has('image') }"
            class="form-control"
            v-model="previewUri"
            :placeholder="imageUriMsg"
          >
          <div class="upload-intructions">
            <span>{{ supportedFormatsMsg }}: JPEG,JPG,PNG,GIF.</span>
          </div>
        </div>
        <div class="form-check">
          <div>
            <span>{{ aspectRatioMsg }}:</span>
          </div>
          <div class="form-check-wrapper">
            <span class="form-check-container">
              <label class="form-check-label" :for="_uid+'1-1'">
                <input
                  type="radio"
                  name="aspect-selector"
                  :id="_uid+'1-1'"
                  class="form-check-input"
                  v-model="aspect"
                  value="1-1"
                >
                <span class="radio">1:1</span>
                <div class="check"></div>
              </label>
            </span>
            <span class="form-check-container">
              <label class="form-check-label" :for="_uid+'2-1'">
                <input
                  type="radio"
                  name="aspect-selector"
                  :id="_uid+'2-1'"
                  class="form-check-input"
                  v-model="aspect"
                  value="2-1"
                >
                <span class="radio">2:1</span>
                <div class="check"></div>
              </label>
            </span>
          </div>
        </div>
        <div class="form-group">
          <input
            type="title"
            name="title"
            :class="{'input-error': errors.has('title') }"
            class="form-control"
            v-model="title"
            v-validate="'required'"
            :placeholder="titleMsg"
          >
          <span v-show="errors.has('title')" class="help input-error">{{ errors.first('title') }}</span>
          <textarea
            @keydown.enter="documentSelectSave($event)"
            type="text"
            name="text"
            :class="{'input-error': errors.has('text') }"
            class="form-control textarea"
            v-model="content"
            :placeholder="descriptionMsg"
          />
          <span v-show="errors.has('text')" class="help input-error">{{ errors.first('text') }}</span>
        </div>
      </div>

      <div class="fixed-options" v-if="options">
        <ul>
          <li v-for="(item, index) in options" v-bind:key="index">
            <span @click="editOption(item, index, $event)" v-html="sanitize(item.previewText)"></span>
            <span @click="deleteOption(index)" class="remove-option">
              <bds-icon name="close"></bds-icon>
            </span>
          </li>
        </ul>
        <div @click="editOption({label: {}, value: {}}, -1, $event, true)" class="btn-dashed primary-color btn">
          <span>{{ addButtonMsg }}</span>
        </div>
      </div>
      <button
        v-if="typeof onMetadataEdit === 'function'"
        class="define-metadata blip-document-select-metadata"
        @click="editMetadata(fullDocument)"
      >{{ metadataButtonText }}</button>
    </form>
    <form v-else novalidate v-on:submit.prevent :class="'editing bubble ' + position">
      <div class="tabs">
        <span :class="{ 'active': headerTab === 'plainText'}" @click="setTab('plainText')">{{ textMsg }}</span>
        <span :class="{ 'active': headerTab === 'weblink'}" @click="setTab('weblink')">{{ linkMsg }}</span>
      </div>

      <div class="form-group" v-if="headerTab === 'weblink'">
        <input
          type="text"
          name="optionTitle"
          class="form-control"
          v-model="selectedOption.label.value.title"
          :placeholder="titleMsg"
        >
      </div>
      <div class="form-group" v-if="headerTab === 'weblink'">
        <input
          type="text"
          name="optionText"
          class="form-control"
          v-validate="'required'"
          v-model="selectedOption.label.value.text"
          :placeholder="textMsg"
        >
        <span
          v-show="errors.has('optionText')"
          class="help input-error"
        >{{ errors.first('optionText') }}</span>
      </div>
      <div class="form-group" v-if="headerTab === 'weblink'">
        <input
          type="text"
          name="weblinkUri"
          class="form-control"
          v-validate="'required'"
          v-model="selectedOption.label.value.uri"
          :placeholder="uriMsg"
          @blur="fetchMetadaForSelectedOption()"
        >
        <span
          v-show="errors.has('weblinkUri')"
          class="help input-error"
        >{{ errors.first('weblinkUri') }}</span>
      </div>
      <div class="form-group" v-if="headerTab === 'weblink'">
        <select v-model="selectedOption.label.value.target" class="form-control">
          <option disabled value>Target</option>
          <option value="blank">Blank</option>
          <option value="self">Self</option>
          <option value="selfCompact">SelfCompact</option>
          <option value="selfTall">SelfTall</option>
        </select>
      </div>

      <div v-if="headerTab === 'plainText'">
        <div class="form-group">
          <input
            type="text"
            name="optionText"
            v-validate="'required'"
            class="form-control"
            v-model="selectedOption.label.value"
            :placeholder="textMsg"
          >
          <span
            v-show="errors.has('optionText')"
            class="help input-error"
          >{{ errors.first('optionText') }}</span>
        </div>

        <input id="showPayload" type="checkbox" v-model="showPayload">
        <label for="showPayload">{{ setPayloadMsg }}</label>
        <div class="line"></div>

        <div v-if="showPayload">
          <div class="form-group">
            <input
              type="text"
              name="type"
              v-validate="'required|mime'"
              class="form-control"
              v-model="selectedOption.value.type"
              :placeholder="postbackMimetypeMsg"
            >
            <span v-show="errors.has('type')" class="help input-error">{{ errors.first('type') }}</span>
          </div>
          <div class="form-group">
            <textarea
              @keydown.enter="saveOption(true, $event)"
              v-if="selectedOption.value.type && selectedOption.value.type.includes('json')"
              type="text"
              name="value"
              v-validate="'required|json'"
              class="form-control"
              v-model="selectedOption.value.value"
              :placeholder="postbackValueMsg"
            />
            <textarea
              @keydown.enter="saveOption(true, $event)"
              v-else
              type="text"
              name="value"
              v-validate="'required'"
              class="form-control"
              v-model="selectedOption.value.value"
              :placeholder="postbackValueMsg"
            />
            <span v-show="errors.has('value')" class="help input-error">{{ errors.first('value') }}</span>
          </div>
        </div>
      </div>

      <div class="form-group blip-card-flex">
        <span class="flex-item">
          <button type="button" @click="cancelOption()" class="btn btn-white color-primary">{{ cancelMsg }}</button>
        </span>
        <span class="flex-item">
          <button
            @click="saveOption(true)"
            class="btn btn-white primary-color"
            :class="{'is-disabled': errors.any() }"
          >{{ applyMsg }}</button>
        </span>
      </div>
    </form>
  </div>
</template>

<script>
import { linkify, isFailedMessage } from '../utils/misc'
import cloneDeep from 'lodash/cloneDeep'
import { default as base } from '../mixins/baseComponent.js'
import { MetadataService } from '../utils/metadataService.js'
import mime from 'mime-types'
const optionSize = 34

let getOptionContent = function(item) {
  let text =
    item.label.value.text ||
    item.label.value.title ||
    item.label.value.uri ||
    item.label.value
  if (text.length > optionSize) {
    return text.substring(0, optionSize) + '...'
  } else {
    return text
  }
}

export default {
  name: 'document-select',
  mixins: [base],
  props: {
    length: {
      type: Number,
      default: 532
    },
    status: {
      type: String,
      default: ''
    },
    onSelected: {
      type: Function
    },
    onOpenLink: {
      type: Function
    },
    failedToSendMsg: {
      type: String,
      default: 'Falha ao enviar a mensagem'
    },
    applyMsg: {
      type: String,
      default: 'Apply'
    },
    cancelMsg: {
      type: String,
      default: 'Cancel'
    },
    setPayloadMsg: {
      type: String,
      default: 'Set payload'
    },
    addButtonMsg: {
      type: String,
      default: 'Add Button'
    },
    textMsg: {
      type: String,
      default: 'Text'
    },
    linkMsg: {
      type: String,
      default: 'Link'
    },
    aspectRatioMsg: {
      type: String,
      default: 'Aspect Ratio'
    },
    showMoreMsg: {
      type: String,
      default: 'Ver mais'
    },
    supportedFormatsMsg: {
      type: String,
      default: 'Supported formats'
    },
    imageUriMsg: {
      type: String,
      default: 'Image Uri'
    },
    titleMsg: {
      type: String,
      default: 'Title'
    },
    descriptionMsg: {
      type: String,
      default: 'Description'
    },
    uriMsg: {
      type: String,
      default: 'Uri'
    },
    postbackMimetypeMsg: {
      type: String,
      default: 'Postback mime type'
    },
    postbackValueMsg: {
      type: String,
      default: 'Postback value'
    },
    disableOptions: {
      type: Boolean,
      default: false
    }
  },
  data: function() {
    return {
      showOptionDialog: undefined,
      headerTab: undefined,
      showPayload: undefined,
      showContent: undefined,
      title: undefined,
      content: undefined,
      aspect: undefined,
      previewUri: undefined,
      selectedOption: undefined,
      options: undefined,
      MetadataService: new MetadataService(),
      selectedIndex: undefined,
      isFailedMessage
    }
  },
  watch: {
    document: function() {
      this.title = this.document.header.value.title
      this.content = this.document.header.value.text
      this.aspect = this.document.header.value.aspectRatio
        ? this.document.header.value.aspectRatio.replace(':', '-')
        : '2-1'
      this.previewUri = this.document.header.value.uri
      this.options = this.document.options.map(function(x) {
        let opt = {
          ...x,
          isLink: x.label.type === 'application/vnd.lime.web-link+json',
          value: x.value
            ? {
              type: x.value.type,
              value: x.value.type.includes('json')
                ? JSON.stringify(x.value.value)
                : x.value.value
            }
            : {}
        }

        opt.previewText = getOptionContent(opt)
        return opt
      })
    }
  },
  computed: {
    type: function() {
      return mime.lookup(this.previewUri)
        ? mime.lookup(this.previewUri)
        : 'image/jpeg'
    },
    previewContent: function() {
      if (this.document.header.value.text) {
        if (this.document.header.value.text.length > this.length) {
          return (
            linkify(this.document.header.value.text.substring(0, this.length)) +
            '...'
          )
        }
        return linkify(this.document.header.value.text)
      }

      if (
        this.document.header.value &&
        this.document.header.type === 'text/plain'
      ) {
        if (this.document.header.value.length > this.length) {
          return (
            linkify(this.document.header.value.substring(0, this.length)) +
            '...'
          )
        }
        return this.document.header.value
          ? linkify(this.document.header.value)
          : ''
      }

      return ''
    },
    hasPreview: function() {
      return (
        (this.document.header.value.text &&
          this.document.header.value.text.length > this.length) ||
        (this.document.header.value &&
          this.document.header.value.length > this.length)
      )
    }
  },
  methods: {
    init: function() {
      this.showOptionDialog = false
      this.headerTab = null
      this.showPayload = false
      this.showContent = false
      this.title = this.document.header.value.title
      this.content = this.document.header.value.text
      this.aspect = this.document.header.value.aspectRatio
        ? this.document.header.value.aspectRatio.replace(':', '-')
        : '2-1'
      this.previewUri = this.document.header.value.uri
      this.selectedOption = { label: {}, value: {} }
      this.options = this.document.options.map((x) => {
        let opt = {
          ...x,
          isLink: x.label.type === 'application/vnd.lime.web-link+json',
          value: x.value
            ? {
              type: x.value.type,
              value: x.value.type.includes('json')
                ? JSON.stringify(x.value.value)
                : x.value.value
            }
            : {}
        }
        opt.previewText = getOptionContent(opt)
        return opt
      })
    },
    setTab: function(name) {
      this.headerTab = name
      if (
        this.headerTab === 'weblink' &&
        typeof this.selectedOption.label.value !== 'object'
      ) {
        this.selectedOption.label.value = {
          text: this.selectedOption.label.value,
          uri: this.selectedOption.LinkUri,
          target: this.selectedOption.LinkTarget
        }
      }
      if (
        this.headerTab === 'plainText' &&
        typeof this.selectedOption.label.value !== 'string'
      ) {
        this.selectedOption.LinkUri = this.selectedOption.label.value.uri
        this.selectedOption.LinkTarget =
          this.selectedOption.label.value.target || 'blank'
        this.selectedOption.label.value = this.selectedOption.label.value.text
      }
    },
    toggleShowPayload: function() {
      this.showPayload = !this.showPayload
    },
    select: function(item, index) {
      this.selectedIndex = index

      if (item.isLink) {
        if (
          item.label.value.target === 'blank' ||
          item.label.value.target === '' ||
          item.label.value.target === undefined
        ) {
          let win = window.open(item.label.value.uri, '_blank')
          win.focus()
        } else if (this.onOpenLink) {
          this.onOpenLink({
            uri: item.label.value.uri,
            target: item.label.value.target,
            title: item.label.value.title || item.label.value.text,
            image: item.label.value.previewUri
          })
        }
        return
      }

      if (this.onSelected) {
        this.onSelected(item.label.value, {
          type: item.value.type || 'text/plain',
          content: item.value.value || item.label.value
        })
      }
    },
    documentSelectSave: async function($event) {
      if (this.errors.any() || ($event && $event.shiftKey)) {
        return
      }

      if ($event) {
        $event.stopPropagation()
        $event.preventDefault()
        $event.returnValue = false
      }

      let result = await this.$validator.validateAll()
      if (!result) return
      this.selectedOption = { label: {}, value: {} }

      var newDocument = {
        ...this.document,
        header: {
          ...this.document.header,
          value: {
            ...this.document.header.value,
            type: this.type
          }
        },
        options: this.options.map(function(x) {
          let value

          if (x.value && x.value.value) {
            if (x.value.type.includes('json')) {
              value = { type: x.value.type, value: JSON.parse(x.value.value) }
            } else {
              value = { type: x.value.type, value: x.value.value }
            }
          } else {
            value = null
          }

          return {
            label: {
              type: x.label.type,
              value: x.label.value
            },
            value
          }
        })
      }
      newDocument.header.value.title = this.title
      newDocument.header.value.text = this.content
      newDocument.header.value.type = this.type
      newDocument.header.value.aspectRatio = this.aspect.replace('-', ':')
      newDocument.header.value.uri = this.previewUri

      this.headerTab = null
      this.showOptionDialog = false
      this.save(newDocument)
    },
    editOption: function(item, index, $event) {
      this.showOptionDialog = true

      this.selectedOption = cloneDeep(item)
      this.selectedOption.index = index

      this.showPayload = typeof this.selectedOption.value.type === 'string'
      this.headerTab =
        this.selectedOption.label.type === 'application/vnd.lime.web-link+json'
          ? 'weblink'
          : 'plainText'
      this.selectedOption.LinkTarget =
        (this.selectedOption.label.value
          ? this.selectedOption.label.value.target
          : '') || ''
      if (this.headerTab === 'weblink') {
        this.selectedOption.label.value.target =
          this.selectedOption.label.value.target || 'blank'
      }
    },
    deleteOption: function(index) {
      this.options.splice(index, 1)
    },
    saveOption: function(reset, $event) {
      if (this.errors.any() || ($event && $event.shiftKey)) {
        return
      }

      if ($event) {
        $event.stopPropagation()
        $event.preventDefault()
        $event.returnValue = false
      }

      this.$validator.validateAll().then((result) => {
        if (!result) return
        if (this.headerTab === 'weblink') {
          this.selectedOption.label.type = 'application/vnd.lime.web-link+json'
          this.selectedOption.isLink = true
          this.selectedOption.value = {}
        } else {
          this.selectedOption.label.type = 'text/plain'
          this.selectedOption.isLink = false
          this.selectedOption.value = this.showPayload
            ? this.selectedOption.value
            : {}
        }
        this.selectedOption.label.value = this.selectedOption.label.value
        this.selectedOption.previewText = getOptionContent(this.selectedOption)

        if (this.selectedOption.index === -1) {
          this.options.push(this.selectedOption)
        } else {
          this.options.splice(this.selectedOption.index, 1, this.selectedOption)
        }

        if (reset) {
          this.selectedOption = { label: {}, value: {} }
          this.headerTab = null
          this.showOptionDialog = false
        }
      })
    },
    cancelOption: function(item) {
      this.errors.clear()
      this.selectedOption = { label: {}, value: {} }
      this.headerTab = null
      this.showOptionDialog = false
    },
    fetchMetadaForSelectedOption: async function() {
      let currentOption = this.selectedOption

      // Only fetch metadata if editing or missing one of options properties
      if (
        !this.isEditing &&
        currentOption.label.value.previewUri &&
        (currentOption.label.value.title !== undefined ||
          currentOption.label.value.text)
      ) {
        return
      }

      const fetchResult = await this.MetadataService.fetchMetadata(
        currentOption.label.value
      )

      if (fetchResult) {
        currentOption.label.value.title =
          currentOption.label.value.title === undefined
            ? fetchResult.title
            : currentOption.label.value.title
        currentOption.label.value.text =
          currentOption.label.value.text || fetchResult.text
        currentOption.label.value.previewUri = fetchResult.imgPreview
        this.selectedOption = currentOption
        // When fetch metadata is done after option is saved should update option
        if (!this.selectedOption.label.value) {
          this.headerTab = 'weblink'
          this.selectedOption.index = this.options.length - 1
          this.saveOption()
        }
      }
    }
  }
}
</script>

<style lang="scss">
@import '../styles/variables.scss';

.pointer-default {
  cursor: default !important;
}

.disabled-options {
  opacity: 0.33;
}

.clicked-option {
  opacity: 1 !important;
}

.document-select {
  .bubble {
    width: $bubble-width;
  }

  .header {
    .title {
      margin: 0;
      padding: 10px 20px;
      text-overflow: ellipsis;
      overflow: hidden;
    }
  }
  .hide-overflow {
    text-overflow: ellipsis;
    overflow: hidden;
  }
  .fixed-options ul {
    margin: 0px;
  }
  .editing {
    flex-direction: column;
  }
  .editing .fixed-options ul {
    margin: 0px -10px;
  }

  .add-button {
    cursor: pointer;
    text-align: center;
    padding: 5px;
    margin: 2px;
    font-size: 16px;
    font-weight: 500;
    color: $vue-light-blip;
    background-color: $vue-white !important;
    border: 1px dashed $vue-london !important;
  }

  .form-check {
    padding: 0px 10px;
    margin: 0;

    input[type='radio'] {
      position: absolute;
      width: 100%;
      height: 100%;
      top: 0;
      left: 0;
      visibility: hidden;
    }

    .form-check-container {
      margin-left: 5px;
      margin-top: 8px;
      position: relative;
      width: 40px;
      height: 20px;
      display: inline-block;
    }

    label {
      position: relative;
      z-index: 1;
      cursor: pointer;
    }
    .check {
      display: block;
      position: absolute;
      border: 1px solid $color-content-default;
      border-radius: 100%;
      height: 16px;
      width: 16px;
      top: 0px;
      left: 0px;
      transition: border 0.25s linear;
      -webkit-transition: border 0.25s linear;
      -moz-transition: border 0.25s linear;
      -ms-transition: border 0.25s linear;
    }
    .check::before {
      display: block;
      position: absolute;
      content: '';
      border-radius: 100%;
      height: 8px;
      width: 8px;
      top: 3px;
      left: 3px;
      margin: auto;
      transition: background 0.25s linear;
      -webkit-transition: background 0.25s linear;
      -moz-transition: background 0.25s linear;
      -ms-transition: background 0.25s linear;
    }
    input[type='radio']:checked ~ .check {
      border: 1px solid $color-content-default;
    }

    input[type='radio']:checked ~ .check::before {
      background: $color-primary;
    }
  }

  .blip-document-select-metadata {
    padding: 10px 5px 0 0;
    text-align: right;
  }
  .icon-margin{
    margin-right: 30%;
  }
}
</style>