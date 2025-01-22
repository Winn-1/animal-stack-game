<script setup lang="ts">
import { ref, onUnmounted } from 'vue'
import type { CSSProperties } from 'vue'

// 图片样式配置
const animalImageStyle: CSSProperties = {
  width: '100%',
  height: '100%',
  objectFit: 'contain',
  display: 'block'
}

// 游戏状态
const gameStarted = ref(false)
const gameOver = ref(false)
const gameResult = ref<'timeout' | 'full' | 'win' | null>(null)
const timeLeft = ref(600) // 10分钟 = 600秒
const timer = ref<number | null>(null)
const showConfirmDialog = ref(false)
const isPaused = ref(false) // 游戏暂停状态

// 待消除区域（最多6个格子）
const pendingArea = ref<Array<{ id: number; type: string; visible?: boolean }>>([]);

// 动物类型和图片路径
const animalTypes = ['sheep', 'goose', 'chicken', 'chick', 'cow', 'goat', 'brown_goat', 'donkey', 'horse']
const animalImages = {
  goose: "https://img.z4a.net/images/2025/01/21/ccd1b713c84ed4a0226a9896c11312df.png",
  sheep: "https://img.z4a.net/images/2025/01/21/7914ffb601280ee2b153987a2ec61254.png",
  chicken: "https://img.z4a.net/images/2025/01/21/907056c68627d80ba00ffb8121b613a2.png",
  chick: "https://img.z4a.net/images/2025/01/21/01640dec8c93ce588ef149431f0b6c21.png",
  cow: "https://img.z4a.net/images/2025/01/21/be69d38164e6be70b0475948951cbe05.png",
  goat: "https://img.z4a.net/images/2025/01/21/aa065361fe486a1c7b9679849c5bb00b.png",
  brown_goat: "https://img.z4a.net/images/2025/01/21/c7d7d252eb9c21a4ce900129a07e24bb.png",
  donkey: "https://img.z4a.net/images/2025/01/21/df14fb83b47fd261886b4116c2115c0f.png",
  horse: "https://img.z4a.net/images/2025/01/21/3049d4665f79df527382466721717fa7.png"
}

// 游戏区域的动物
interface Animal {
  id: number;
  type: string;
  x: number;
  y: number;
  visible: boolean;
  zIndex: number;
  offsetX?: number;
  offsetY?: number;
}

const animals = ref<Animal[]>([]);

// 检查动物是否被遮挡
const isAnimalCovered = (targetAnimal: { x: number; y: number; zIndex: number }) => {
  const VISIBLE_THRESHOLD = 0.1; // 当动物被遮挡超过90%面积时判定为不可选中
  let totalOverlapArea = 0;
  const animalSize = 60;
  const animalArea = animalSize * animalSize;

  // 计算目标动物的边界框
  const targetLeft = (targetAnimal.x / 100) * window.innerWidth - animalSize / 2;
  const targetRight = targetLeft + animalSize;
  const targetTop = (targetAnimal.y / 100) * window.innerHeight - animalSize / 2;
  const targetBottom = targetTop + animalSize;

  // 检查所有可能遮挡的动物
  for (const animal of animals.value) {
    if (!animal.visible || animal.zIndex <= targetAnimal.zIndex) continue;

    // 计算遮挡动物的边界框
    const otherLeft = (animal.x / 100) * window.innerWidth - animalSize / 2;
    const otherRight = otherLeft + animalSize;
    const otherTop = (animal.y / 100) * window.innerHeight - animalSize / 2;
    const otherBottom = otherTop + animalSize;

    // 计算重叠区域
    const overlapLeft = Math.max(targetLeft, otherLeft);
    const overlapRight = Math.min(targetRight, otherRight);
    const overlapTop = Math.max(targetTop, otherTop);
    const overlapBottom = Math.min(targetBottom, otherBottom);

    if (overlapLeft < overlapRight && overlapTop < overlapBottom) {
      const overlapArea = (overlapRight - overlapLeft) * (overlapBottom - overlapTop);
      totalOverlapArea += overlapArea;

      // 如果被遮挡面积超过阈值，则认为动物不可选中
      if (totalOverlapArea >= animalArea * (1 - VISIBLE_THRESHOLD)) {
        return true;
      }
    }
  }

  return false; // 如果总遮挡面积小于阈值，则动物可选中
}

// 点击动物
// 计算两个点之间的距离
const calculateDistance = (x1: number, y1: number, x2: number, y2: number) => {
  return Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(y2 - y1, 2));
};

