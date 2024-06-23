<template>
  <header>
    <h3>Tetris</h3>
  </header>
  <div class="tetris" @touchend="handleTouchEnd" @touchmove="handleTouchMove">
    <audio ref="audioPlayer">
      <!-- 用于部署之后用fetch可以调用，并不直接调用该音频 -->
      <source src="../src/assets/music.mp3" type="audio/mpeg">
    </audio>
    <div class="game-container">
      <div class="screen" ref="screen">
        <div v-for="(cell, index) in grid" :key="index" :class="['cell', cell]"></div>
      </div>
      <div class="panel">
        <div class="score">score:<span>{{ score }}</span></div>
      </div>
    </div>
    <div class="keywords" @touchend="handleTouchEnd" @touchmove="handleTouchMove">
      <div class="start">
        <button @touchstart="handleTouchStart('start')" @click="StartGame" :disabled="!gameover">Start</button>
      </div>
      <div class="controls">
        <div class="rotate">
          <button @touchstart="handleTouchStart('rotate')" @mousedown="buttonDown('rotate')" @mouseup="buttonUp"
            :disabled="gameover">Rotate</button>
        </div>
        <div class="move">
          <button @touchstart="handleTouchStart('left')" @mousedown="buttonDown('left')" @mouseup="buttonUp"
            @click="moveLeft" :disabled="gameover">Left</button>
          <button @touchstart="handleTouchStart('down')" @mousedown="buttonDown('down')" @mouseup="buttonUp"
            @click="moveDown" :disabled="gameover">Down</button>
          <button @touchstart="handleTouchStart('right')" @mousedown="buttonDown('right')" @mouseup="buttonUp"
            @click="moveRight" :disabled="gameover">Right</button>
        </div>
      </div>
    </div>
  </div>
  <div class="my-tag">wyh 2024/06/16</div>
</template>

