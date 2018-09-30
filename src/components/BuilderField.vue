<template>
  <k-field 
    :label="label"	
    class="kBuilder"
    :class="'kBuilder--col-' + columnsCount"
  >
    <k-draggable 
      class="kBuilder__blocks k-grid" 
      @end="drag=false" 
      v-model="blocks"
      @update="onBlockMoved"
      @add="onBlockAdded"
      @remove="onBlockRemoved"
      :move="onMove"
      :options="draggableOptions"
    >
      <k-column 
        class="kBuilder__column"
        :width="columnWidth"
        v-for="(block, index) in blocks" 
        :key="block.uniqueKey" 
      >
        <div 
          class="kBuilder__inlineAddButton"
          :class="{'kBuilder__inlineAddButton--horizontal': (columnsCount == 1), 'kBuilder__inlineAddButton--vertical': (columnsCount > 1)}"
          @click="onClickAddBlock(index)"
        ></div>
        <builder-block 
          :block="block" 
          :index="index"
          :columnsCount="columnsCount"
          :pageUid="pageUid" 
          :pageId="pageId" 
          :showPreview.sync="block.showPreview" 
          @input="onBlockInput" 
          @cloneBlock="cloneBlock"
          @deleteBlock="deleteBlock"/>
        <div 
          v-if="(columnsCount % index == 0 && columnsCount > 1)"
          class="kBuilder__inlineAddButton kBuilder__inlineAddButton--vertical kBuilder__inlineAddButton--after"
          @click="onClickAddBlock(index + 1)"
        ></div>
      </k-column >
      <k-column :width="columnWidth" v-if="!limit || blockCount < limit">
        <k-button 
          icon="add" 
          @click="onClickAddBlock()"
          class="kBuilder__addButton"
        >
          {{addBlockButtonLabel}}
        </k-button>
      </k-column>
    </k-draggable>
    <k-dialog 
      ref="dialog" 
    >
      <k-list>
        <k-list-item
          class="kBuilder__addBlockButton"
          v-for="(value, key) in fieldsets" 
          :key="key" 
          @click="addBlock(key)"
          :text="value.label"   
        />
      </k-list>
    </k-dialog>
  </k-field>
</template>

<script>
import BuilderBlock from "./BuilderBlock.vue";
export default {
  props: {
    value: String,
    fieldsets: Object,
    columns: Number,
    limit: Number,
    label: String,
    preview: Object,
    pageId: String,
    pageUid: String,
  },
  components: { BuilderBlock },
  mounted() {
    if (this.value) {
      this.value.forEach((block, index) => {
        let fieldSet = this.fieldsets[block._key]
        this.blocks.push(this.newBlock(fieldSet, block._key, block, index))
      });
      this.lastUniqueKey = this.value.length
    }
  },
  data() {
    return {
      blocks: [],
      toggle: true,
      targetPosition: null,
      lastUniqueKey: 0,
      dataField: {
        label: 'date label',
        type: 'date'
      },
      dateValue: null
    }
  },
  computed: {
    val() {
      return this.blocks.map(block => block.content)
    },
    columnsCount() {
      return this.columns ? this.columns : '1'
    },
    columnWidth() {
      return this.columns ? '1/' + this.columns : '1/1'
    },
    draggableOptions(){
        return { 
          group:'kirby-builder', 
          put:'kirby-builder', 
          clone: true, 
          forceFallback: true, 
          handle: '.kBuilder__dragDropHandle', 
          scroll: true 
        }
    },
    blockCount() {
      return this.blocks.length
    },
    fieldsetCount() {
      return Object.keys(this.fieldsets).length
    },
    fieldsetKeys() {
      return Object.keys(this.fieldsets)
    },
    addBlockButtonLabel() {
      if (this.fieldsetCount == 1) {
        return `Add ${this.fieldsets[Object.keys(this.fieldsets)[0]].label}`
      } else {
        return 'Add'
      }
    },
    supportedBlockTypes() {
      return Object.keys(this.fieldsets)
    }
  },
  methods: {
    onBlockInput(event) {
      this.$emit("input", this.val);
    },
    onBlockMoved(event) {
      this.$emit("input", this.val);
    },
    onBlockAdded(event) {
      this.$emit("input", this.val);
    },
    onBlockRemoved(event) {
      this.$emit("input", this.val);
    },
    onMove(event) {
      //Prevent sorting behind last element that contains the add button
      const isNotLastIndex = event.relatedContext.index != this.blocks.length + 1
      const isEmptyList = (this.blocks.length == 0)
      const isSupportedBlockType = this.supportedBlockTypes.includes(event.relatedContext.element.blockKey)
      return (isEmptyList || isNotLastIndex) && isSupportedBlockType
    },
    onClickAddBlock(position) {
      this.targetPosition = position
      if (this.fieldsetCount == 1) {
        this.addBlock(this.fieldsetKeys[0])
      } else {
        this.$refs.dialog.open()
      }
    },
    addBlock(key) {
      let position = this.targetPosition == null ? this.blocks.length : this.targetPosition
      let fieldSet = this.fieldsets[key]
      let newBlock = this.newBlock(fieldSet, key, this.getBlankContent(key, fieldSet), this.lastUniqueKey++)
      newBlock.isNew = true
      this.blocks.splice(position, 0, JSON.parse(JSON.stringify(newBlock)))
      this.$refs.dialog.close()
      this.targetPosition = null
      this.$emit("input", this.val);
    },
    getBlankContent(key, fieldSet) {
      let content = { '_key': key }
      if (fieldSet.fields) {
        Object.keys(fieldSet.fields).forEach(fieldName => {
          content[fieldName] = fieldSet.fields[fieldName].value || fieldSet.fields[fieldName].default || ''
        })
      } else if (fieldSet.tabs) {
        for (const tabName in fieldSet.tabs) {
          content[tabName] = {}
          if (fieldSet.tabs.hasOwnProperty(tabName)) {
            const tab = fieldSet.tabs[tabName];
            Object.keys(tab.fields).forEach(fieldName => {
              content[tabName][fieldName] = tab.fields[fieldName].value || tab.fields[fieldName].default || ''
            })
          }
        }
      }
      return content
    },
    cloneBlock(index) {
      let clone = JSON.parse(JSON.stringify(this.blocks[index]))
      this.removeProperty(clone.content, '_uid')
      this.blocks.splice(index + 1, 0, clone)
      this.blocks[index + 1].uniqueKey = this.lastUniqueKey++
      this.$emit("input", this.val);
    },
    removeProperty(obj, property) {
      for (prop in obj) {
        if (prop === property) {
          delete obj[prop]
        } else if (typeof obj[prop] === 'object') {
          this.removeProperty(obj[prop], property)
        }
      }
    },
    deleteBlock(index) {
      this.clearLocalUiStates(this.blocks[index])
      this.blocks.splice(index, 1);
      this.$emit("input", this.val);
    },
    clearLocalUiStates(obj) {
      for (const prop in obj) {
        if (obj.hasOwnProperty(prop)) {
          const element = obj[prop];
          if (prop === '_uid') {    
            localStorage.removeItem(`kBuilder.uiState.${obj[prop]}`)
          } else if (typeof obj[prop] === 'object') {
            this.clearLocalUiStates(obj[prop])
          }
        }
      }
    },
    newBlock(fieldSet, key, content, uniqueKey) {
      return {
        fields: fieldSet.fields ? fieldSet.fields : null,
        tabs: fieldSet.tabs ? fieldSet.tabs : null,
        blockKey: key,
        content: content,
        label: fieldSet.label,
        uniqueKey: uniqueKey,
        preview: fieldSet.preview,
        showPreview: false
      }
    }
  }
}
</script>

