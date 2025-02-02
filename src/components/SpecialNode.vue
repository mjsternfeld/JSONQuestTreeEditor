<script setup lang="ts">
import { computed } from 'vue'
import { Position, Handle } from '@vue-flow/core'
import type { NodeProps } from '@vue-flow/core'
  
const props = defineProps<NodeProps>()

const data = computed(() => props.data)


//fields relevant to the output json
const description = computed<string>(() => data.value.description)
const isRequired = computed<boolean>(() => data.value.isRequired)


</script>

<template>
  <div class="vue-flow__node-default">
    
    <div>
      <p>ID: {{ id }}</p>
      <p>Description: {{description}}</p>
    </div>


    <Handle class="handle-top" type="target" :position="Position.Top"/>

    <!--Handles for describing mutex relationships between quest nodes (objectives)-->
    <Handle class="handle-left" type="target" :position="Position.Left"/>
    <Handle class="handle-right" type="source" :position="Position.Right"/> 

    <!-- only allow output edges if this isn't an exit node -->
    <Handle v-if="!isRequired" class="handle-bottom" type="source" :position="Position.Bottom" />

  
  </div>
</template>