<template>
  <div :class="['kBuilderBlock', 'kBuilderBlock--col-' + columnsCount, {'kBuilderBlock--pending': pending }]">
    <div :class="'kBuilderBlock__header kBuilderBlock__header--col-' + columnsCount" >
      <k-icon 
        type="sort" 
        :class="'kBuilder__dragDropHandle kBuilder__dragDropHandle--col-' + columnsCount" 
      />
      <span 
        class="kBuilderBlock__label"
        @click="toggleExpand"
      >
        <k-icon
          class="kBuilderBlock__expandedIcon" 
          :class="{'kBuilderBlock__expandedIcon--expanded': expanded}" 
          type="angle-down"
        />
        {{block.label}}
      </span>
      <div class="kBuilderBlock__actions">
        <k-button-group class="kBuilderBlock__actionsGroup">
          <k-button 
            v-if="fieldSets.length > 1 || block.preview"
            v-for="fieldSet in fieldSets"
            :key="'showFierldSetButton-' + _uid + fieldSet.key"
            :icon="tabIcon(fieldSet.icon)" 
            :image="tabImage(fieldSet.icon)"
            @click="displayFieldSet(fieldSet.key); toggleExpand(true)"
            class="kBuilderBlock__actionsButton"
            :class="{'kBuilderBlock__actionsButton--active': (activeFieldSet == fieldSet.key && expanded )}"
          >{{fieldSet.label}}</k-button>
          <k-button 
            v-if="block.preview"
            icon="preview" 
            @click="displayPreview()"
            class="kBuilderBlock__actionsButton" 
            :class="{'kBuilderBlock__actionsButton--active': showPreview && expanded}"
          ></k-button>
        </k-button-group>
        <div class="kBuilderBlock__control">
          <k-dropdown class="kBuilderBlock__actionsDropDown">
            <k-button 
              icon="dots" 
              @click="$refs['blockActions' + block.uniqueKey].toggle()"
              class="kBuilderBlock__actionsButton"
            ></k-button>
            <k-dropdown-content :ref="'blockActions' + block.uniqueKey">
              <k-dropdown-item 
                icon="copy" 
                @click="$emit('clone', index)"
              >{{ $t('builder.clone') }}</k-dropdown-item>
              <k-dropdown-item 
                icon="trash" 
                @click="$emit('delete', index)"
              >{{ $t('delete') }}</k-dropdown-item>
            </k-dropdown-content>
          </k-dropdown>
        </div>
      </div>
    </div>
    <div 
      class="kBuilderBlock__content"
      v-show="expanded"
    >
      <iframe 
        v-if="block.preview && showPreview && previewUrl"
        class="kBuilder__previewFrame" 
        @loaded="onPreviewLoaded"
        :style="{height: previewHeight + 'px'}"
        :src="previewUrl"
      ></iframe>
      <k-fieldset 
        v-if="(activeFieldSet === fieldSet.key)" 
        v-for="fieldSet in fieldSets"
        class="kBuilderBlock__form"
        v-model="fieldSet.model" 
        :value="{}" 
        :fields="fieldSet.fields" 
        :validate="true"
        v-on="$listeners"
        :key="fieldSet.key + _uid" 
      />
    </div>
  </div>
</template>