// 处理动物的鼠标悬停效果
const handleAnimalHover = (targetAnimal: { x: number; y: number; id: number; visible?: boolean }) => {
  const INFLUENCE_RADIUS = 15; // 影响半径（百分比）
  const MAX_OFFSET = 3; // 最大位移（百分比）

  animals.value.forEach(animal => {
    if (animal.id === targetAnimal.id || !animal.visible) return;

    const distance = calculateDistance(targetAnimal.x, targetAnimal.y, animal.x, animal.y);
    if (distance <= INFLUENCE_RADIUS) {
      const angle = Math.atan2(animal.y - targetAnimal.y, animal.x - targetAnimal.x);
      const offset = (1 - distance / INFLUENCE_RADIUS) * MAX_OFFSET;
      
      animal.offsetX = Math.cos(angle) * offset;
      animal.offsetY = Math.sin(angle) * offset;
    } else {
      animal.offsetX = 0;
      animal.offsetY = 0;
    }
  });
};

// 重置所有动物的位移
const resetAnimalOffsets = () => {
  animals.value.forEach(animal => {
    animal.offsetX = 0;
    animal.offsetY = 0;
  });
};

interface ClickableAnimal {
  id: number;
  type: string;
  visible?: boolean;
}

const handleAnimalClick = (animal: ClickableAnimal) => {
  if (!animal.visible || pendingArea.value.length >= 6 || isPaused.value) return;
  
  // 获取目标动物
  const targetAnimal = animals.value.find(a => a.id === animal.id);
  if (!targetAnimal) return;
  
  // 检查是否被遮挡
  if (isAnimalCovered(targetAnimal)) return;
  
  // 获取目标位置（待消除区域的空位）
  const emptySlotIndex = pendingArea.value.length;
  const targetSlot = document.querySelector(`.pending-slot:nth-child(${emptySlotIndex + 1})`);
  const targetRect = targetSlot?.getBoundingClientRect();
  const animalElement = document.querySelector(`[data-animal-id="${animal.id}"]`) as HTMLElement | null;
  
  if (targetRect && animalElement) {
    const startRect = animalElement.getBoundingClientRect();
    const startX = startRect.left;
    const startY = startRect.top;
    const endX = targetRect.left;
    const endY = targetRect.top;

    // 设置动画样式
    animalElement.style.position = 'fixed';
    animalElement.style.left = `${startX}px`;
    animalElement.style.top = `${startY}px`;
    animalElement.style.zIndex = '9999';
    
    // 使用线性动画
    const startTime = performance.now();
    const duration = 300; // 缩短动画时长到300ms以提升流畅度

    function animate(currentTime: number) {
      const elapsed = currentTime - startTime;
      const progress = Math.min(elapsed / duration, 1);

      // 使用线性动画
      const currentX = startX + (endX - startX) * progress;
      const currentY = startY + (endY - startY) * progress;

      // 添加缩放和旋转效果
      const scale = 1 + 0.2 * (1 - progress); // 开始时较大，逐渐恢复正常大小
      const rotation = progress * 360; // 完整旋转一圈

      if (!animalElement) return;
      
      animalElement.style.transform = `translate(-50%, -50%) scale(${scale}) rotate(${rotation}deg)`;
      animalElement.style.left = `${currentX}px`;
      animalElement.style.top = `${currentY}px`;

      if (progress < 1) {
        requestAnimationFrame(animate);
      } else {
        // 动画结束，重置变换
if (animalElement) {
  animalElement.style.transform = 'translate(-50%, -50%)';
}
      }
    }

    requestAnimationFrame(animate);
    
    // 动画结束后处理
    setTimeout(() => {
      // 恢复原始样式
      animalElement.style.position = '';
      animalElement.style.left = '';
      animalElement.style.top = '';
      animalElement.style.zIndex = '';
      animalElement.style.transition = '';
      
      // 添加到待消除区域
      pendingArea.value.push({ id: animal.id, type: animal.type });
      // 隐藏原位置的动物
      targetAnimal.visible = false;
    }, 300);
  } else {
    // 如果获取位置失败，直接添加到待消除区域
    pendingArea.value.push({ id: animal.id, type: animal.type });
    targetAnimal.visible = false;
  }
  
  // 在动画完成后检查是否可以消除和游戏是否结束
  setTimeout(() => {
    // 检查是否可以消除
    checkMatch();
    
    // 检查游戏是否结束
    checkGameOver();
  }, 350); // 设置延时稍大于动画时间，确保动画完成
}

