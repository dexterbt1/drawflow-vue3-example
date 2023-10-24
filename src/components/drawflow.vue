<template>

<el-container>
  <el-header class="header">
      <h3>Drawflow Example vue3</h3>
      <el-button type="primary"   @click="exportEditor">Export</el-button>
  </el-header>
  <el-container class="container">
    <el-aside width="250px" class="column">
        <ul>
            <li v-for="n in listNodes" :key="n" draggable="true" :data-node="n.item" @dragstart="drag($event)" class="drag-drawflow" >
                <div class="node" :style="`background: ${n.color}`" >{{ n.name }}</div>
            </li>
        </ul>
    </el-aside>
    <el-main>
        <div id="drawflow" @drop="drop($event)" @dragover="allowDrop($event)"></div>
        <div id="canvas-toolbar">
          <el-button @click="zoomIn">+</el-button>
          <el-button @click="zoomOut">-</el-button>
        </div>
    </el-main>
  </el-container>
</el-container>
<el-dialog
    v-model="dialogVisible"
    title="Export"
    width="50%"
  >
    <span>Data:</span>
    <pre><code>{{dialogData}}</code></pre>
    <template #footer>
      <span class="dialog-footer">
        <el-button @click="dialogVisible = false">Cancel</el-button>
        <el-button type="primary" @click="dialogVisible = false"
          >Confirm</el-button
        >
      </span>
    </template>
  </el-dialog>
</template>

<script>

import debounce from "lodash/debounce"

import Drawflow from 'drawflow'
import styleDrawflow from 'drawflow/dist/drawflow.min.css'
import style from '../assets/style.css' 
import { onMounted, shallowRef, h, getCurrentInstance, render, readonly, ref } from 'vue'

import ApiCallNode from './nodes/ApiCallNode.vue'
import ScriptNode from './nodes/ScriptNode.vue'
import ConsoleLogNode from './nodes/ConsoleLogNode.vue'


