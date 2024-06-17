<template>
  <div class="tetris">
    <div class="screen" ref="screen">
      <div v-for="(cell, index) in grid" :key="index" :class="['cell', cell]"></div>
    </div>
    <div class="controls">
      <button @click="moveLeft">Left</button>
      <button @click="moveRight">Right</button>
      <button @click="rotate">Rotate</button>
      <button @click="moveDown">Down</button>
    </div>
  </div>
</template>

<script>
import { reactive } from "vue";
export default {
  data() {
    return {
      // 新图形填充用filled,已经固定的用fixed
      width: 10,  // 宽格子数
      height: 20, // 高格子数
      totalCeil: 200, // 总格子数
      grid: reactive(Array(200).fill(null)),  // 定义网格界面
      currentPosition: 4, // 图形出现的初始位置（中心点）
      currentRotation: 0, // 当前旋转
      timerId: null,  // 下落时间间隔
      tetrominoes: [],  // 定义空图形数组
      currentTetromino: null, // 当前图形（数组）
      removerow: true,  // 判断是否满足移除条件
      eachrow: 0, // 用于遍历每一行
      beRemovedRow: 0,  // 记录被移除的行
      ceil: 0, // 用于遍历每一个格子
      Flag: false,
    };
  },
  created() { // 初始化调用函数
    this.tetrominoes = [  // 定义图形
      [0, this.width, this.width * 2, 1], // “L”形
      [0, this.width, this.width + 1, this.width * 2 + 1], // “Z”形
      [0, this.width, this.width * 2, this.width + 1],  // “山”形
      [0, 1, this.width, this.width + 1], // 方形
      [0, this.width, this.width * 2, this.width * 3] // “|”形
    ];
    this.currentTetromino = this.getRandomTetromino();  // 获取随机形状
    this.draw(); // 绘图
    // this.timerId = setInterval(this.moveDown, 1000); // 图形下落时间间隔
    this.removeRow();
  },
  mounted() {
    window.addEventListener("keydown", this.handleKeydown);
  },
  beforeUnmount() {
    window.removeEventListener("keydown", this.handleKeydown);
  },
  methods: {
    getRandomTetromino() {
      const randomIndex = Math.floor(Math.random() * this.tetrominoes.length);  // 生成随机下标
      return this.tetrominoes[randomIndex]; // 返回随机数组（图形）
    },
    draw() {  // 绘出图形
      this.currentTetromino.forEach(index => {
        // console.log(this.currentPosition + index);
        this.grid[this.currentPosition + index] = 'filled'; // 给格子填充颜色
      });
    },
    undraw() {  // 消除图形
      this.currentTetromino.forEach(index => {
        this.grid[this.currentPosition + index] = null;
      });
    },
    fixCeil() { // 固定标识，用于与新图形进行重叠判断
      this.currentTetromino.forEach(index => {
        this.grid[this.currentPosition + index] = "fixed";
      })
    },
    handleKeydown(event) {
      switch (event.key) {
        case "ArrowLeft":
          this.moveLeft();
          break;
        case "ArrowRight":
          this.moveRight();
          break;
        case "ArrowUp":
          this.rotate();
          break;
        case "ArrowDown":
          this.moveDown();
          break;
      }
    },
    moveDown() {  // 向下移动
      const isAtBottomEdge = this.currentTetromino.some(index => (this.currentPosition + index) + this.width >= this.width * this.height);
      const isRepeat = this.currentTetromino.some(index => this.grid[this.currentPosition + index + this.width] === 'fixed');
      if (!isAtBottomEdge && !isRepeat) {  // 如果没到达底边界
        this.undraw();
        this.currentPosition += this.width; // 换行
        this.draw();
      } else {  // 重置初始位置，生成新的图形
        this.fixCeil();
        this.removeRow();
        this.currentPosition = 4;
        this.currentTetromino = this.getRandomTetromino();  // 重新获取随机形状
        this.draw();
      }
    },
    moveLeft() {  // 向左移动
      this.undraw();
      const isAtLeftEdge = this.currentTetromino.some(index => (this.currentPosition + index) % this.width === 0);
      const isRepeat = this.currentTetromino.some(index => this.grid[this.currentPosition + index - 1] === 'fixed');
      if (!isAtLeftEdge && !isRepeat) this.currentPosition -= 1;
      this.draw();
    },
    moveRight() { // 向右移动
      this.undraw();
      const isAtRightEdge = this.currentTetromino.some(index => (this.currentPosition + index) % this.width === this.width - 1);
      const isRepeat = this.currentTetromino.some(index => this.grid[this.currentPosition + index + 1] === 'fixed');
      if (!isAtRightEdge && !isRepeat) this.currentPosition += 1;
      this.draw();
    },
    rotate() {
      // 旋转逻辑待补充
      if(this.currentTetromino === this.tetrominoes[0]){
        
      }
    },
    getTop() {  // 获取每列最上面那个填充的元素（如果被移除行上方为空，不记录在内）
      let topValuesByRemainder = {};

      for (var num = 0; num <= this.beRemovedRow; num++) {  //遍历取余，只保存每列最上方且被填充的数字
        const remainder = num % 10; // 取余器
        if (this.grid[num] === "fixed" && !topValuesByRemainder[remainder] || num < topValuesByRemainder[remainder]) {
          topValuesByRemainder[remainder] = num;  // 记录符合的数
        }
      }
      const result = Object.values(topValuesByRemainder); // 将结果转换为数组
      console.log(result);
      return result;  // 返回结果数组
    },
    removeRow() {
      for (this.eachrow = 9; this.eachrow <= this.totalCeil; this.eachrow += 10) {  // 遍历行
        this.removerow = true;  // 默认移除该行
        for (this.ceil = (this.eachrow - 9); this.ceil <= this.eachrow; this.ceil++) {  // 遍历每个格子
          if (this.grid[this.ceil] != 'fixed') { // 如果该行存在空格子
            this.removerow = false; // 移除为假
            break;  // 直接跳出本行循环
          }
        }
        if (this.removerow) { // 如果要移除该行
          this.beRemovedRow = this.eachrow; // 记录被移除行的行末数字（用于行下移判断）
          for (this.ceil = (this.eachrow - 9); this.ceil <= this.eachrow; this.ceil++) {
            this.grid[this.ceil] = null;  // 取消填充
          }
          this.RowDown(); // 实现下移
        }
      }
    },
    RowDown() { // 满足两个条件：1、在被消除行的上方 2、上方不为空
      this.getTop().forEach(ceil => { // 遍历数组
        this.grid[ceil] = null; // 把每列最上方的填充值去掉
        for (this.ceil = (this.beRemovedRow - 9); this.ceil <= this.beRemovedRow; this.ceil++) {  // 遍历被移除的行
          if ((ceil % 10) == (this.ceil % 10)) {  // 如果是同一列上方被移除的块
            this.grid[this.ceil] = 'fixed';  // 填充到被移除的行
          }
        }
      });
    }
  },
  beforeDestroy() {
    clearInterval(this.timerId);
  }
};
</script>

<style>
* {
  margin: 0;
  padding: 0;
}

html,
body {
  height: 100%;
  display: flex;
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  height: 100%;
}

.tetris {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.screen {
  display: grid;
  grid-template-columns: repeat(10, 20px);
  grid-template-rows: repeat(20, 20px);
  gap: 1px;
  background-color: #ddd;
  border: 2px solid #000;
}

.cell {
  width: 20px;
  height: 20px;
  background-color: white;
}

.cell.filled {
  background-color: blue;
}

.cell.fixed {
  background-color: blue;
}

.controls {
  margin-top: 20px;
}

button {
  margin: 5px;
  padding: 10px;
}
</style>
