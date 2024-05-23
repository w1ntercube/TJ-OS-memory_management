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
    computed: {
      totalPages() {
        // 计算总页数，根据指令表格的长度和每页显示的指令数
        return Math.ceil(this.instructionTable.length / this.pageSize);
      },
      visibleInstructions() {
        // 计算当前页显示的指令
        const start = (this.currentPage - 1) * this.pageSize;
        const end = start + this.pageSize;
        return this.instructionTable.slice(start, end);
      },
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
      // 去除输入中的空白字符
      const trimmedInput = this.customInstructions.trim();
      if (trimmedInput === "") {
        // 输入为空或者只包含空格，设置输入错误状态并返回
        this.inputError = true;
        return;
      }
      // 将输入字符串拆分为数字数组
      const instructions = this.customInstructions
        .trim()
        .split(/\s+/)
        .map(Number);
      const valid = instructions.every(Number.isInteger);
      if (valid && instructions.length > 0) {
        // 如果所有指令都是有效整数并且非空
        this.instructionSequence = instructions;
        this.totalInstructions = instructions.length;
        this.currentInstructionIndex = 0;
        this.currentInstruction =
          this.instructionSequence[this.currentInstructionIndex];
        this.showCustomInstructionInput = false; // 隐藏自定义指令输入框
        this.inputError = false;
        this.initializeSimulation();
      } else {
        // 如果输入不合法，设置输入错误状态但不清空输入框
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
      let m = Math.floor(Math.random() * this.totalInstructions); // 随机选取一个起始执行指令，如序号为m
      let count = 0;

      while (count < this.totalInstructions) {
        sequence.push(m);
        count++;
        if (count >= this.totalInstructions) break;

        m = (m + 1) % this.totalInstructions; // 顺序执行下一条指令，即序号为m+1的指令
        sequence.push(m);
        count++;
        if (count >= this.totalInstructions) break;

        if (m > 0) {
          m = Math.floor(Math.random() * m); // 通过随机数，跳转到前地址部分0－m-1中的某个指令处，其序号为m1
          sequence.push(m);
          count++;
          if (count >= this.totalInstructions) break;

          m = (m + 1) % this.totalInstructions; // 顺序执行下一条指令，即序号为m1+1的指令
          sequence.push(m);
          count++;
          if (count >= this.totalInstructions) break;
        }

        if (m < this.totalInstructions - 2) {
          m =
            Math.floor(Math.random() * (this.totalInstructions - 1 - m)) +
            m +
            2; // 通过随机数，跳转到后地址部分m1+2~319中的某条指令处，其序号为m2
          sequence.push(m);
          count++;
          if (count >= this.totalInstructions) break;

          m = (m + 1) % this.totalInstructions; // 顺序执行下一条指令，即m2+1处的指令
          sequence.push(m);
          count++;
        }
      } // 重复跳转到前地址部分、顺序执行、跳转到后地址部分、顺序执行的过程，
      // end of while
      return sequence;
    },
    executePrevInstruction() {
      if (this.currentInstructionIndex > 0) {
        // 如果当前指令索引大于0，则可以回退指令
        this.currentInstructionIndex--;
        this.currentInstruction =
          this.instructionSequence[this.currentInstructionIndex]; // 更新当前指令

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
          // 如果找到了页面，更新LRU或FIFO队列
          if (this.selectedAlgorithm === "FIFO") {
            this.fifoQueue[pageIndex]++;
          } else if (this.selectedAlgorithm === "LRU") {
            this.lruQueue[pageIndex]++;
          }
        } else {
          // 如果页面未找到，即发生了缺页
          this.pageFaults--;

          if (this.selectedAlgorithm === "FIFO") {
            this.fifoQueue = this.fifoQueue.slice(0, -1); // 恢复FIFO队列
          } else if (this.selectedAlgorithm === "LRU") {
            this.lruQueue = this.lruQueue.slice(0, -1); // 恢复LRU队列
          }
        }

        this.updatePageFaultRate();
      }
    },
    executeNextInstruction() {
      if (this.currentInstructionIndex < this.instructionSequence.length) {
        // 如果当前指令索引小于指令序列的长度，则可以执行下一条指令

        // 先把当前指令的内存状态记录下来
        this.memoryHistory.push([...this.memory]);

        // 访问当前指令对应的内存页面
        this.accessMemory(this.currentInstruction, this.selectedAlgorithm);

        // 增加当前指令索引，更新当前指令
        this.currentInstructionIndex++;
        this.currentInstruction =
          this.instructionSequence[this.currentInstructionIndex];

        // 更新缺页率
        this.updatePageFaultRate();
      }

      // 检查是否已执行完所有指令
      if (this.currentInstructionIndex >= this.instructionSequence.length) {
        this.instructionsComplete = true; // 标记指令执行完成
      }
    },
    startAutoExecution() {
      if (this.autoExecutionInterval) return; // 如果已经在自动执行，则直接返回

      // 计算自动执行的时间间隔（毫秒）
      const intervalTime = Math.floor(1000 / this.executionSpeed);

      // 设置定时器以自动执行指令
      this.autoExecutionInterval = setInterval(() => {
        if (this.currentInstructionIndex >= this.instructionSequence.length) {
          // 如果当前指令索引已达到指令序列长度，停止自动执行
          clearInterval(this.autoExecutionInterval);
          this.autoExecutionInterval = null;
          this.instructionsComplete = true; // 标记指令执行完成
        } else {
          // 执行下一条指令
          this.executeNextInstruction();
        }
      }, intervalTime);
    },
    pauseAutoExecution() {
      if (this.autoExecutionInterval) {
        // 如果当前正在自动执行，则清除定时器以暂停执行
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
        // 如果页面不在内存中，发生缺页
        pageFault = true;
        this.pageFaults++;

        // 寻找空闲内存块
        const emptyIndex = this.memory.indexOf(null);
        if (emptyIndex !== -1) {
          // 如果有空闲块，直接插入页面
          this.memory[emptyIndex] = pageNumber;
          this.fifoQueue[emptyIndex] = this.currentInstructionIndex;
        } else {
          // 如果没有空闲块，使用FIFO算法替换页面
          const fifoIndex = this.fifoQueue.shift(); // 取出最早插入的页面索引
          this.memory[fifoIndex] = pageNumber; // 替换该索引处的页面
          this.fifoQueue.push(fifoIndex); // 更新FIFO队列
        }
      }
      // 更新指令表格，记录当前指令的执行情况
      this.updateInstructionTable(pageNumber, pageFault);
    },
    accessMemoryLRU(pageNumber, pageIndex) {
      let pageFault = false;
      if (pageIndex === -1) {
        // 如果页面不在内存中，发生缺页
        pageFault = true;
        this.pageFaults++; // 增加缺页次数

        // 寻找空闲内存块
        const emptyIndex = this.memory.indexOf(null);
        if (emptyIndex !== -1) {
          // 如果有空闲块，直接插入页面
          this.memory[emptyIndex] = pageNumber;
          this.lruQueue[emptyIndex] = this.currentInstructionIndex;
        } else {
          // 如果没有空闲块，使用LRU算法替换页面
          const lruIndex = this.lruQueue.indexOf(Math.min(...this.lruQueue)); // 找到最久未使用的页面索引
          this.memory[lruIndex] = pageNumber; // 替换该索引处的页面
          this.lruQueue[lruIndex] = this.currentInstructionIndex; // 更新LRU队列
        }
      } else {
        // 如果页面在内存中，更新LRU队列
        this.lruQueue[pageIndex] = this.currentInstructionIndex;
      }
      // 更新指令表格，记录当前指令的执行情况
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
        // 如果当前正在自动执行，则清除定时器以停止执行
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