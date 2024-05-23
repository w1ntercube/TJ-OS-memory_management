<template>
  <div class="memory-management">
    <div class="controls">
      <div class="instruction-count-container">
        <label for="instructionCount">指令数:</label>
        <input
          type="number"
          id="instructionCount"
          v-model.number="totalInstructions"
          @input="validateInstructionCount"
          min="1"
        />
        <button
          @click="openCustomInstructionInput"
          :disabled="instructionsComplete"
          class="buttons"
        >
          自定义指令
        </button>
      </div>

      <div v-if="instructionCountError" class="error-message">
        <p style="color: red">指令数必须为正整数。</p>
      </div>

      <div class="algorithm-buttons">
        <button
          @click="selectAlgorithm('FIFO')"
          :disabled="
            selectedAlgorithm || instructionsComplete || instructionCountError
          "
          class="buttons"
        >
          FIFO 算法
        </button>
        <button
          @click="selectAlgorithm('LRU')"
          :disabled="
            selectedAlgorithm || instructionsComplete || instructionCountError
          "
          class="buttons"
        >
          LRU 算法
        </button>
      </div>

      <div class="execution-buttons">
        <button
          @click="executePrevInstruction"
          :disabled="
            !selectedAlgorithm ||
            instructionsComplete ||
            currentInstructionIndex === 0
          "
          class="buttons"
        >
          单步回退
        </button>
        <button
          @click="executeNextInstruction"
          :disabled="
            !selectedAlgorithm ||
            instructionsComplete ||
            instructionCountError ||
            executionSpeedError
          "
          class="buttons"
        >
          单步执行
        </button>
        <button
          @click="startAutoExecution"
          :disabled="
            !selectedAlgorithm ||
            instructionsComplete ||
            autoExecutionInterval ||
            instructionCountError ||
            executionSpeedError
          "
          class="buttons"
        >
          自动执行
        </button>
        <button
          @click="pauseAutoExecution"
          :disabled="instructionsComplete || !autoExecutionInterval"
          class="buttons"
        >
          暂停
        </button>

        <div class="execution-speed-container">
          <label for="executionSpeed">执行速度（每秒）:</label>
          <input
            type="number"
            id="executionSpeed"
            v-model.number="executionSpeed"
            @input="validateExecutionSpeed"
            min="0.1"
          />
        </div>

        <div v-if="executionSpeedError" class="error-message">
          <p style="color: red">执行速度必须为正数且大于等于 0.1。</p>
        </div>
      </div>

      <button @click="resetSimulation" class="buttons reset-button">
        重置
      </button>
    </div>

    <table class="memory-table">
      <thead>
        <tr>
          <th>内存块编号</th>
          <th v-for="(block, index) in memoryBlocks" :key="'header' + index">
            {{ index }}
          </th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>存储的指令页号</td>
          <td v-for="(block, index) in memory" :key="'block' + index">
            {{ block !== null ? block : "空" }}
          </td>
          <td v-for="index in memoryBlocks.length" :key="'空' + index">空</td>
        </tr>
      </tbody>
    </table>

    <table class="instruction-table">
      <thead>
        <tr>
          <th>指令编号</th>
          <th>当前要执行的指令</th>
          <th>指令页号</th>
          <th>是否缺页</th>
          <th>当前缺页率</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="row in visibleInstructions" :key="row.index">
          <td>{{ row.index }}</td>
          <td>{{ row.instruction }}</td>
          <td>{{ row.pageNumber }}</td>
          <td>{{ row.pageFault }}</td>
          <td>{{ row.pageFaultRate }}%</td>
        </tr>
      </tbody>
    </table>

    <div v-if="instructionsComplete" class="completion-message">
      <p>执行完最后一条指令后，缺页率为：{{ pageFaultRate.toFixed(3) }}%。</p>
    </div>

    <div class="page-turner">
      <button @click="prevPage" :disabled="currentPage === 1" class="buttons">
        上一页
      </button>
      <span>第{{ currentPage }} / {{ totalPages }}页</span>
      <button
        @click="nextPage"
        :disabled="currentPage === totalPages"
        class="buttons"
      >
        下一页
      </button>
    </div>

    <div v-if="showCustomInstructionInput" class="modal">
      <div class="modal-content">
        <span class="close" @click="closeCustomInstructionInput">&times;</span>
        <h2>输入自定义指令</h2>
        <textarea
          v-model="customInstructions"
          placeholder="指令用空格分隔。"
        ></textarea>
        <div v-if="inputError" class="error-message">
          <p style="color: red">非法输入，请用空格分隔指令。</p>
        </div>
        <div>
          <button @click="applyCustomInstructions" class="buttons">使用</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      totalInstructions: 320,
      pageSize: 10,
      memoryBlocks: 4,
      instructionCountError: false,
      executionSpeedError: false,
      memory: [null, null, null, null],
      memoryHistory: [],
      pageFaults: 0,
      pageFaultRate: 0,
      instructionSequence: [],
      fifoQueue: [],
      lruQueue: [],
      currentInstruction: null,
      currentInstructionIndex: 0,
      selectedAlgorithm: null,
      autoExecutionInterval: null,
      executionSpeed: 20,
      instructionsComplete: false,
      showCustomInstructionInput: false,
      customInstructions: "",
      inputError: false,
      instructionTable: [],
      currentPage: 1,
    };
  },
  computed: {
    totalPages() {
      return Math.ceil(this.instructionTable.length / this.pageSize);
    },
    visibleInstructions() {
      const start = (this.currentPage - 1) * this.pageSize;
      const end = start + this.pageSize;
      return this.instructionTable.slice(start, end);
    },
  },
  methods: {
    validateInstructionCount() {
      if (
        this.totalInstructions < 1 ||
        !Number.isInteger(this.totalInstructions)
      ) {
        this.instructionCountError = true;
      } else {
        this.instructionCountError = false;
      }
    },
    validateExecutionSpeed() {
      if (this.executionSpeed < 0.1 || !Number.isFinite(this.executionSpeed)) {
        this.executionSpeedError = true;
      } else {
        this.executionSpeedError = false;
      }
    },
    openCustomInstructionInput() {
      this.showCustomInstructionInput = true;
    },
    closeCustomInstructionInput() {
      this.showCustomInstructionInput = false;
      this.inputError = false;
      this.customInstructions = "";
    },
    applyCustomInstructions() {
      const trimmedInput = this.customInstructions.trim();
      if (trimmedInput === "") {
        // 输入为空或者只包含空格
        this.inputError = true;
        return;
      }
      const instructions = this.customInstructions
        .trim()
        .split(/\s+/)
        .map(Number);
      const valid = instructions.every(Number.isInteger);
      if (valid && instructions.length > 0) {
        this.instructionSequence = instructions;
        this.totalInstructions = instructions.length;
        this.currentInstructionIndex = 0;
        this.currentInstruction =
          this.instructionSequence[this.currentInstructionIndex];
        this.showCustomInstructionInput = false;
        this.inputError = false; // 清除错误状态
        this.initializeSimulation();
      } else {
        // 显示错误提示但不清空输入框
        this.inputError = true;
      }
    },
    selectAlgorithm(algorithm) {
      this.selectedAlgorithm = algorithm;
      if (!this.customInstructions) {
        this.instructionSequence = this.generateRandomInstructionSequence();
        this.totalInstructions = this.instructionSequence.length;
      }
      this.initializeSimulation();
    },
    initializeSimulation() {
      this.memory = new Array(this.memoryBlocks).fill(null);
      this.memoryHistory = [];
      this.pageFaults = 0;
      this.fifoQueue = new Array(this.memoryBlocks).fill(0);
      this.lruQueue = new Array(this.memoryBlocks).fill(0);
      this.currentInstructionIndex = 0;
      this.currentInstruction =
        this.instructionSequence[this.currentInstructionIndex];
      this.instructionsComplete = false;
      this.instructionTable = [];
      if (this.autoExecutionInterval) {
        clearInterval(this.autoExecutionInterval);
        this.autoExecutionInterval = null;
      }
    },
    generateRandomInstructionSequence() {
      let sequence = [];
      let m = Math.floor(Math.random() * this.totalInstructions);
      let count = 0;

      while (count < this.totalInstructions) {
        sequence.push(m);
        count++;
        if (count >= this.totalInstructions) break;

        m = (m + 1) % this.totalInstructions;
        sequence.push(m);
        count++;
        if (count >= this.totalInstructions) break;

        if (m > 0) {
          m = Math.floor(Math.random() * m);
          sequence.push(m);
          count++;
          if (count >= this.totalInstructions) break;

          m = (m + 1) % this.totalInstructions;
          sequence.push(m);
          count++;
          if (count >= this.totalInstructions) break;
        }

        if (m < this.totalInstructions - 2) {
          m =
            Math.floor(Math.random() * (this.totalInstructions - 1 - m)) +
            m +
            2;
          sequence.push(m);
          count++;
          if (count >= this.totalInstructions) break;

          m = (m + 1) % this.totalInstructions;
          sequence.push(m);
          count++;
        }
      }
      return sequence;
    },
    executePrevInstruction() {
      if (this.currentInstructionIndex > 0) {
        this.currentInstructionIndex--;
        this.currentInstruction =
          this.instructionSequence[this.currentInstructionIndex];

        // 移除最后一条指令的表格条目
        this.instructionTable.pop();
        // 恢复之前的内存状态
        if (this.memoryHistory.length > 0) {
          this.memory = this.memoryHistory.pop();
        }
        // 撤销该指令的内存访问
        let pageNumber = Math.floor(this.currentInstruction / this.pageSize);
        let pageIndex = this.memory.indexOf(pageNumber);

        if (pageIndex !== -1) {
          // 找到页面，更新LRU或FIFO队列
          if (this.selectedAlgorithm === "FIFO") {
            this.fifoQueue[pageIndex]++;
          } else if (this.selectedAlgorithm === "LRU") {
            this.lruQueue[pageIndex]++;
          }
        } else {
          // 页面未找到，即发生了缺页
          this.pageFaults--;
          // 根据算法类型恢复FIFO或LRU队列
          if (this.selectedAlgorithm === "FIFO") {
            this.fifoQueue = this.fifoQueue.slice(0, -1);
          } else if (this.selectedAlgorithm === "LRU") {
            this.lruQueue = this.lruQueue.slice(0, -1);
          }
        }

        this.updatePageFaultRate();
      }
    },
    executeNextInstruction() {
      if (this.currentInstructionIndex < this.instructionSequence.length) {
        // 先把当前指令的内存访问记录下来
        this.memoryHistory.push([...this.memory]);

        this.accessMemory(this.currentInstruction, this.selectedAlgorithm);
        this.currentInstructionIndex++;
        this.currentInstruction =
          this.instructionSequence[this.currentInstructionIndex];
        this.updatePageFaultRate();
      }
      if (this.currentInstructionIndex >= this.instructionSequence.length) {
        this.instructionsComplete = true;
      }
    },
    startAutoExecution() {
      if (this.autoExecutionInterval) return;

      const intervalTime = Math.floor(1000 / this.executionSpeed);
      this.autoExecutionInterval = setInterval(() => {
        if (this.currentInstructionIndex >= this.instructionSequence.length) {
          clearInterval(this.autoExecutionInterval);
          this.autoExecutionInterval = null;
          this.instructionsComplete = true;
        } else {
          this.executeNextInstruction();
        }
      }, intervalTime);
    },
    pauseAutoExecution() {
      if (this.autoExecutionInterval) {
        clearInterval(this.autoExecutionInterval);
        this.autoExecutionInterval = null;
      }
    },
    accessMemory(instruction, algorithm) {
      let pageNumber = Math.floor(instruction / this.pageSize);
      let pageIndex = this.memory.indexOf(pageNumber);

      if (algorithm === "FIFO") {
        this.accessMemoryFIFO(pageNumber, pageIndex);
      } else if (algorithm === "LRU") {
        this.accessMemoryLRU(pageNumber, pageIndex);
      }
    },
    accessMemoryFIFO(pageNumber, pageIndex) {
      let pageFault = false;
      if (pageIndex === -1) {
        // 页面未找到，发生缺页
        pageFault = true;
        this.pageFaults++;

        // 寻找空闲内存块
        const emptyIndex = this.memory.indexOf(null);
        if (emptyIndex !== -1) {
          // 有空闲块，直接插入
          this.memory[emptyIndex] = pageNumber;
          this.fifoQueue[emptyIndex] = this.currentInstructionIndex;
        } else {
          // 没有空闲块，使用FIFO替换页面
          const fifoIndex = this.fifoQueue.indexOf(Math.min(...this.fifoQueue));
          this.memory[fifoIndex] = pageNumber;
          this.fifoQueue[fifoIndex] = this.currentInstructionIndex;
        }
      } else {
        // 页面已在内存中，更新FIFO队列
        this.fifoQueue[pageIndex] = this.currentInstructionIndex;
      }
      this.updateInstructionTable(pageNumber, pageFault);
    },
    accessMemoryLRU(pageNumber, pageIndex) {
      let pageFault = false;
      if (pageIndex === -1) {
        // 页面未找到，发生缺页
        pageFault = true;
        this.pageFaults++;

        // 寻找空闲内存块
        const emptyIndex = this.memory.indexOf(null);
        if (emptyIndex !== -1) {
          // 有空闲块，直接插入
          this.memory[emptyIndex] = pageNumber;
          this.lruQueue[emptyIndex] = this.currentInstructionIndex;
        } else {
          // 没有空闲块，使用LRU替换页面
          const lruIndex = this.lruQueue.indexOf(Math.min(...this.lruQueue));
          this.memory[lruIndex] = pageNumber;
          this.lruQueue[lruIndex] = this.currentInstructionIndex;
        }
      } else {
        // 页面已在内存中，更新LRU队列
        this.lruQueue[pageIndex] = this.currentInstructionIndex;
      }
      this.updateInstructionTable(pageNumber, pageFault);
    },
    resetSimulation() {
      this.totalInstructions = 320;
      this.memory = [];
      this.memoryHistory = [];
      this.pageFaults = 0;
      this.pageFaultRate = 0;
      this.instructionSequence = [];
      this.fifoQueue = [];
      this.lruQueue = [];
      this.currentInstruction = null;
      this.currentInstructionIndex = 0;
      this.selectedAlgorithm = null;
      if (this.autoExecutionInterval) {
        clearInterval(this.autoExecutionInterval);
        this.autoExecutionInterval = null;
      }
      this.instructionsComplete = false;
      this.customInstructions = "";
      this.instructionTable = [];
      this.currentPage = 1;
    },
    updateInstructionTable(pageNumber, pageFault) {
      this.instructionTable.push({
        index: this.currentInstructionIndex,
        instruction: this.currentInstruction,
        pageNumber: pageNumber,
        pageFault: pageFault ? "是" : "否",
        pageFaultRate: this.pageFaultRate.toFixed(3),
      });
    },
    updatePageFaultRate() {
      this.pageFaultRate =
        (this.pageFaults / (this.currentInstructionIndex + 1)) * 100;
    },
    prevPage() {
      if (this.currentPage > 1) {
        this.currentPage--;
      }
    },
    nextPage() {
      if (this.currentPage < this.totalPages) {
        this.currentPage++;
      }
    },
  },
};
</script><style scoped>
.memory-management {
  max-width: 1200px;
  margin: -30px;
  padding-left: 50px;
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 5px;
  margin-bottom: 20px;
}

