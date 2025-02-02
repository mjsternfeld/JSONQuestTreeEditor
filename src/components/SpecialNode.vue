<script setup lang="ts">
import { computed } from 'vue'
import { Position, Handle } from '@vue-flow/core'
import type { NodeProps } from '@vue-flow/core'
  
const props = defineProps<NodeProps>()

const data = computed(() => props.data)


//fields relevant to the output json
const description = computed<string>(() => data.value.description)
const isRequired = computed<boolean>(() => data.value.isRequired)
const isEndNode = computed<boolean>(() => data.value.isEndNode)

const handleConnectable = () => {return true;}

</script>

<template>
  <div class="vue-flow__node-default">
    
    <div>
      <p>ID: {{ id }}</p>
      <p>Description: {{description}}</p>
    </div>

    <Handle :id="`${id}-target-normal`" class="handle-top" type="target" :position="Position.Top" :data="{handleType: 'normal'}" :connectable="handleConnectable"/>
    <!-- only allow output edges if this isn't an exit node -->
    <Handle v-if="!isEndNode" :id="`${id}-source-normal`" class="handle-bottom" type="source" :position="Position.Bottom" :data="{handleType: 'normal'}" :connectable="handleConnectable"/>
    

    <!--Handles for describing mutex relationships between quest nodes (objectives)-->
    <!--Mutex is not transitive! You have to connect ALL the nodes this node is mutually exclusive with-->
    <!--Mutex is however symmetrical. Therefore, creating a mutex edge from A to B automatically creates an additional edge from B to A-->
    <Handle class="handle-left" :id="`${id}-target-mutex`" type="target" :position="Position.Left" :data="{handleType: 'mutex'}" :connectable="handleConnectable"/>
    <Handle class="handle-right" :id="`${id}-source-mutex`" type="source" :position="Position.Left" :data="{handleType: 'mutex'}" :connectable="handleConnectable"/> 


  
  </div>
</template>