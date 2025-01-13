<template>
  <nav class="navbar fluid" id="nav">
    <div class="container">
      <div class="navbar__brand">
        <a href="https://edux.com.br" class="navbar__logo">Neander JS</a>
      </div>
      <ul class="navbar__menu">
        <li class="navbar__item">
          <button
            class="navbar__button cant"
            id="step_by_step_button"
            data-canblock
            @click="_stepExec"
          >
            <i class="bi bi-arrow-right"></i> Executar Passo a Passo
          </button>
        </li>

        <li class="navbar__item">
          <button
            class="navbar__button cant"
            id="auto_step_button"
            data-canblock
            @click="_autoExec"
          >
            <i class="bi bi-play-fill"></i> Executar Automaticamente
          </button>
        </li>

        <li class="navbar__item">
          <div class="dropdown">
            <button
              class="btn btn-secondary dropdown-toggle"
              type="button"
              data-bs-toggle="dropdown"
              aria-expanded="false"
            >
              Limpar Memória
            </button>
            <ul class="dropdown-menu">
              <li>
                <button class="dropdown-item" @click="clearProgramMemory" type="button">
                  Limpar Memória de Programa
                </button>
              </li>
              <li>
                <button class="dropdown-item" @click="clearDataMemory" type="button">
                  Limpar Memória de Dados
                </button>
              </li>
              <li>
                <button class="dropdown-item" @click="clearMemory" type="button">
                  Limpar Toda a Memória
                </button>
              </li>
            </ul>
          </div>
        </li>
      </ul>
    </div>
  </nav>
  <nav class="navbar sub fluid" v-show="isSubNavVisivle">
    <div class="container">
      <ul class="navbar__menu" v-show="currentButton === 'step'">
        <li class="navbar__item">
          <button
            class="navbar__button cant"
            id="down-button"
            @click="_stepProgram(-1)"
          >
            <i class="bi bi-arrow-left"></i> Retroceder
          </button>
        </li>

        <li class="navbar__item">
          <button
            class="navbar__button"
            id="up-button"
            @click="_stepProgram(+1)"
          >
            <i class="bi bi-arrow-right"></i> Avançar
          </button>
        </li>
      </ul>

      <ul class="navbar__menu" v-show="currentButton === 'auto'">
        <li class="navbar__item input">
          <label for="time">Informe o tempo de cada passo (ms)</label>
          <input
            type="number"
            step="50"
            min="100"
            name="time"
            v-model="time_ref"
          />
        </li>
        <li class="navbar__item">
          <button class="navbar__button" @click="_autostepProgram">
            <i class="bi bi-play-fill"></i> Executar
          </button>
        </li>

        <li class="navbar__item">
          <button class="navbar__button active" @click="_pauseProgram">
            <i class="bi bi-pause-fill"></i> Pausar
          </button>
        </li>
      </ul>
    </div>
  </nav>
</template>

<script setup>
import { onMounted, ref } from "vue";

const time_ref = ref(1000);
const currentButton = ref("step");
const isSubNavVisivle = ref(false);
const emit = defineEmits(["stepUp", "stepDown", "startAutoExecute", "pauseAutoExecute", "clearProgramMemory", "clearDataMemory", "clearMemory"]);

const currentButtonMap = {
  step: "step_by_step_button",
  auto: "auto_step_button",
};


function _activeButton(event) {
  event.target
    .closest("button")
    .parentElement?.parentElement?.querySelector(".active")
    ?.classList.remove("active");
  const button = event.target.closest("button");
  button.classList.toggle("active");
}

function _stepExec(event) {
  _activeButton(event);
  currentButton.value = currentButton.value === "step" ? "" : "step";
}

function _autoExec(event) {
  _activeButton(event);
  currentButton.value = currentButton.value === "auto" ? "" : "auto";
}


function _autostepProgram(event) {
  _activeButton(event);
  emit("startAutoExecute", time_ref.value);
}

function _pauseProgram(event) {
    _activeButton(event);
    emit("pauseAutoExecute");

}

function _stepProgram(step) {
  if (step === 1) {
    emit("stepUp");
  } else {
    emit("stepDown");
  }
}