// 检查匹配
const checkMatch = () => {
  // 统计每种动物的数量
  const counts = new Map<string, number>();
  pendingArea.value.forEach(animal => {
    counts.set(animal.type, (counts.get(animal.type) || 0) + 1);
  });

  // 检查每种动物是否达到3个
  for (const [type, count] of counts.entries()) {
    if (count >= 3) {
      // 找出所有匹配的动物
      const matchedAnimals = pendingArea.value.filter(animal => animal.type === type);
      
      // 获取要移除的动物ID
      const animalIdsToRemove = matchedAnimals.slice(0, 3).map(animal => animal.id);
      
      // 创建动画效果
      const animationPromises = animalIdsToRemove.map(id => {
        return new Promise<void>(resolve => {
          const elements = document.querySelectorAll(`img[data-animal-id="${id}"]`);
          elements.forEach(element => {
            if (element instanceof HTMLElement) {
              element.style.transition = 'all 0.5s ease-out';
              element.style.transform = 'scale(1.2) rotate(10deg)';
              element.style.opacity = '0';
            }
          });
          setTimeout(resolve, 500);
        });
      });

      // 等待所有动画完成后再更新状态
      Promise.all(animationPromises).then(() => {
        // 从待消除区域移除已消除的动物
        pendingArea.value = pendingArea.value.filter(animal => !animalIdsToRemove.includes(animal.id));
        
        // 检查层级可见性
        checkLayerVisibility();
        
        // 检查游戏状态
        const visibleAnimals = animals.value.filter(animal => animal.visible);
        if (visibleAnimals.length === 0 && pendingArea.value.length === 0) {
          endGame('win');
        } else {
          // 检查是否还有可以消除的组合
          setTimeout(() => {
            checkGameOver();
            checkMatch();
          }, 100);
        }
      });

      return; // 找到一组匹配后就退出当前检查
    }
  }
}

// 检查游戏是否结束
const checkGameOver = () => {
  // 如果待消除区域满了，且没有可以消除的组合，游戏结束
  if (pendingArea.value.length >= 6) {
    const counts = new Map<string, number>();
    pendingArea.value.forEach(animal => {
      counts.set(animal.type, (counts.get(animal.type) || 0) + 1);
    });
    
    let hasMatch = false;
    for (const count of counts.values()) {
      if (count >= 3) {
        hasMatch = true;
        break;
      }
    }
    
    if (!hasMatch) {
      endGame('full');
    }
  }
  
  // 检查是否获胜（所有动物都被消除）
  const visibleAnimals = animals.value.filter(animal => animal.visible);
  if (visibleAnimals.length === 0 && pendingArea.value.length === 0) {
    endGame('win');
  }
}

// 开始游戏
const startGame = () => {
  gameStarted.value = true
  gameOver.value = false
  timeLeft.value = 600
  // 初始化游戏区域
  initializeGame()
  // 启动计时器
  timer.value = setInterval(() => {
    if (!isPaused.value && timeLeft.value > 0) {
      timeLeft.value--
    } else if (timeLeft.value === 0) {
      endGame('timeout')
    }
  }, 1000)
}

// 检查层级可见性
const checkLayerVisibility = () => {
  // 当动物被消除后，检查并更新每个动物的可见性
  animals.value.forEach(animal => {
    if (!animal.visible) return;
    animal.visible = !isAnimalCovered(animal);
  });
};

// 初始化游戏
const initializeGame = () => {
  // 清空现有动物
  animals.value = []
  pendingArea.value = []
  
  // 计算布局参数
  interface AreaSize {
    width: number;
    height: number;
  }
  interface Margin {
    left: number;
    top: number;
    right: number;
    bottom: number;
  }
  interface Position {
    x: number;
    y: number;
    layer: number;
  }

  const areaSize: AreaSize = {
    width: 70,    // 保持宽度不变
    height: 25    // 保持高度不变
  }
  const margin: Margin = {
    left: 15,     // 保持左边距不变
    top: 35,      // 将上边距从38%减少到35%，使区域整体上移
    right: 85,    // 保持右边界不变
    bottom: 60    // 相应调整下边界
  }
  const layers = 6
  
  // 生成初始动物布局
  const numAnimalsPerLayer = 18  // 保持每层动物数量不变
  const positions: Position[] = []
  
  // 为每层生成随机位置
  for (let layer = 0; layer < layers; layer++) {
    for (let i = 0; i < numAnimalsPerLayer; i++) {
      // 在调整后的区域内生成随机位置
      const x = margin.left + Math.random() * areaSize.width
      const y = margin.top + Math.random() * areaSize.height
      
      positions.push({
        x,
        y,
        layer
      })
    }
  }
  
  // 随机打乱每层的位置
  for (let layer = 0; layer < layers; layer++) {
    const layerPositions = positions.filter(pos => pos.layer === layer)
    for (let i = layerPositions.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1))
      const temp = layerPositions[i]
      layerPositions[i] = layerPositions[j]
      layerPositions[j] = temp
    }
  }
  
  // 生成动物
  let id = 0
  for (let layer = 0; layer < layers; layer++) {
    const layerPositions = positions.filter(pos => pos.layer === layer)
    for (let i = 0; i < Math.min(numAnimalsPerLayer, layerPositions.length); i++) {
      const pos = layerPositions[i]
      const animal = {
        id: id++,
        type: animalTypes[Math.floor(Math.random() * animalTypes.length)],
        x: pos.x,
        y: pos.y,
        visible: true,
        zIndex: layer * 100 + i // 使用层级作为基础z-index
      }
      animals.value.push(animal)
    }
  }
}

