<template>

  <Navbar @startAutoExecute="run" @pauseAutoExecute="stop"
    @clearProgramMemory="clearProgramMemory" @clearDataMemory="clearDataMemory" @clearMemory="clearMemory" />
  <div class="panel prevent-select py-4 px-4">

    <div class="data_columns">

      <div class="data-col" id="program-memory">

        <div class="data-holder">
          <span class="data_title">Memória de Programa</span>
          <div class="data_header">

            <span>Endereço</span>
            <span>Valor</span>
          </div>
          <ul class="data" ref="data_ref">
            <li>
              <div class="cell"></div>
              <input class="value" />
            </li>
          </ul>

        </div>


      </div>

      <div class="data-col">

        <div class="data-holder">
          <span class="data_title">Memória de Dados</span>
          <div class="data_header">
            <span>Endereço</span>
            <span>Valor</span>
          </div>
          <ul class="data" ref="data_ref_mem">
            <li>
              <div class="cell"></div>
              <input class="value" />
            </li>
          </ul>
        </div>


      </div>

    </div>

    <div class="editor_col">
      <div class="editor">
        <TextEditor ref="editor_ref"></TextEditor>
      </div>


    </div>

    <div class="actions_col">

      <div class=" visor">
        <div class="visor__holder">
          <p>AC</p>
          <p>{{ acc }}</p>
        </div>
        <div class="visor__holder">
          <p>PC</p>
          <p>{{ pc }}</p>
        </div>
        <div class="visor__holder">
          <p>N</p>
          <p>{{ n }}</p>
        </div>
        <div class="visor__holder">
          <p>Z</p>
          <p>{{ z }}</p>
        </div>

        <div class="visor__holder">
          <p>Instruções</p>
          <p>{{ inst }}</p>
        </div>

      </div>

      <div class="actions-panel">
        <button class="load" @click="mount">
          Carregar Programa
        </button>

        <button
          id="step-back"
          class="step-back cant"
          @click="buttonStep(-1)"
        >
          <i class="bi bi-arrow-left"></i> Retroceder
        </button>

        <button
          id="step-front"
          class="step-front"
          @click="buttonStep(+1)"
        >
          <i class="bi bi-arrow-right"></i> Avançar
        </button>
      
       </div>
      
    </div>






  </div>
</template>

<script setup>
import TextEditor from "./TextEditor.vue";
import { computed, nextTick, onMounted, ref } from "vue";
import Manager from "../utils/logicProcessor";
import Navbar from "./Navbar.vue";
import { parse } from "vue/compiler-sfc";

const data_ref = ref(null);
const data_ref_mem = ref(null);
const editor_ref = ref(null);
const acc_value = ref("0");
const instruction_time_ref = ref("1");
const pc_value = ref("0");
const n_value = ref("0");
const z_value = ref("1");
const inst_value = ref("0");
const instruction_mode = ref("pp");
const end_value = ref(false);
const is_running = ref(false);
const current_interval = ref(null);
const is_compiled = ref(false);
let manager;
let lastTarget;

const acc = computed(() => ("" + acc_value.value).padStart(3, "0"));
const pc = computed(() => ("" + pc_value.value).padStart(3, "0"));
const n = computed(() => "" + n_value.value);
const z = computed(() => "" + z_value.value);
const inst = computed(() => "" + inst_value.value);

// end nav bar functions


function _setMemory() {
  manager.setMemory(
    Uint8Array.from(
      document.querySelectorAll("input.value"),
      (el) => el.value
    )
  );
}


function _generateDataCells() {
  let cellNumber = 0;
  for (let j = 0; j < 2; j++) {
    const current = j == 0 ? data_ref : data_ref_mem;
    let i = j === 0 ? 0 : 128;
    let condition = j === 0 ? 128 : 256;

    for (; i < condition; i++) {
      const clone = current.value.firstElementChild.cloneNode(true);
      const cell = clone.querySelector(".cell");
      const value = clone.querySelector(".value");
      cell.innerHTML = +i.toString().padStart(3, "0").toUpperCase();
      value.setAttribute("data-index", cellNumber);
      value.value = "000";

      if (j === 0 && i === 0) {
        cell.classList.add("indicator")
      }

      cellNumber++;
      current.value.append(clone);

    }
    current.value.firstElementChild.remove()
  }

}