function enableCurrentButton(id){
    const button = document.getElementById(currentButtonMap[currentButton.value]);
    button.classList.add("active");
}

function disableNavButtons() {
  const buttons = document.querySelectorAll("[data-canblock]");
  buttons.forEach((button) => {
    button.classList.add("cant");
    button.classList.remove("active");

  });

  isSubNavVisivle.value = false;
}

function enableNavButtons() {
  const buttons = document.querySelectorAll("[data-canblock]");
  buttons.forEach((button) => {
    button.classList.remove("cant");
  });

  enableCurrentButton();

  isSubNavVisivle.value = true;
}


function clearProgramMemory() {
  emit("clearProgramMemory");
}

function clearDataMemory() {
  emit("clearDataMemory");
}

function clearMemory() {
  emit("clearMemory");
}




onMounted(() => {
  // Initialize tooltips
  const tooltips = document.querySelectorAll('[data-toggle="tooltip"]');
  tooltips.forEach((tooltip) => {
    new bootstrap.Tooltip(tooltip, {
      placement: tooltip.getAttribute("data-placement") || "top",
    });
  });

 // create neander window object
  window.neander = {
    disableNavButtons,
    enableNavButtons
  };   
});


</script>

<style lang="scss" scoped>
.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 0;
  background-color: var(--panel-color);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);

  &.sub {
    position: fixed;
    bottom: 0vh;
    width: 100%;
    border-top: 1px solid var(--sub-panel-color);
    background-color: var(--panel-color);
    height: 100px;
    z-index: 1000;

    &.active {
      display: flex;
    }

    button {
      background-color: var(--sub-panel-color);
      height: fit-content;
    }

    .navbar__menu {
      height: 100%;
    }

    .navbar__item {
      height: 100%;
      display: flex;
      justify-content: center;
      align-items: center;
    }

    .container {
      height: 100%;
      justify-content: center;
      align-items: center;
    }
  }

  ul {
    padding: 0;
    margin: 0;
  }

  .navbar__brand {
    font-size: 1.5rem;
  }

  .navbar__item.input {
    color: var(--text-color);
    display: flex;
    gap: 1rem;
    align-items: flex-start;
    justify-content: center;
    flex-direction: column;

    input {
      width: 100%;
      outline: none;
    }
  }

  .navbar__logo {
    color: var(--text-color);
    text-decoration: none;
    font-weight: bold;
  }

  .navbar__menu {
    display: flex;
    list-style: none;
    gap: 1.5rem;
  }

  .dropdown {
    height: 100%;
  }

  .navbar__button,
  .dropdown-toggle {
    height: 100%;
    background-color: transparent;
    color: var(--text-color);
    border: none;
    border-radius: 0.25rem;
    padding: 0.5rem 1rem;
    font-size: 1rem;
    cursor: pointer;
    transition: background-color 0.3s;
    display: flex;
    align-items: center;
    gap: 0.5rem;

    &:hover {
      background-color: var(--button-hover);
    }
  }

  .navbar__button i {
    font-size: 1.2rem;
  }
}

button.active {
  background-color: var(--button-color) !important;
}

.cant {
  pointer-events: none;
  opacity: 0.3;
}


@media screen and (max-width: 1200px) {
  .navbar__brand {
    margin: 0 auto
  }

  .navbar{
    height: 150px;
    margin-bottom: 120px;
  }
  .navbar.sub{
    top: 150px;
    position: absolute;
    z-index: 0;
    margin-top: .5rem;
  }
    
}

@media screen and (max-width: 535px){
    .navbar{
        height: fit-content;
        margin-bottom: 0;
        .container{   
            justify-content: center;
            align-items: center;
        }
        .navbar__item{
            display: flex;
            justify-content: center;
            align-items: center;
           
        }
        .navbar__brand{
            width: 100%;
            text-align: center;
            margin-bottom: 1rem
        }
    }

    .navbar.sub{
        top:0;

        position: relative;
        z-index: 0;
        height: fit-content;   
      }

    span, p, li, button{   
      font-size: .8rem !important;
    }
    .navbar__menu{
        display: flex;
        flex-direction: column;
    }

}
</style>