// 返回主页
const handleReturnClick = () => {
  showConfirmDialog.value = true
}

// 确认返回主页
const returnToHome = () => {
  showConfirmDialog.value = false
  gameStarted.value = false
  gameOver.value = false
  if (timer.value) {
    clearInterval(timer.value)
    timer.value = null
  }
  animals.value = []
  pendingArea.value = []
}

// 取消返回
const cancelReturn = () => {
  showConfirmDialog.value = false
}

// 结束游戏
const endGame = (reason: 'timeout' | 'full' | 'win') => {
  gameOver.value = true
  gameResult.value = reason
  if (timer.value) {
    clearInterval(timer.value)
    timer.value = null
  }
}

// 组件卸载时清理计时器
onUnmounted(() => {
  if (timer.value) {
    clearInterval(timer.value)
  }
})
</script>

<template>
  <div class="game-container">
    <!-- 首页 -->
    <div v-if="!gameStarted" class="start-screen">
      <div class="start-content">
        <img src="https://img.z4a.net/images/2025/01/22/b1dd4ee8071e03de4d52889e6ec05932.png" alt="擒兽记" class="game-title" />
        <p class="game-desc">收集相同的动物，三个一组可以消除。看看你能坚持多久！</p>
        <button @click="startGame" class="start-btn">开始游戏</button>
      </div>
    </div>

    <!-- 游戏中页面 -->
    <div v-else class="game-area">
      <div class="game-header">
        <div class="timer">{{ Math.floor(timeLeft / 60) }}:{{ (timeLeft % 60).toString().padStart(2, '0') }}</div>
        <button class="pause-btn" @click="isPaused = !isPaused">{{ isPaused ? '继续' : '暂停' }}</button>
        <button class="return-btn" @click="handleReturnClick">返回主页</button>
      </div>

      <div class="main-area">
        <div
          v-for="animal in animals"
          :key="animal.id"
          class="animal"
          :data-animal-id="animal.id"
          :style="{
            left: `${animal.x + (animal.offsetX || 0)}%`,
            top: `${animal.y + (animal.offsetY || 0)}%`,
            zIndex: animal.zIndex,
            display: animal.visible ? 'flex' : 'none'
          }"
          @mouseenter="handleAnimalHover(animal)"
          @mouseleave="resetAnimalOffsets"
          @click="handleAnimalClick(animal)"
        >
          <img :src="animalImages[animal.type as keyof typeof animalImages]" :alt="animal.type" :style="animalImageStyle" />
        </div>
      </div>

      <div class="pending-area">
        <div
          v-for="index in 6"
          :key="index"
          class="pending-slot"
        >
          <template v-if="pendingArea[index]">
            <img :src="animalImages[pendingArea[index].type as keyof typeof animalImages]" :alt="pendingArea[index].type" :style="animalImageStyle" />
          </template>
        </div>
      </div>
    </div>

    <!-- 游戏结束页面 -->
    <div v-if="gameOver" class="game-over-overlay">
      <div class="game-over">
      <h2 :class="['game-over-title', gameResult]">
        {{ gameResult === 'win' ? '恭喜过关！' :
           gameResult === 'timeout' ? '时间到！' :
           gameResult === 'full' ? '格子已满！' : '游戏结束' }}
      </h2>
      <p class="game-result" v-if="gameResult === 'win'">
        恭喜你成功通关！你用时 {{ 600 - timeLeft }} 秒完成了游戏。
      </p>
      <p class="game-result" v-else-if="gameResult === 'timeout'">
        时间已用完，再接再厉！
      </p>
      <p class="game-result" v-else-if="gameResult === 'full'">
        格子已满，游戏结束！
      </p>
      <div class="game-over-buttons">
        <button @click="startGame" class="restart-btn">重新开始</button>
        <button @click="returnToHome" class="home-btn">返回主页</button>
      </div>
    </div>
    </div>

    <!-- 确认弹窗 -->
    <div v-if="showConfirmDialog" class="confirm-dialog">
      <div class="confirm-dialog-content">
        <p>返回主页？游戏将会终止</p>
        <div class="confirm-dialog-buttons">
          <button @click="returnToHome" class="confirm-btn">确认</button>
          <button @click="cancelReturn" class="cancel-btn">取消</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.game-container {
  width: 100%;
  height: 100vh;
  margin: 0;
  padding: 10px;
  display: flex;
  justify-content: center;
  align-items: center;
  background-image: url('https://img.z4a.net/images/2025/01/22/eb0a93850b3bc42a6235b0da4a60554c.png');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  background-color: #f5f5f5;
  box-sizing: border-box;
  overflow: hidden;
  position: fixed;
  top: 0;
  left: 0;
}