function _updateStepButtons() {
  if (inst_value.value >= 1) {
    document.getElementById("step-back").classList.remove("cant")
  }
  else {
    document.getElementById("step-back").classList.add("cant")
  }

  if (end_value.value) {
    document.getElementById("step-front").classList.add("cant")
  }
  else {
    document.getElementById("step-front").classList.remove("cant")
  }
}



function _changeData(memory) {
  const element = document.querySelectorAll("input.value");
  for (let j = 0; j < memory.length; j++) {
    element[j].value = String(memory[j]).padStart(3, "0");
  }
}

function _parse(str) {
  let instruction_map = new Map([
    ["nop", "00"],
    ["sta", "01"],
    ["lda", "02"],
    ["add", "03"],
    ["or", "04"],
    ["and", "05"],
    ["not", "06"],
    ["jmp", "08"],
    ["jn", "09"],
    ["jz", "10"],
    ["hlt", "15"],
    ["var", "98"],
    ["ads", "99"]

  ]);

  let instruction_map_reversed = new Map([
    ["00","nop"],
    ["01","sta"],
    ["02","lda"],
    ["03","add"],
    ["04","or"],
    ["05","and"],
    ["06","not"],
    ["08","jmp"],
    ["09","jn"],
    ["10","jz"],
    ["15","hlt"],
    ["98","var"],
    ["99","ads"]

  ]);

  let address_map = ['10', '09', '08', '01']
  
  let instruction_values = Array.from(instruction_map.values())
  let result = [];
  let addresses = {};
  let vars = {};
  let lastChar = "";
  let currentChar;
  let error = false;



  for (let index = 0; index < str.length; index++) {
    currentChar = str[index];
    if (!str[index + 1]) break;
    if (
      !(
        currentChar.charCodeAt() >= 0 &&
        currentChar.charCodeAt() <= 31 &&
        currentChar.charCodeAt() != 10
      )
    ) {
      if (currentChar.charCodeAt() == 10) currentChar = " ";
      if (!(currentChar == " " && lastChar == " ")) result.push(currentChar);
      lastChar = currentChar;
    }
  }
  if (str.length > 0 && str[str.length - 1] != " ")
    result.push(str[str.length - 1]);


  if (str.length > 0) {
    result = result
      .join("")
      .toLowerCase()
      .split(" ")
      .map((tk) => (instruction_map.get(tk) ? instruction_map.get(tk) : tk));
  }



  for (let index = 0; index < result.length; index++ ){
    const tk = result[index]
    const nextTk = result[index+1]
    const instruction = instruction_map_reversed.get(String(nextTk).padStart(2,"0"))
    let name = "ADS"
    if(tk == 99){

      if(instruction_values.includes(nextTk)){
        alert(`Atenção: ${name} não pode ser definido como a instrução reservada "${instruction}"  `)
        error = true
      }
      else if(nextTk == null){
        alert(`Atenção: ${name} precisa ser seguindo de um nome válido`)
        error = true
      }
      else{
          addresses[nextTk] = nextTk
          result[index] = "X"
          result[index+1] = "X"
      }
    
    }
    if(tk == 98){
      // var nome endereço
      const thirdTk = result[index+2]
      name = "VAR"

      if(instruction_values.includes(nextTk)){
        alert(`Atenção: ${name} não pode ter como nome uma instrução reservada "${instruction}"  `)
        error = true
      }

      else if(nextTk == null || !Number.isNaN(Number(nextTk))){
        alert(`Atenção: ${name} precisa ter um nome válido`)
        error = true
      }
      
      else if (instruction_values.includes(thirdTk )|| !thirdTk){
        alert(`Atenção: ${name} precisa ter um endereço válido`)
        error = true
      }

      else if(Number(thirdTk) >= 255 || Number(thirdTk) < 0 ){
        alert(`Atenção: ${name} só recebe valores entre 0 e 254`)
        error = true
      }
      

      else{
          vars[nextTk] = thirdTk
          result[index] = "X"
          result[index+1] = "X"
          result[index+2] = "X"
      }

      
    }
    if(error) break
  }
 
  for( let index = result.length-1; index >=0; index--){
    if(result[index] === "X") result.splice(index,1)
  }

  if(error) return false

  // tudo separado

  result =  result.map(tk => {

    if(vars[tk]) return vars[tk]
    return tk
  })


  let removed = 0
  result = result.map((tk, index) => {
    if(addresses[tk]) {
      if(!address_map.includes(result[index-1])){
        addresses[tk] = index - removed
        removed++
        return 'X'
      }
    }
    return tk
  })


  for( let index = result.length-1; index >=0; index--){
    if(result[index] === "X") result.splice(index,1)
  
  }


  result = result.map((tk) => {
    if(addresses[tk]) {
      return addresses[tk]
    }
    return tk
  })



  result.forEach((el, index) => {
    result[index] = parseInt(el);
    if (isNaN(result[index])) {
      alert(`Atenção: instrução inválida "${el}"`)
      error = true
    }
    else if (result[index] > 255) {
      result[index] = 255
    }
    else if (result[index] < 0) {
      result[index] = "000"
    }
    else {
      result[index] = el.toString().padStart(3, "0");
    }
  });

  if(error) return false
  nextTick(() => {
    data_ref.value.dispatchEvent(new Event("input"));
  })

  return result;
}


