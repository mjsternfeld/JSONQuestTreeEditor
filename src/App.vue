<script setup lang="ts">
import { ref } from 'vue'
import type { Node, Edge, Connection, EdgeMouseEvent, NodeMouseEvent } from '@vue-flow/core'  
import { VueFlow, ConnectionMode, useVueFlow } from '@vue-flow/core'

// these components are only shown as examples of how to use a custom node or edge
// you can find many examples of how to create these custom components in the examples page of the docs
import SpecialNode from './components/SpecialNode.vue'
import SpecialEdge from './components/SpecialEdge.vue'
import MutexEdge from './components/MutexEdge.vue'

const {onEdgeClick, onNodeClick} = useVueFlow() //event for selecting edges to delete them

//selecting node for the deletenode dialog
let selectedNode = ref<string>('');
onNodeClick((param:NodeMouseEvent) => {
  console.log("selected node " + param.node.id);
  selectedNode.value = param.node.id;
})

const handleDeleteNode = (id: string) => {
  console.log("handleDeleteNode reached with id = " + id);
  nodes.value = nodes.value.filter(node => node.id != (id));
  selectedNode.value = '';
}

//selecting edge to display in the "delete edge" dialog
let selectedEdge = ref<string>('');
onEdgeClick((param:EdgeMouseEvent) => {
  // console.log('edge clicked', param.edge)
  selectedEdge.value = param.edge.id;
  // console.log("new selectedEdge value: " + selectedEdge);
})

const handleDeleteEdge = (id: string) => {
  console.log("handleDeleteEdge reached (in App.vue)");
  if(id.includes('special')){
    edges.value = edges.value.filter(edge => edge.id !== id)
  }else if(id.includes('mutex')){ //delete both mutex edges
    
    let edgeSource = id.split('-')[2];
    let edgeTarget = id.split('-')[3];
    
    let firstId = `edge-mutex-${edgeSource}-${edgeTarget}`;
    let secondId = `edge-mutex-${edgeTarget}-${edgeSource}`;

    console.log("first ID = " + firstId);
    console.log("second ID = " + secondId);
    

    edges.value = edges.value.filter(edge => edge.id != (firstId))
    edges.value = edges.value.filter(edge => edge.id != (secondId))
  }
  selectedEdge.value = '';
  
}

//nodes for vue flow
const nodes = ref<Node[]>([])

//edges for vue flow
const edges = ref<Edge[]>([])


//the data type of nodes in the output json
interface OutputNode {
  id: number;
  description: string;
  isRequired: boolean;
  isEndNode: boolean;
  successors: number[];
  mutuallyExclusiveWith: number[];
}