.game-area {
  position: relative;
  display: flex;
  flex-direction: column;
  gap: 10px;
  width: 414px;
  height: 896px;
  max-width: 100vw;
  max-height: 100vh;
}

.main-area {
  flex: 1;
  position: relative;
  height: calc(100% - 120px);
  margin-top: 40px;
}

.animal {
  position: absolute;
  width: 60px;
  height: 60px;
  background-color: transparent;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transform: translate(-50%, -50%);
  transition: transform 0.2s, left 0.3s ease-out, top 0.3s ease-out;
  overflow: hidden;
  touch-action: manipulation;
}

.pending-area {
  width: 100%;
  display: grid;
  grid-template-columns: repeat(6, 1fr);
  gap: 4px;
  padding: 4px;
  height: 60px;
  position: relative;
  transform: translateY(-100px);
}

.pending-slot {
  aspect-ratio: 1;
  border: 5px solid #453D38;
  border-radius: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 2px;
  width: 45px;
  height: 45px;
  background: #FFF;
  box-shadow: 0px 6px 0px 0px rgba(0, 0, 0, 0.13), 0px -6px 0px 0px #A4B3C7 inset;
}

.pending-slot img {
  width: 100%;
  height: 100%;
  object-fit: contain;
  display: block;
}

.start-screen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-image: url('https://img.z4a.net/images/2025/01/22/-.png');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.start-content {
  transform: translateY(-100px);
}

.game-over-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.game-over {
  position: relative;
  transform: none;
  text-align: center;
  padding: 40px;
  background-color: rgba(255, 255, 255, 0.95);
  border-radius: 12px;
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.15);
  min-width: 300px;
}

.game-over-title {
  font-size: 32px;
  margin-bottom: 20px;
  font-weight: bold;
}

.game-over-title.win {
  color: #42b883;
}

.game-over-title.timeout,
.game-over-title.full {
  color: #e74c3c;
}

.game-over-buttons {
  display: flex;
  justify-content: center;
  gap: 15px;
  margin-top: 25px;
}

button {
  padding: 12px 24px;
  font-size: 18px;
  background-color: #42b883;
  color: white;
  border: none;
  border-radius: 50px;
  cursor: pointer;
  transition: background-color 0.3s;
}

button:hover {
  background-color: #3aa876;
}

.game-title {
  width: 270px;
  height: auto;
  position: relative;
  z-index: 1;
  margin: 0 auto 20px;
  display: block;
}

.game-desc {
  font-size: 18px;
  color: #666;
  margin-bottom: 30px;
  line-height: 1.5;
}

.start-btn {
  font-size: 24px;
  padding: 15px 40px;
}

.game-header {
  position: relative;
  width: 100%;
  padding: 10px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: calc(14px + 1vmin);
  background-color: rgba(255, 255, 255, 0.8);
  border-radius: 8px;
  margin-bottom: 10px;
}

.pause-btn {
  padding: 8px 16px;
  font-size: 16px;
  background-color: #42b883;
  margin: 0 10px;
}

.game-result {
  font-size: 20px;
  color: #666;
  margin: 15px 0;
}

.restart-btn,
.home-btn {
  margin: 10px;
  min-width: 120px;
}

.home-btn {
  background-color: #666;
}

.home-btn:hover {
  background-color: #555;
}

.confirm-dialog {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.confirm-dialog-content {
  background-color: white;
  padding: 30px;
  border-radius: 8px;
  text-align: center;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.confirm-dialog-content p {
  margin-bottom: 20px;
  font-size: 18px;
  color: #333;
}

.confirm-dialog-buttons {
  display: flex;
  justify-content: center;
  gap: 15px;
}

.confirm-btn {
  background-color: #42b883;
}

.cancel-btn {
  background-color: #666;
}

.cancel-btn:hover {
  background-color: #555;
}
</style>