export default {
  name: 'drawflow',
  setup() {
   const listNodes = readonly([
        {
            name: 'Get/Post',
            color: '#49494970',
            item: 'ApiCallNode',
            input:0,
            output:1
        },
        {
            name: 'Script',
            color: 'blue',
            item: 'ScriptNode',
            input:1,
            output:2
        },
         {
            name: 'console.log',
            color: '#ff9900',
            item: 'ConsoleLogNode',
            input:1,
            output:0
        },
    ])
   
   const editor = shallowRef({})
   const dialogVisible = ref(false)
   const dialogData = ref({})
   const Vue = { version: 3, h, render };
   const internalInstance = getCurrentInstance()
   internalInstance.appContext.app._context.config.globalProperties.$df = editor;
   
    function exportEditor() {
      dialogData.value = editor.value.export();
      dialogVisible.value = true;
    }

    const drag = (ev) => {
      if (ev.type === "touchstart") {
        mobile_item_selec = ev.target.closest(".drag-drawflow").getAttribute('data-node');
      } else {
      ev.dataTransfer.setData("node", ev.target.getAttribute('data-node'));
      }
    }
    const drop = (ev) => {
      if (ev.type === "touchend") {
        var parentdrawflow = document.elementFromPoint( mobile_last_move.touches[0].clientX, mobile_last_move.touches[0].clientY).closest("#drawflow");
        if(parentdrawflow != null) {
          addNodeToDrawFlow(mobile_item_selec, mobile_last_move.touches[0].clientX, mobile_last_move.touches[0].clientY);
        }
        mobile_item_selec = '';
      } else {
        ev.preventDefault();
        var data = ev.dataTransfer.getData("node");
        addNodeToDrawFlow(data, ev.clientX, ev.clientY);
      }

    }
    const allowDrop = (ev) => {
      ev.preventDefault();
    }

    const panzoomDebug = (ed, ev) => {
      console.log('panzoomDebug', [ed.canvas_x, ed.canvas_y], ed.zoom);
    }

    const zoomIn = () => {
      editor.value.zoom_in();
    }

    const zoomOut = () => {
      editor.value.zoom_out();
    }


   let mobile_item_selec = '';
   let mobile_last_move = null;
   function positionMobile(ev) {
     mobile_last_move = ev;
   }

    function addNodeToDrawFlow(name, pos_x, pos_y) {
      pos_x = pos_x * ( editor.value.precanvas.clientWidth / (editor.value.precanvas.clientWidth * editor.value.zoom)) - (editor.value.precanvas.getBoundingClientRect().x * ( editor.value.precanvas.clientWidth / (editor.value.precanvas.clientWidth * editor.value.zoom)));
      pos_y = pos_y * ( editor.value.precanvas.clientHeight / (editor.value.precanvas.clientHeight * editor.value.zoom)) - (editor.value.precanvas.getBoundingClientRect().y * ( editor.value.precanvas.clientHeight / (editor.value.precanvas.clientHeight * editor.value.zoom)));
    
      const nodeSelected = listNodes.find(ele => ele.item == name);
      editor.value.addNode(name, nodeSelected.input,  nodeSelected.output, pos_x, pos_y, name, {}, name, 'vue');
      
    }


   onMounted(() => {

      var elements = document.getElementsByClassName('drag-drawflow');
      for (var i = 0; i < elements.length; i++) {
        elements[i].addEventListener('touchend', drop, false);
        elements[i].addEventListener('touchmove', positionMobile, false);
        elements[i].addEventListener('touchstart', drag, false );
      }
        
       const id = document.getElementById("drawflow");
       editor.value = new Drawflow(id, Vue, internalInstance.appContext.app._context);

       // NOTE: monkey patch, taken from https://github.com/jerosoler/Drawflow/issues/88
       editor.value.translate_to = function(x, y, zoom){
         this.canvas_x = x;
         this.canvas_y = y;
         let storedZoom = zoom;
         this.zoom = 1;
         this.precanvas.style.transform = "translate("+this.canvas_x+"px, "+this.canvas_y+"px) scale("+this.zoom+")";
         this.zoom = storedZoom;
         this.zoom_last_value = 1;
         this.zoom_refresh();
       }

       editor.value.start();
       
       editor.value.registerNode('ApiCallNode', ApiCallNode, {}, {});
       editor.value.registerNode('ScriptNode', ScriptNode, {}, {});
       editor.value.registerNode('ConsoleLogNode', ConsoleLogNode, {}, {});

       editor.value.on('zoom', (ev) => { panzoomDebug(editor.value, ev) });
       editor.value.on('translate', debounce((ev) => { panzoomDebug(editor.value, ev) }, 500));

       //editor.value.import({"drawflow":{"Home":{"data":{"5":{"id":5,"name":"ScriptNode","data":{"script":"(req,res) => {\n console.log(req);\n}"},"class":"ScriptNode","html":"ScriptNode","typenode":"vue","inputs":{"input_1":{"connections":[{"node":"6","input":"output_1"}]}},"outputs":{"output_1":{"connections":[]},"output_2":{"connections":[]}},"pos_x":1000,"pos_y":117},"6":{"id":6,"name":"ApiCallNode","data":{"url":"localhost/add", "method": "post"},"class":"ApiACallNode","html":"ApiCallNode","typenode":"vue","inputs":{},"outputs":{"output_1":{"connections":[{"node":"5","output":"input_1"}]}},"pos_x":137,"pos_y":89}}}}})

      var initialBP = {
        "drawflow": {
          "Home": {
            "data": {
              "5": {
                "id": 5,
                "name": "ScriptNode",
                "data": {
                  "script": "(req,res) => {\n console.log(req);\n}"
                },
                "class": "ScriptNode",
                "html": "ScriptNode",
                "typenode": "vue",
                "inputs": {
                  "input_1": {
                    "connections": [
                      {
                        "node": "6",
                        "input": "output_1"
                      }
                    ]
                  }
                },
                "outputs": {
                  "output_1": {
                    "connections": [
                      {
                        "node": "7",
                        "output": "input_1"
                      }
                    ]
                  },
                  "output_2": {
                    "connections": []
                  }
                },
                "pos_x": 396,
                "pos_y": 304
              },
              "6": {
                "id": 6,
                "name": "ApiCallNode",
                "data": {
                  "url": "localhost/add",
                  "method": "post"
                },
                "class": "ApiACallNode",
                "html": "ApiCallNode",
                "typenode": "vue",
                "inputs": {},
                "outputs": {
                  "output_1": {
                    "connections": [
                      {
                        "node": "5",
                        "output": "input_1"
                      }
                    ]
                  }
                },
                "pos_x": -85,
                "pos_y": 122
              },
              "7": {
                "id": 7,
                "name": "ConsoleLogNode",
                "data": {},
                "class": "ConsoleLogNode",
                "html": "ConsoleLogNode",
                "typenode": "vue",
                "inputs": {
                  "input_1": {
                    "connections": [
                      {
                        "node": "5",
                        "input": "output_1"
                      }
                    ]
                  }
                },
                "outputs": {},
                "pos_x": 781.5714285714286,
                "pos_y": 171
              }
            }
          }
        }
      }
      editor.value.import(initialBP);
      editor.value.translate_to(23, -40, 0.7);
  })

  return {
    exportEditor, listNodes, drag, drop, allowDrop, dialogVisible, dialogData, zoomIn, zoomOut
  }

  } // setup
}

</script>
<style scoped>
.header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 1px solid #494949;
}
.container {
    min-height: calc(100vh - 100px);
}
.column {
    border-right: 1px solid #494949;
}
.column ul {
    padding-inline-start: 0px;
    padding: 10px 10px;
 
}
.column li {
    background: transparent;
}

.node {
    border-radius: 8px;
    border: 2px solid #494949;
    display: block;
    height:60px;
    line-height:40px;
    padding: 10px;
    margin: 10px 0px;
    cursor: move;

}
#drawflow {
  width: 100%;
  height: 100%;
  text-align: initial;
  background: #2b2c30;
  background-size: 20px 20px;
  background-image: radial-gradient(#494949 1px, transparent 1px);
}

.el-main {
  position: relative;
}

#canvas-toolbar {
  position: absolute;
  bottom: 1rem;
  right: 1rem;
}
#canvas-toolbar button {
  padding: 0px 16px 0 14px;
}
</style>