<script>
// 正常玩法
// 下降旋转法
// 颜色匹配法
// 设置得分
// 设置gameover
import { reactive } from "vue";
export default {
  data() {
    return {
      // 新图形填充用filled,已经固定的用fixed
      width: 10,  // 宽格子数
      height: 20, // 高格子数
      totalCeil: 200, // 总格子数
      grid: reactive(Array(200).fill(null)),  // 定义网格界面
      currentPosition: 14, // 图形出现的初始位置（中心点）！并非旋转中心点！
      currentRotation: 0, // 初始化当前旋转
      nextRotation: 0,  // 初始化下一次旋转
      timerDorp: null,  // 下落时间间隔
      timerbutton: null, // 点击按钮计时器
      tetrominoes: [],  // 定义空图形数组
      currentTetromino: null, // 当前图形（数组）
      removerow: true,  // 判断是否满足移除条件
      eachrow: 0, // 用于遍历每一行
      beRemovedRow: 0,  // 记录被移除的行
      ceil: 0, // 用于遍历每一个格子
      gameover: true, // 默认停止游戏
      score: 0, // 得分
      currentButton: null,  // 当前正在被点击的按钮
      currentAudioTime: 0,  // 音频播放开始时间
      endAudioTime: 0,  // 音频播放结束时间
      audioPlayer: null, // 音频播放器
      curretTouching: null, // 手机端获取长按按钮
      nextone: false, // 是否为下一个图形
    };
  },
  setup() {
    // 创建音频上下文
    const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    let audioBuffer = null;
    let source = null;

    // 预加载音频文件
    fetch('https://websiteforwyh.github.io/tetris/media/music.15aae32a.mp3')
      .then(response => response.arrayBuffer())
      .then(arrayBuffer => audioCtx.decodeAudioData(arrayBuffer))
      .then(decodedAudioData => {
        audioBuffer = decodedAudioData;
      })
      .catch(error => console.error('Error loading audio file', error));  // 抓取异常并打印信息

    // 播放音频的函数
    const playAudio = (startTime = 0, endTime) => {
      if (audioBuffer) {  // 如果音频存在
        // 检查时间范围
        if (startTime < 0 || startTime >= audioBuffer.duration || endTime <= startTime || endTime > audioBuffer.duration) {
          console.error("Invalid start or end time");
          return;
        }

        // 创建一个新的音频源
        source = audioCtx.createBufferSource();
        source.buffer = audioBuffer;
        source.connect(audioCtx.destination);

        // 设置播放结束时间
        const duration = endTime - startTime;

        // 从指定的时间点开始播放
        source.start(0, startTime, duration);

        // 消除行音频段：(0, 0.9)
        // 未知音频：(1.2, 1.5)
        // 旋转音频：(2, 2.3)
        // 移动音频：(2.7, 3)
        // 结束音乐：(3.5, 7.2)
        // 碰撞结束：(8, 9)
      }
    };

    return {  // 返回该函数
      playAudio,
    };
  },
  created() { // 初始化调用函数
    this.tetrominoes = [  // 定义图形
      [ // “L”形
        [-this.width - 1, -this.width, 0, this.width],
        [- 1, 0, 1, -this.width + 1],
        [-this.width, 0, this.width, this.width + 1],
        [this.width - 1, - 1, 0, 1],
      ],
      [ // 反“L”形
        [-this.width, 0, this.width, -this.width + 1],
        [-1, 0, 1, this.width + 1],
        [0, -this.width, this.width, this.width - 1],
        [-this.width - 1, - 1, 0, 1],
      ],
      [ // 反“Z”形
        [0, -this.width, 1, this.width + 1],
        [- 1, -this.width, 0, -this.width + 1],
        [0, -this.width, 1, this.width + 1],
        [- 1, -this.width, 0, -this.width + 1],
      ],
      [ // “Z”形
        [1, -this.width + 1, this.width, 0],
        [-this.width - 1, 0, -this.width, 1],
        [1, -this.width + 1, this.width, 0],
        [-this.width - 1, 0, -this.width, 1],
      ],
      [ // “山”形
        [0, -this.width, this.width, 1],
        [- 1, 0, 1, this.width],
        [0, -this.width, this.width, - 1],
        [0, -this.width, - 1, 1],
      ],
      [ // 方形
        [0, 1, -this.width, -this.width + 1],
        [0, 1, -this.width, -this.width + 1],
        [0, 1, -this.width, -this.width + 1],
        [0, 1, -this.width, -this.width + 1],
      ],
      [ // “|”形
        [-this.width, 0, this.width, this.width * 2],
        [- 1, 0, 1, 2],
        [-this.width, 0, this.width, this.width * 2],
        [- 1, 0, 1, 2],
      ],
    ];
    this.currentTetromino = this.getRandomTetromino();  // 获取随机形状
    this.nextRotation = (this.nextRotation < 3) ? this.currentRotation + 1 : 0; // 定义下一个旋转状态下标
  },
  mounted() {
    window.addEventListener("keydown", this.handleKeydown);
    this.audioPlayer = this.$refs.audioPlayer;  // 将音频播放器设为关联的播放器audio
  },
  beforeUnmount() {
    window.removeEventListener("keydown", this.handleKeydown);
  },
  methods: {
    getRandomTetromino() {  // 获取随机方块形状
      const randomIndex = Math.floor(Math.random() * this.tetrominoes.length);  // 生成随机下标
      return this.tetrominoes[randomIndex]; // 返回随机数组（图形）
    },
    draw() {  // 绘出图形
      this.currentTetromino[this.currentRotation].forEach(index => {
        this.grid[this.currentPosition + index] = 'filled'; // 给格子填充颜色
      });
    },
    undraw() {  // 消除图形
      this.currentTetromino[this.currentRotation].forEach(index => {
        this.grid[this.currentPosition + index] = null;
      });
    },
    fixCeil() { // 固定标识，用于与新图形进行重叠判断
      this.currentTetromino[this.currentRotation].forEach(index => {
        this.playAudio(1.2, 1.5); // 固定音频
        this.grid[this.currentPosition + index] = "fixed";
      })
    },
    handleKeydown(event) {  // 设置按键功能
      switch (event.key) {
        case "ArrowLeft":
        case "A":
        case "a":
          if (!this.gameover) {
            this.moveLeft();
          }
          break;
        case "ArrowRight":
        case "D":
        case "d":
          if (!this.gameover) {
            this.moveRight();
          }
          break;
        case "ArrowUp":
        case "W":
        case "w":
          if (!this.gameover) {
            this.rotate();
          }
          break;
        case "ArrowDown":
        case "S":
        case "s":
          if (this.nextone === true) {  // 如果进入下一个，暂时取消连续移动
            break;
          } else {
            if (!this.gameover) {
              this.playAudio(2.7, 3); // 播放移动音频
              this.moveDown();
            }
            break;
          }
        case " ": // 空格键
          if (this.gameover) {  // 游戏结束状态才调用
            this.StartGame();
          }
          break;
      }
    },
    buttonDown(button) {
      // 按下按钮持续移动
      this.buttonUp();
      this.currentButton = button;
      if (this.currentButton === 'left') {
        this.timerbutton = setInterval(this.moveLeft, 90);
      } else if (this.currentButton === 'right') {
        this.timerbutton = setInterval(this.moveRight, 90);
      } else if (this.currentButton === 'down') {
        this.timerbutton = setInterval(this.moveDown, 80);
      } else if (this.currentButton === 'rotate') {
        this.rotate();
      }
    },
    buttonUp() {
      // 不点击按钮时清除计时器
      clearInterval(this.timerbutton);
      this.timerbutton = null;
      this.currentButton = null;  // 重置被点击的按钮
    },
    moveDown() {  // 向下移动
      const isAtBottomEdge = this.currentTetromino[this.currentRotation].some(index => (this.currentPosition + index) + this.width >= this.totalCeil);
      // const isRepeat = this.currentTetromino[this.currentRotation].some(index => this.grid[this.currentPosition + index + this.width] === 'fixed');     
      const isRepeat = this.currentTetromino[this.currentRotation].some(index => (this.grid[this.currentPosition + index + this.width] !== null && this.grid[this.currentPosition + index + this.width] !== "filled"));
      if (!this.gameover) {
        if (!isAtBottomEdge && !isRepeat) {  // 如果没到达底边界或与其他方块碰撞
          this.nextone = false;
          this.undraw();
          if (this.currentButton || this.curretTouching) {  // 如果是主动点击
            this.playAudio(2.7, 3); // 播放音频
          }
          this.currentPosition += this.width; // 换行
          this.draw();
        } else {  // 重置初始位置，生成新的图形
          this.nextone = true;
          this.buttonUp();
          this.fixCeil(); // 固定旧图形（改变其颜色）
          this.removeRow(); // 判断是否需要移除行
          this.currentTetromino = this.getRandomTetromino();  // 重新获取随机形状
          this.currentPosition = 14; // 重置图形位置（中心点）
          const isFirstRow = this.currentTetromino[this.currentRotation].some(index => (this.currentPosition + index) / 10 < 1);
          if (!isFirstRow) this.currentPosition -= this.width;
          this.GameOver();  // 游戏结束判断
          this.draw();
        }
      }
    },
    moveLeft() {  // 向左移动
      this.undraw();
      const isAtLeftEdge = this.currentTetromino[this.currentRotation].some(index => (this.currentPosition + index) % this.width === 0);
      const isLeftRepeat = this.currentTetromino[this.currentRotation].some(index => this.grid[this.currentPosition + index - 1] === 'fixed');
      if (!this.gameover && !isAtLeftEdge && !isLeftRepeat) {
        this.playAudio(2.7, 3); // 播放移动音频
        this.currentPosition -= 1;
      }
      this.draw();
    },
    moveRight() { // 向右移动
      this.undraw();
      const isAtRightEdge = this.currentTetromino[this.currentRotation].some(index => (this.currentPosition + index) % this.width === this.width - 1);
      const isRightRepeat = this.currentTetromino[this.currentRotation].some(index => this.grid[this.currentPosition + index + 1] === 'fixed');
      if (!this.gameover && !isAtRightEdge && !isRightRepeat) {
        this.currentPosition += 1;
        this.playAudio(2.7, 3); // 播放移动音频
      }
      this.draw();
    },
    rotate() {  // 图形旋转
      // 在左边界旋转，超出的方块会出现在上一行的右边界，右边界同理，以此作为左右边界的限制旋转依据
      const isAtLeftEdge = (this.currentTetromino[this.nextRotation].some(index => (this.currentPosition + index) % 10 > 7) && (this.currentPosition % this.width < 5) ? true : false);
      const isAtRightEdge = (this.currentTetromino[this.nextRotation].some(index => (this.currentPosition + index) % 10 < 2) && (this.currentPosition % this.width >= 5) ? true : false);
      const isAtBottomEdge = this.currentTetromino[this.nextRotation].some(index => (this.currentPosition + index) > this.totalCeil);
      const isRepeat = this.currentTetromino[this.nextRotation].some(index => this.grid[this.currentPosition + index] === 'fixed');
      if (!this.gameover && !isRepeat && !isAtBottomEdge && !isAtLeftEdge && !isAtRightEdge) {
        this.playAudio(2, 2.3);
        this.undraw();
        this.currentRotation = (this.currentRotation < 3) ? this.currentRotation + 1 : 0; // 0-3直接循环
        this.nextRotation = (this.nextRotation < 3) ? this.currentRotation + 1 : 0; // 定义下一个旋转状态下标
        this.draw();
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
          this.playAudio(0, 0.9); // 消除音效
          this.RowDown(); // 实现下移
          this.score += 10; // 加分
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
    },
    resetCeil() {  // 重新开始时重置所有格子
      for (var i = 0; i < this.totalCeil; i++) {
        this.grid[i] = null;
      }
    },
    StartGame() { // 开始游戏
      if (this.gameover) {
        this.gameover = false;  // 设置游戏结束为假
        this.resetCeil(); // 重置格子
        this.score = 0; // 重置得分
        this.draw();  // 绘图
        this.timerDorp = setInterval(this.moveDown, 1000); // 图形下落时间间隔   
      }
    },
    GameOver() {  // 游戏结束判断
      const Repeat = this.currentTetromino[this.currentRotation].some(index => this.grid[this.currentPosition + index] === 'fixed');
      if (Repeat) { // 检测到碰撞
        this.playAudio(8, 9); // 碰撞音效
        const intervalId = setInterval(() => {  // 延时播放下一段音乐
          this.scanCeilFill(199);
          this.playAudio(3.5, 7.2); // 游戏结束音效
          clearInterval(intervalId);
        }, 1900);
        this.gameover = true;
        this.removeInterval();
      }
    },
    scanCeilFill(start) { // 游戏结束扫描动画
      if (start < 9) {
        this.scanCeilRemove(0); // 填充结束开始调用清除
        return; // 终止条件
      }

      for (let i = start; i >= start - 9; i--) {  // 遍历填充
        this.grid[i] = 'fixed';
      }

      setTimeout(() => {
        this.scanCeilFill(start - 10); // 递归调用，间隔10秒打印下一行
      }, 80);
    },
    scanCeilRemove(end) { // 游戏结束扫描动画
      if (end > 199) {
        return; // 终止条件
      }

      for (let i = end; i <= end + 9; i++) {  // 遍历置空
        this.grid[i] = null;
      }

      setTimeout(() => {
        this.scanCeilRemove(end + 10); // 递归调用，间隔10秒打印下一行
      }, 80);
    },
    removeInterval() {  // 去除计时器函数
      clearInterval(this.timerDorp);
      this.timerDorp = null;
    },
    resizeWindow() {
      console.log(window.screen.width);
    },
    handleTouchStart(touching) {
      clearInterval(this.timerbutton);  // 清除一遍定时器，避免上次的未成功清除
      this.timerbutton = null;
      this.curretTouching = touching;
      if (this.curretTouching === 'start') {
        this.StartGame();
      } else if (!this.gameover) {
        if (this.curretTouching === 'left') {
          this.moveLeft();
          this.timerbutton = setInterval(this.moveLeft, 90);
        } else if (this.curretTouching === 'right') {
          this.moveRight();
          this.timerbutton = setInterval(this.moveRight, 90);
        } else if (this.curretTouching === 'down') {
          this.moveDown();
          this.timerbutton = setInterval(this.moveDown, 80);
        } else if (this.curretTouching === 'rotate') {
          this.rotate();
        }
      }
    },
    handleTouchEnd(e) {
      e.preventDefault(); // 阻止默认行为
      clearInterval(this.timerbutton);
      this.timerbutton = null;
      this.curretTouching = null;
    },

    handleTouchMove(e) {
      e.preventDefault();
      clearInterval(this.timerbutton);
      this.timerbutton = null;
      this.curretTouching = null;
    },
  },
  beforeDestroy() {
    clearInterval(this.timerDorp);
    this.timerDorp = null;
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
  touch-action: manipulation;
  -webkit-user-select: none;
  user-select: none;
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  -webkit-user-select: none;
  user-select: none;
}

.tetris {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  border-radius: 5px;
  background-color: bisque;
  height: 100%;
  width: 375px;
  -webkit-user-select: none;
  user-select: none;
}

.game-container {
  display: flex;
  justify-content: space-between;
  width: 240px;
  padding: 0 20px 0 0;
  position: relative;
  right: 30px;
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

.filled {
  background-color: blue;
}

.fixed {
  background-color: grey;
}

.keywords {
  margin-top: 20px;
  display: flex;
  flex-direction: row-reverse;
  align-items: center;
}

button {
  margin: 5px;
  padding: 10px;
  cursor: pointer;
  /* 阻止默认选中行为 */
  -webkit-user-select: none;
  user-select: none;
  border-radius: 35px;
  width: 60px;
  height: 60px;
}

.start button{
  border-radius: 20px;
}

.score {
  padding: 10px;
}

.score span {
  padding: 10px;
}

.start {
  margin: 20px;
}

.rotate {
  position: relative;
}

.my-tag {
  display: block;
  position: absolute;
  bottom: 0;
}

header {
  position: absolute;
  top: 0;
}
</style>