<style>
.kBuilder__addButton{
  width: 100%;
  background-color: transparent;
  padding: calc(.625rem * 4) .75rem;
  border: 1px dashed #CCC;
  transition: background-color .3s, border-color .3s;
}
.kBuilder__addButton:hover{
  background-color: #81a2be;
  border-color: transparent;
}
.kBuilder__addBlockButton{
  cursor: pointer;
}
.kBuilder .kBuilder--col-1{
  padding-left: 25px;
}
.kBuilder__dragDropHandle{
  width: 38px;
  height: 38px;
  color: #16171a;
  opacity: .25;
  z-index: 1;
  cursor: -webkit-grab;
  will-change: opacity,color;
  -webkit-transition: opacity .3s;
  transition: opacity .3s;
}
.kBuilder__dragDropHandle--col-1{
  position: absolute;
  left: -38px;
  top: 0;
  display: flex;
  opacity: 0;
}
.kBuilder__blocks:hover .kBuilder__dragDropHandle,
kBuilder__blocks:hover .kBuilder__dragDropHandle--col-1{
  opacity: .25;
}
.kBuilder__block .kBuilder__dragDropHandle:hover,
kBuilder__block:hover .kBuilder__dragDropHandle--col-1{
  opacity: 1;
}
.kBuilder__inlineAddButton{
  cursor: pointer;
  position: absolute;
  opacity: 0;
  transition: opacity .3s;  
}
.kBuilder__inlineAddButton:hover{
  opacity: 1;
}
.kBuilder__inlineAddButton::before{
  content: "";
  border-color: #4271ae;
  border-style: dashed;
  border-width: 0;
  display: block;
}
.kBuilder__inlineAddButton--horizontal{
  height: calc(.625rem * 2);
  width: 100%;
  bottom: 100%;
}
.kBuilder__inlineAddButton--horizontal::before{
  border-bottom-width: 2px;
  padding-top: calc(.625rem - 1px);
}
.kBuilder__inlineAddButton--vertical{
  width: 1.5rem;
  height: 100%;
  right: 100%;
}
.kBuilder__inlineAddButton--vertical.kBuilder__inlineAddButton--after{
  left: 100%;
  right: auto;
  top: 0;
}
.kBuilder__inlineAddButton--vertical::before{
  width: calc(1.5rem / 2 + 1px);
  height: 100%;
  border-right-width: 2px;
}

/* .blocklist-item {
  display: inline-block;
  margin-right: 10px;
} */
.blocklist-enter-active, .blocklist-leave-active {
  transition: all .5s;
}
.blocklist-enter, .blocklist-leave-to /* .list-leave-active below version 2.1.8 */ {
  opacity: 0;
  transform: translateY(-5%);
}

.kBuilder--col-1 .kBuilder__blocks{
  grid-row-gap: calc(.625rem * 2);
}

.kBuilder__column{
  position: relative;
}

.kBuilder__previewFrame{
  width: 100%;
  height: 30px;
  border: none;
  display: block;
}
.kBuilder__previewFrame--hidden{
  display: none;
}

.kBuilder__blockContent--hidden{
  display: none;
}

.sortable-ghost .kBuilder__previewFrame{
  pointer-events: none;
}
</style>