//importing a set of nodes from a json file
const handleImport = () => {

  const input = document.createElement('input');
  input.type = 'file';
  input.accept = 'application/json';
  input.onchange = async (event: Event) => {
    const file = (event.target as HTMLInputElement).files?.[0];
    if (file) {
      const text = await file.text();
      const json = JSON.parse(text);
      const outputNodes: OutputNode[] = json.objectives;

      //fetch nodes
      nodes.value = outputNodes.map(node => ({
        id: node.id.toString(),
        position: { x: Math.random() * 400, y: Math.random() * 400 },
        data: {
          description: node.description,
          isRequired: node.isRequired,
          isEndNode: node.isEndNode
        },
        type: 'special'
      }));

      //create normal and mutex edges
      edges.value = [];
      outputNodes.forEach(node => {
        node.successors.forEach(successor => {
          edges.value.push({
            id: `edge-${node.id}-${successor}`,
            source: node.id.toString(),
            target: successor.toString(),
            type: 'special'
          });
        });
        node.mutuallyExclusiveWith.forEach(mutex => { //two edges since mutex is symmetrical
          edges.value.push({
            id: `edge-mutex-${node.id}-${mutex}`,
            source: node.id.toString(),
            target: mutex.toString(),
            type: 'mutex',
            data: { label: 'mutually exclusive with' }
          });
          edges.value.push({
            id: `edge-mutex-${mutex}-${node.id}`,
            source: mutex.toString(),
            target: node.id.toString(),
            type: 'mutex',
            data: { label: 'mutually exclusive with' }
          });
        });
      });
    }
  };
  input.click();

}
//format current nodes and edges into a json file compatible with the game's quest system
const handleExport = () => {

  const outputNodes: OutputNode[] = nodes.value.map(node => ({
    id: parseInt(node.id),
    description: node.data.description || '',
    isRequired: node.data.isRequired,
    isEndNode: node.data.isEndNode,
    successors: edges.value
      .filter(edge => edge.type === 'special' && edge.source === node.id)
      .map(edge => parseInt(edge.target)),
    mutuallyExclusiveWith: edges.value //here we only need to consider outgoing mutex edges since we're setting the mutex array per-node
      .filter(edge => edge.type === 'mutex' && edge.source === node.id)
      .map(edge => parseInt(edge.target))
  }));

  console.log(outputNodes);
  const output = { objectives: outputNodes };
  const blob = new Blob([JSON.stringify(output, null, 2)], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'outputNodes.json';
  a.click();
  URL.revokeObjectURL(url);
  
}

const handleAddNode = () => {
  console.log('AddNode button clicked');  
  nodes.value.push({
    id: (nodes.value.length + 1).toString(),
    position: { x: Math.random() * 100, y: Math.random() * 100 },
    data: { description: `node description` },
    type: 'special'
  });
}

//delete all nodes and edges
const handleClear = () => {
  nodes.value = [];
  edges.value = [];
}


const handleConnect = (params: Connection) => {
  console.log('Connection made:', params);

  const sourceNode = nodes.value.find(node => node.id === params.source);
  const targetNode = nodes.value.find(node => node.id === params.target);

  console.log("sourceNode = " + sourceNode + ", targetNode = " + targetNode);

  if (sourceNode && targetNode) { //connect two existing nodes

    const sourceHandleType = params.sourceHandle?.includes('mutex') ? 'mutex' : 'normal';
    const targetHandleType = params.targetHandle?.includes('mutex') ? 'mutex' : 'normal';

    if ((sourceHandleType === 'mutex' && targetHandleType !== 'mutex') ||  //check if handle types match, don't create an edge if they don't
      (sourceHandleType !== 'mutex' && targetHandleType === 'mutex')) {
      console.log('Invalid connection: cannot connect mutex handle to normal handle');
      return;
    }

    const edgeType = (sourceHandleType === 'mutex' || targetHandleType === 'mutex') ? 'mutex' : 'special';

    console.log('sourceHandleType == ' + sourceHandleType + ', targetHandleType == ' + targetHandleType);

    if(edgeType === 'mutex'){ //create two edges since mutual-exclusivity is symmetrical
      edges.value.push({
        id: `edge-${edgeType}-${params.source}-${params.target}`,
        source: params.source,
        target: params.target,
        sourceHandle: `${params.source}-source-mutex`,
        targetHandle: `${params.target}-target-mutex`,
        type: edgeType,
        data: { label: 'mutually exclusive with' }
      });
      edges.value.push({
        id: `edge-${edgeType}-${params.target}-${params.source}`,
        target: params.source,
        source: params.target,
        sourceHandle: `${params.target}-source-mutex`,
        targetHandle: `${params.source}-target-mutex`,
        type: edgeType,
        data: { label: 'mutually exclusive with' }
      });
    }else if(edgeType === 'special'){ //just create one edge
      edges.value.push({
        id: `edge-${edgeType}-${params.source}-${params.target}`,
        source: params.source,
        target: params.target,
        sourceHandle: params.sourceHandle,
        targetHandle: params.targetHandle,
        type: edgeType
      });
    }
  }
}



</script>

<template>
  
  <div class="sidebar">
    <button class="sidebar-button" v-on:click="handleImport">Import</button>
    <hr>
    <button class="sidebar-button" v-on:click="handleExport">Export</button>
    <hr>
    <button class="sidebar-button" v-on:click="handleAddNode">Add Node</button>
    <hr>
    <button class="sidebar-button" v-on:click="handleClear">Clear</button>
    <hr style="margin-top: 50px; margin-bottom: 50px;">
    <div v-if="selectedEdge" class="edge-editor">
      <p>Selected Edge: {{ selectedEdge }}</p>
      <button @click="handleDeleteEdge(selectedEdge)">Delete Selected Edge</button>
    </div>
    <div v-if="selectedNode" class="edge-editor">
      <p>Selected Node: {{ selectedNode }}</p>
      <button @click="handleDeleteNode(selectedNode)">Delete Selected Node</button>
    </div>
  </div>

  <div class="flow-container">
    <VueFlow 
    :nodes="nodes" :edges="edges"
    @connect="handleConnect"
    @delete-edge="handleDeleteEdge"
    :connection-mode="ConnectionMode.Strict"> <!--only allow "source x target" edges-->
      <!-- bind your custom node type to a component by using slots, slot names are always `node-<type>` -->
      <template #node-special="specialNodeProps">
        <SpecialNode v-bind="specialNodeProps" />
      </template>
      <!-- bind your custom edge type to a component by using slots, slot names are always `edge-<type>` -->
      <template #edge-special="specialEdgeProps" @delete-edge="handleDeleteEdge">
        <SpecialEdge v-bind="specialEdgeProps" />
      </template>
      <template #edge-mutex="mutexEdgeProps" @delete-edge="handleDeleteEdge">
        <MutexEdge v-bind="mutexEdgeProps" />
      </template>
    </VueFlow>
  </div>

  

</template>

<style>
/* import the necessary styles for Vue Flow to work */
@import '@vue-flow/core/dist/style.css';

/* import the default theme, this is optional but generally recommended */
@import '@vue-flow/core/dist/theme-default.css';


.flow-container{
  width: 80%;
  height: 100%;
}


.sidebar{
  flex-direction: column; width:10vw; height: 100vh; background-color: gray;
}

.sidebar-button{
  width: 90%;
  padding: 10px;
  margin: 10px;
}

.edge-editor{
  width: 100%;
  display:flex;
  flex-direction: column;
  align-items: center;
}

</style>