function _syncState(values) {
  acc_value.value = values.acc;
  pc_value.value = values.pc;
  n_value.value = values.n;
  z_value.value = values.z;
  inst_value.value = values.inst;
  end_value.value = values.end;
}



function _updateFromProcessor(values) {

  _syncState(values)
  _changeData(values.memory);
}

function _manageExecution() {
  current_interval.value = setInterval(
    () => step(1),
    parseFloat(instruction_time_ref.value)
  );
}

function _setExecutionTime(time) {
  instruction_time_ref.value = time;
}
function _programMemoryIsEmpty() {
  const htmlData = Array.from(document.querySelectorAll("input.value"));
  return htmlData.slice(0, 128).every(el => parseInt(el.value || 0) == 0)
}

function _updateVisualIndicator() {
  const cells = Array.from(document.querySelectorAll("#program-memory .cell"));

  cells.forEach((el) => {
    el.classList.remove("indicator");
  });
  const cell = cells[parseInt(pc_value.value)];
  cell.classList.add("indicator");
  cell.scrollIntoView({ behavior: "instant", block: "center", inline: "center" });

}


function step(direction) {
  _updateFromProcessor(manager.step(direction));
  _updateVisualIndicator();
  _updateStepButtons();

}

function buttonStep(direction){
  step(direction)

}


function mount() {
  const tokens = _parse(editor_ref.value.code.trim());
  const htmlData = Array.from(document.querySelectorAll("input.value"));
  if(!tokens) return
  if (tokens.length) {
    for (let index = 0; index < tokens.length; index++) {
      htmlData[index].value = tokens[index];
    }
  }

  _setMemory();

  is_compiled.value = true;
}

function clearProgramMemory() {
  // reset program memory
  stop();
  const htmlData = Array.from(document.querySelectorAll("input.value"));
  for (let index = 0; index < 128; index++) {
    htmlData[index].value = "000";
  }


  nextTick(() => {
    data_ref.value.dispatchEvent(new Event("input"));
  })


  manager.resetProgramMemoryAndRegisters();
  _syncState(manager.getInternalState());
  _updateVisualIndicator();
  _updateStepButtons();

}

function clearDataMemory() {
  const htmlData = Array.from(document.querySelectorAll("input.value"));
  for (let index = 128; index < 256; index++) {
    htmlData[index].value = "000";
  }


  _setMemory()
}

function clearMemory() {
  clearProgramMemory();
  clearDataMemory();
}


function run(time) {
  _setExecutionTime(time);
  _manageExecution();
}

function stop() {
  clearInterval(current_interval.value);
}

function dataValueClicked(event) {
  const target = event.target.closest(".value");
  if (target) {
    target.select();


  }
}

function dataValueEdited(event) {
  const target = event.target;
  const value = parseInt(target.value);
  if (value < 0) {
    target.value = "000";
  }
  else if (value > 255) {
    target.value = "255";
  }

  else if (!target.value) {
    target.value = "000"
  }
  else {
    target.value = value.toString().padStart(3, "0");
  }

  nextTick(() => {
    _setMemory();
  });





}
onMounted(() => {
  _generateDataCells();
  manager = new Manager();
  data_ref.value.addEventListener("click", dataValueClicked);
  data_ref_mem.value.addEventListener("click", dataValueClicked);

  data_ref.value.addEventListener("input", dataValueEdited);
  data_ref_mem.value.addEventListener("input", dataValueEdited);



});
</script>