<script>
export default {
  props: {
    endpoints: Object,
    block: Object,
    index: Number,
    columnsCount: Number,
    pageUid: String,
    pageId: String,
  },
  mounted() {
    if (this.block.isNew) {
      this.$nextTick(function () {
        this.pending = false
      });
    } else {
      this.pending = false
    }
    if (!this.block.content._uid) {
      this.block.content._uid = this.block.content._key + '_' + new Date().valueOf() + '_' + this._uid
    }
    if (!this.activeFieldSet) {
      this.activeFieldSet = this.fieldSets[0].key
    }
    let localUiState = JSON.parse(localStorage.getItem(this.localUiStateKey))
    if (localUiState) {
      this.expanded = localUiState.expanded
      this.showPreview = localUiState.showPreview
      this.activeFieldSet = localUiState.activeFieldSet
    } else {
      this.storeLocalUiState()
    }
    if (this.block.preview && this.showPreview) {
      this.displayPreview(this.block.preview)
    } else {
      this.displayFieldSet(this.activeFieldSet)
    }
    if (this.block.isNew) {
      this.$emit('input')
    }
  },
  data() {
    return {
      pending: true,
      activeFieldSet: null,
      expanded: true,
      previewFrameContent: null,
      previewHeight: 0,
      previewStored: false,
      showPreview: false,
    }
  },
  computed: {
    localUiStateKey() {
      return `kBuilder.uiState.${this.block.content._uid}`
    },
    extendedUid() {
      return this.pageId.replace('/', '-') + '-' + this._uid
    },
    previewUrl() {
      if (this.previewStored) {
        return 'kirby-builder-preview/' + this.extendedUid + '?' + this.objectToGetParams(this.block.preview) + '&pageid=' + this.pageId
      } else {
        return null
      }
    },
    fieldSets() {
      let fieldSets = []
      let fields = {};
      if (this.block.tabs) {
        for (const tabKey in this.block.tabs) {
          if (this.block.tabs.hasOwnProperty(tabKey)) {
            const tab = this.block.tabs[tabKey];
            fieldSets.push(this.newFieldSet(tab, tabKey, this.block.content))
          }
        }
      } else if (this.block.fields) {
        let b = this.block;
        let bc = this.block.content;
        Object.keys(this.block.fields).forEach(name => {
          let field = this.block.fields[name];
          field.endpoints = {
            field: field.parent + "/fields/" + b.builderRootName + "+" + b.blockKey + "+" + field.name,
            // section: ? + "/sections/" + "images",
            // model: ?
          };
          fields[name] = field;
        });
        let fs = this.newFieldSet(fields, 'content', bc, 'edit', this.$t('edit'));
        fieldSets.push(fs);
      }
      return fieldSets;
    },
  },
  methods: {
    onBlockInput(event) {
      this.$emit("input", this.val)
    },
    displayPreview() {
      this.showPreview = true
      this.expanded = true
      let previewData = {
        preview: this.block.preview,
        blockcontent: this.block.content,
        blockUid: this.extendedUid
      }
      this.$api.post('kirby-builder/preview', previewData)
        .then(() => {
          this.previewStored = true
        })
      this.storeLocalUiState()
    },
    displayFieldSet(fieldSetKey) {
      this.showPreview = false
      this.activeFieldSet = fieldSetKey
      this.previewHeight = 0
      this.storeLocalUiState()
    },
    onPreviewLoaded(event) {
      this.previewHeight = event.detail.height
      this.activeFieldSet = null
    },
    toggleExpand(expanded) {
      if (typeof expanded === 'boolean') {
        this.expanded = expanded
      } else {
        this.expanded = (this.expanded) ? false : true
      }
      this.storeLocalUiState()
    },
    newFieldSet(fields, key, model, icon, label) {
      let newFieldSet = {
        fields: fields,
        key: key,
        model: model,
        icon: icon || fieldSet.icon || null,
        label: label || fieldSet.label || null,
      }
      return newFieldSet
    },
    objectToGetParams(obj) {
      return Object.keys(obj).map(function (key) {
        return key + '=' + obj[key];
      }).join('&');
    },
    storeLocalUiState() {
      let localUiState = {
        expanded: this.expanded,
        showPreview: this.showPreview,
        activeFieldSet: this.activeFieldSet,
      }
      localStorage.setItem(this.localUiStateKey, JSON.stringify(localUiState))
    },
    tabIcon(icon) {
      if (!icon) {
        return null
      }
      const isPath = (icon.indexOf('/') > -1 && icon.indexOf('.') > -1)
      return isPath ? null : icon
    },
    tabImage(icon) {
      if (!icon) {
        return null
      }
      const isPath = (icon.indexOf('/') > -1 && icon.indexOf('.') > -1)
      return isPath ? icon : null
    }
  }
}
</script>

<style lang="stylus">
  .kBuilderBlock
    background: white;
    box-shadow: 0 2px 5px rgba(22,23,26,.05);
    position: relative;
    opacity: 1;
    transition: opacity .5s, transform .5s;
    &--pending
      opacity: 0;
      transform: translateY(5%);
    &__label
      display flex
      cursor pointer
      height 100%
      align-items center
    &__expandedIcon
      margin-right 4px
      transform rotate(-90deg)
      &--expanded
        transform rotate(0)
    &__header
      height: 38px;    
      font-size: .875rem;
      // color: #999;
      display: flex;
      align-items: center;
      &--col-1
        padding-left: .75rem;
    &__actions
      position: absolute;
      right: 0;
      top: 0;
      display: flex;
    &__actionsGroup
      margin-right 0
      &.k-button-group>.k-button
        padding-top 0
        padding-bottom 0
    &__actionsDropDown
      display inline-block
    &__actionsButton
      min-width 38px
      height 38px
      opacity .5
      color rgb(22, 23, 26)
      &:hover
        opacity .7
      &--active
        pointer-events: none;
        opacity 1
      .k-button-figure img
        background-color transparent
        border-radius 0
    &__form
      padding: .625rem .75rem 2.25rem .75rem ;
    .sortable-drag
      cursor: -webkit-grab;
    &
    .kBuilderBlock
    .k-structure
    .k-card
    .k-list-item
      box-shadow: 0 2px 5px rgba(22,23,26,.15), 0 0 0 1px rgba(22,23,26,.05)
    .k-structure
      margin-left: 25px;
  .sortable-ghost > .k-column-content > .kBuilderBlock
  .sortable-ghost > .kBuilderBlock
    box-shadow 0 0 0 2px #4271ae, 0 5px 10px 2px rgba(22,23,26,.25)
</style>