.algorithm-buttons,
.execution-buttons {
  display: flex;
  gap: 20px;
  flex-wrap: wrap;
}

.memory-table,
.page-turner,
.completion-message {
  margin-bottom: 20px;
}

.instruction-table {
  margin-bottom: 10px;
}
.buttons {
  font-size: 16px;
  padding: 10px 20px;
  margin: 5px 0;
  background-color: #4caf50;
  color: #fff;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.buttons:hover {
  background-color: #45a049;
}

.buttons:disabled {
  background-color: lightgray;
  cursor: not-allowed;
}

label {
  margin-right: 20px;
}

input {
  padding: 5px;
  border-radius: 3px;
  border: 1px solid #ccc;
}

.memory-table,
.instruction-table {
  width: 100%;
  border-collapse: collapse;
}

.memory-table th,
.memory-table td,
.instruction-table th,
.instruction-table td {
  padding: 5px;
  border: 1px solid #ccc;
  text-align: center;
}

.memory-table th,
.instruction-table th {
  background-color: #f2f2f2;
}

.page-turner {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 20px;
}

.modal-content {
  background-color: #fefefe;
  margin: 5% auto;
  padding: 20px;
  border: 1px solid #888;
  width: 50%;
  border-radius: 10px;
}

.modal {
  display: flex;
  justify-content: center;
  align-items: center;
  position: fixed;
  z-index: 1;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.4);
}

textarea {
  width: 100%;
  height: 100px;
  padding: 10px;
  margin-top: 10px;
  border-radius: 5px;
  border: 1px solid #ccc;
}

.close {
  color: #aaa;
  float: right;
  font-size: 28px;
  font-weight: bold;
}

.close:hover,
.close:focus {
  color: black;
  text-decoration: none;
  cursor: pointer;
}

.error-message {
  padding: 10px;
  background-color: #ffe6e6;
  border: 1px solid #ff0000;
  border-radius: 5px;
  margin: 0px 0px;
}

.instruction-count-container,
.execution-speed-container {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-right: 10px;
}

.buttons.reset-button {
  width: 100%;
  margin: 5px auto;
  background-color: #add8e6;
}

.memory-table td {
  width: 20%;
  height: 20px;
  border: 1px solid #ccc;
  text-align: center;
}
</style>