<style>
  
.cm-scroller{
    scrollbar-color: #4e4f52 #31353d;
  }
</style>
<style lang="scss" scoped>
.panel {
  // background-color: red;
  width: 100%;
  height: calc(100vh - 60px);
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 1rem;
  overflow: hidden;
  
}


.editor_col {
  box-sizing: border-box !important;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  justify-content: flex-start;
  height: 100%;
  max-height: calc(100vh - 108px );


  .editor {

    width: 100%;
    height: 100%;
    overflow-x: hidden;
    overflow-y: auto;
    grid-column: 1/3;
    grid-row: 2/4;
    border: 1px solid var(--panel-color);
  
  }
}


.data_columns {
  display: flex;
  flex-direction: row;
  align-items: flex-start;
  justify-content: center;
  overflow-x: hidden;
  overflow-y: hidden;
  width: 100%;
  height: 100%;
  grid-column: 1;
  grid-row: 1/1;
  gap: 1rem;

  .data-col {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    width: 100%;
    height: 100%;
    overflow-y: auto;
  }


  .data-holder {
    height: 100%;
    display: flex;
    flex-direction: column;
    align-items: center;

  }

  .data_title {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;
    width: 100%;
    height: 40px;
    background-color: var(--button-color);
    color: var(--text-color);
    font-size: 1rem;
    border-bottom: 1px dashed var(--border-color);
  }

  .data_header {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: center;
    flex-wrap: wrap;
    width: 100%;
    height: 40px;
    background-color: var(--button-color);

    span {
      width: 50%;
      text-align: center;
      font-size: 1rem;
      color: var(--text-color);

      &:first-child {
        width: 45%;
      }

      &:last-child {
        border-left: 1px dashed var(--border-color);
      }
    }

  }

  .data {
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
    height: 100%;
    margin: 0;
    width: fit-content;
    overflow-y: auto;
    background-color: var(--panel-color);
    color: var(--primary-color);
    padding: 0;
    scrollbar-color: var(--border-color) var(--panel-color);

    li {
      display: flex;
      flex-direction: row;
      align-items: center;
      justify-content: flex-start;
      width: 100%;
      height: 2rem;
      border-bottom: 1px dashed var(--border-color);
      font-size: 1rem;

      &:last-child {
        border-bottom: none;
      }


      .cell {
        display: flex;
        align-items: center;
        justify-content: center;
        width: 50%;
        height: 100%;
        font-size: 1rem;
     

        &.indicator{
          background-color: rgba(0,0,0,0.5);
          position: relative;

        }
      }

      .spacer {
        width: 1rem;
      }

      .value {
        border: none;
        border-left: 1px dashed var(--border-color);
        width: 50%;
        text-align: center;
        background-color: transparent;
        outline: none;
        color: white;

        cursor: pointer;
      }
    }

  }

}


.visor {
  display: flex;
  padding: .5rem;
  background-color: var(--panel-color);
  // border-radius: .25rem;
  height: 100px;

  .visor__holder {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    width: 100%;
    height: 100%;
    color: var(--text-color);
    padding: 1rem;

    font-size: 1rem;
    border-right: 2px dashed var(--border-color);

    &:last-child {
      border-right: none;
    }


    p:first-child {
      margin-bottom: .5rem;
    }
  }
}


.actions-panel{
  height: calc(100% - 100px);
  display: grid;
  padding: 1rem 0 0 0;
  grid-template-rows: repeat(12, 1fr);
  grid-template-columns: 1fr 1fr 1fr ;



  .load{
    grid-column: 1/4;
    grid-row: 1;
  }


  .step-back{
    grid-column: 1;
    grid-row: 12;
   
  }


  .step-front{
    grid-column: 3;
    grid-row: 12;
   
  }
  
 
}


button {
  width: 100%;
  // margin-bottom: 1rem;
  background-color: var(--button-color);
  color: var(--text-color);
  padding: .5rem 1rem;
  border: none;
  border-radius: .25rem;
  cursor: pointer;
  outline: none;
  font-size: 1rem;

  &.cant{
    opacity: .5;
    pointer-events: none;
  }
}
</style>
