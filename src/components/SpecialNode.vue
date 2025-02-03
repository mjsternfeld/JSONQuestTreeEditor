<script setup lang="ts">
import { computed } from 'vue'
import { Position, Handle } from '@vue-flow/core'
import type { NodeProps } from '@vue-flow/core'

import { ref } from 'vue'

//fields relevant to the output json
const description = ref<string>('')
const isRequired = ref<boolean>(false)
const isEndNode = ref<boolean>(false)

const handleConnectable = () => {return true;}

const isEditing = ref(false)

const toggleEditing = () => {
  if (!isEditing.value) {
    // Start editing, copy the current data to local fields
    description.value = data.value.description
    isRequired.value = data.value.isRequired
    isEndNode.value = data.value.isEndNode
  }
  isEditing.value = !isEditing.value
}

const saveChanges = () => {
  // Save changes to the actual data
  data.value.description = description.value
  data.value.isRequired = isRequired.value
  data.value.isEndNode = isEndNode.value
  isEditing.value = false
}

const discardChanges = () => {
  // Discard changes, reset local fields
  description.value = data.value.description
  isRequired.value = data.value.isRequired
  isEndNode.value = data.value.isEndNode
  isEditing.value = false
}

const updateDescription = (event: Event) => {
  const target = event.target as HTMLInputElement
  description.value = target.value
}

const updateIsEndNode = (event: Event) => {
  const target = event.target as HTMLInputElement
  isEndNode.value = target.checked
}

const updateIsRequired = (event: Event) => {
  const target = event.target as HTMLInputElement
  isRequired.value = target.checked
}

const props = defineProps<NodeProps>()

const data = computed(() => props.data)

const formattedDescription = computed(() => {
  return description.value.replace(/\n/g, '<br>')
})

</script>


<style scoped>
.vue-flow__node-default {
  cursor: pointer;
}

.editing-container {
  display: flex;
  flex-direction: column;
}

.editing-input {
  width: 100%;
  
  align-self: center;
  line-break: auto;
}

.handle-top{
  width: 10px;
  height: 10px;
  background-color: green !important;
  color: green !important;
}

.handle-left{
  width: 10px;
  height: 10px;
  background-color: yellow !important;
  color: green !important;
}

.handle-right{
  width: 10px;
  height: 10px;
  background-color: yellow !important;
  color: red !important;
}

.handle-bottom{
  width: 10px;
  height: 10px;
  background-color: red !important;
  color: red !important;
}
.button-row{
  display:flex;
  flex-direction: row;
}
</style>

<template>
  <div class="vue-flow__node-default" @click="toggleEditing">
    <div v-if="isEditing" class="editing-container">
      <textarea class="editing-input" v-model="description" @input="updateDescription" @click.stop></textarea>
      <label class="editing-input">
      <input type="checkbox" v-model="isEndNode" @change="updateIsEndNode" @click.stop/>
      Is End Node
      </label>
      <label class="editing-input">
      <input type="checkbox" v-model="isRequired" @change="updateIsRequired" @click.stop/>
      Is Required
      </label>
      <div class="button-row">
      <button @click.stop="saveChanges">Save changes</button>
      <button @click.stop="discardChanges">Discard changes</button>
      </div>      
    </div>
    <div v-else>
      <p>ID: {{ id }}</p>
      <hr>
      <p>Description: <br> <span v-html="formattedDescription"></span></p>
      <hr>
      <p>{{ isEndNode ? "Exit Node" : "Not an exit node" }}</p>
      <hr>
      <p>{{ isRequired ? "Required objective" : "Optional objective" }}</p>
    </div>

    <!-- only allow output edges if this isn't an exit node -->
    <Handle :id="`${id}-target-normal`" class="handle-top" type="target" :position="Position.Top" :data="{handleType: 'normal'}" :connectable="handleConnectable" />
    <Handle v-if="!isEndNode" :id="`${id}-source-normal`" class="handle-bottom" type="source" :position="Position.Bottom" :data="{handleType: 'normal'}" :connectable="handleConnectable" />
    
    <!--Handles for describing mutex relationships between quest nodes (objectives)-->
    <!--Mutex is not transitive! You have to connect ALL the nodes this node is mutually exclusive with-->
    <!--Mutex is however symmetrical. Therefore, creating a mutex edge from A to B automatically creates an additional edge from B to A-->
    <Handle class="handle-left" :id="`${id}-target-mutex`" type="target" :position="Position.Left" :data="{handleType: 'mutex'}" :connectable="handleConnectable" />
    <Handle class="handle-right" :id="`${id}-source-mutex`" type="source" :position="Position.Left" :data="{handleType: 'mutex'}" :connectable="handleConnectable" />
  </div>
</template>

