//需要安装依赖：npm install vuedraggable@next
<template>
  <div class="trip-planner-container">
    <div class="plan-hint">
      ↺ 重置当天行程<br>
      🗑 删除当天便签
      <br><br>
      🖱️ 拖拽行程改变顺序和日期<br>
      ✏️ 点击行程编辑内容
    </div>
    <h1>行程规划</h1>
    <div class="input-row">
      <label>计划旅行天数：</label>
      <input type="number" v-model.number="days" min="1" max="30" />
      <span class="hint"></span>
      <button @click="clearAllPlans" class="clear-all-btn">清空所有行程</button>
    </div>
    <div class="tip-text">点击便签底部为每日增加行程</div>
    <div class="notes-row" ref="planNotesContainer">
      <div
        v-for="(day, idx) in dayPlans"
        :key="idx"
        :class="['day-note', dayNoteColorClass(idx)]"
      >
        <div class="note-header">
          <span>Day {{ idx + 1 }}</span>
          <span class="icon-btn reset-btn" title="重置" @click="clearDay(idx)">↺</span>
          <span class="icon-btn delete-note-btn" title="删除便签" @click="deleteDay(idx)">🗑</span>
        </div>
        <draggable
          v-model="dayPlans[idx]"
          :group="'plans'"
          item-key="item"
          class="plan-list"
        >
          <template #item="{ element, index }">
            <div class="plan-item">
              <template v-if="editingIdx.day === idx && editingIdx.plan === index">
                <input
                  :id="`edit-input-${idx}-${index}`"
                  v-model="editingValue"
                  @keyup.enter="saveEdit(idx, index)"
                  @blur="saveEdit(idx, index)"
                  class="edit-plan-input"
                />
              </template>
              <template v-else>
                <span @click="startEdit(idx, index, element)" class="editable-plan">{{ index + 1 }}. {{ element }}</span>
                <span class="icon-btn del-item-btn" title="删除行程" @click="deletePlan(idx, index)">×</span>
              </template>
            </div>
          </template>
        </draggable>
        <div v-if="showInputIdx === idx" class="add-row">
          <div class="plan-input-wrapper">
            <input
              :ref="setAddInputRef(idx)"
              v-model="newPlan[idx]"
              @keydown.enter.prevent="confirmAddPlan(idx)"
              placeholder="添加行程..."
              @focus="showPlanSpotDropdown = true"
              @blur="hidePlanSpotDropdown"
            />
             <div v-if="showPlanSpotDropdown && savedSpots && savedSpots.length > 0" class="spots-dropdown">
              <div
                v-for="spot in savedSpots"
                :key="spot.id"
                class="spot-item"
                @mousedown.prevent="selectPlanSpot(idx, spot.name)"
              >
                <i class="fas fa-map-marker-alt"></i> {{ spot.name }}
              </div>
            </div>
          </div>
          <button @click="confirmAddPlan(idx)"><span class="confirm-text">确&nbsp;认</span></button>
        </div>
        <button :class="['add-btn', addBtnColorClass(idx)]" @click="showAddInput(idx)">＋</button>
      </div>
    </div>
    
    <!-- 导出按钮在页面底部，与标题竖直对齐 -->
    <div class="export-section-bottom">
      <button @click="exportToPDF" class="export-btn" :disabled="isExporting">
        {{ isExporting ? '导出中...' : '导出' }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, nextTick, inject, onMounted, onBeforeUnmount } from 'vue'
import draggable from 'vuedraggable'
import html2canvas from 'html2canvas'
import jsPDF from 'jspdf'

// 注入提供的savedSpots引用
const savedSpots = inject('savedSpots', []);

const days = ref(1)
const dayPlans = ref([[]])
const newPlan = ref([''])
const showInputIdx = ref(-1)
const addInputRefs = ref([])
const editingIdx = ref({ day: -1, plan: -1 })
const editingValue = ref('')
const showPlanSpotDropdown = ref(false);

const isExporting = ref(false)
const planNotesContainer = ref(null);

// 从本地存储加载数据
onMounted(() => {
  const savedDays = localStorage.getItem('tripPlanDays')
  const savedDayPlans = localStorage.getItem('tripPlanData')
  
  if (savedDays) {
    days.value = parseInt(savedDays)
  }
  
  if (savedDayPlans) {
    try {
      dayPlans.value = JSON.parse(savedDayPlans)
      // 确保newPlan数组长度与dayPlans匹配
      if (newPlan.value.length < dayPlans.value.length) {
        for (let i = newPlan.value.length; i < dayPlans.value.length; i++) {
          newPlan.value.push('')
        }
      }
    } catch (e) {
      console.error('加载行程数据失败:', e)
    }
  }
})

// 保存数据到本地存储
function savePlansToStorage() {
  localStorage.setItem('tripPlanDays', days.value.toString())
  localStorage.setItem('tripPlanData', JSON.stringify(dayPlans.value))
}

// 监听dayPlans变化，保存到本地存储
watch(dayPlans, () => {
  savePlansToStorage()
}, { deep: true })

// 监听days变化，保存到本地存储
watch(days, () => {
  savePlansToStorage()
})


function setAddInputRef(idx) {
  return (el) => {
    addInputRefs.value[idx] = el
  }
}

watch(days, (val) => {
  if (val === null || val === undefined || val === '') {
    dayPlans.value = [];
    newPlan.value = [];
    return;
  }
  if (val < 1) days.value = 1;
  if (val > dayPlans.value.length) {
    for (let i = dayPlans.value.length; i < val; i++) {
      dayPlans.value.push([]);
      newPlan.value.push('');
    }
  } else {
    dayPlans.value.length = val;
    newPlan.value.length = val;
  }
})

function showAddInput(idx) {
  showInputIdx.value = idx
  newPlan.value[idx] = ''
  nextTick(() => {
    if (addInputRefs.value[idx]) {
      addInputRefs.value[idx].focus()
    }
  })
}
function confirmAddPlan(idx) {
  console.log('confirmAddPlan called', idx);
  const plan = newPlan.value[idx]?.trim()
  if (plan) {
    dayPlans.value[idx].push(plan)
    newPlan.value[idx] = ''
    showInputIdx.value = -1
    showPlanSpotDropdown.value = false;
  }
}
function deletePlan(dayIdx, planIdx) {
  dayPlans.value[dayIdx].splice(planIdx, 1)
}
function clearDay(idx) {
  dayPlans.value[idx] = []
}
function deleteDay(idx) {
  dayPlans.value.splice(idx, 1)
  newPlan.value.splice(idx, 1)
  days.value = dayPlans.value.length
  if (showInputIdx.value === idx) showInputIdx.value = -1
}
function dayNoteColorClass(idx) {
  const day = idx % 3;
  if (day === 0) return 'day-note-yellow';
  if (day === 1) return 'day-note-blue';
  return 'day-note-pink';
}
function addBtnColorClass(idx) {
  const day = idx % 3;
  if (day === 0) return 'add-btn-yellow';
  if (day === 1) return 'add-btn-blue';
  return 'add-btn-pink';
}
function startEdit(dayIdx, planIdx, val) {
  editingIdx.value = { day: dayIdx, plan: planIdx }
  editingValue.value = val
  nextTick(() => {
    const input = document.getElementById(`edit-input-${dayIdx}-${planIdx}`)
    if (input) input.focus()
  })
}
function saveEdit(dayIdx, planIdx) {
  if (editingValue.value.trim()) {
    dayPlans.value[dayIdx][planIdx] = editingValue.value.trim()
  }
  editingIdx.value = { day: -1, plan: -1 }
  editingValue.value = ''
}

function selectPlanSpot(idx, spotName) {
  console.log('selectPlanSpot called', idx, spotName);
  newPlan.value[idx] = spotName;
  // 直接调用confirmAddPlan来添加行程
  confirmAddPlan(idx);
}

function hidePlanSpotDropdown() {
  // 增加延迟，避免与点击事件冲突
  setTimeout(() => {
    // 只有在没有选择项目时才关闭下拉菜单
    if (!newPlan.value[showInputIdx.value]?.trim()) {
      showPlanSpotDropdown.value = false;
    }
  }, 300);
}

function clearAllPlans() {
  if (confirm('确定要清空所有行程数据吗？此操作不可恢复！')) {
    days.value = 1;
    dayPlans.value = [[]];
    newPlan.value = [''];
    localStorage.removeItem('tripPlanDays');
    localStorage.removeItem('tripPlanData');
  }
}

// PDF导出功能
async function exportToPDF() {
  if (isExporting.value) return
  
  try {
    isExporting.value = true
    
    // 确保没有正在编辑的状态
    if (editingIdx.value.day !== -1) {
      editingIdx.value = { day: -1, plan: -1 }
      editingValue.value = ''
    }
    
    // 隐藏添加行程的输入框
    const originalShowInputIdx = showInputIdx.value
    showInputIdx.value = -1
    
    await nextTick()
    
    // 创建用于导出的内容容器
    const exportContainer = createExportContainer()
    document.body.appendChild(exportContainer)
    
    await nextTick()
    
    // 使用html2canvas截图
    const canvas = await html2canvas(exportContainer, {
      scale: 2,
      useCORS: true,
      allowTaint: true,
      backgroundColor: '#ffffff'
    })
    
    // 移除临时容器
    document.body.removeChild(exportContainer)
    
    // 创建PDF
    const pdf = new jsPDF('p', 'mm', 'a4')
    const pdfWidth = pdf.internal.pageSize.getWidth()
    const pdfHeight = pdf.internal.pageSize.getHeight()
    
    const imgWidth = pdfWidth
    const imgHeight = (canvas.height * pdfWidth) / canvas.width
    
    if (imgHeight <= pdfHeight) {
      // 单页显示
      pdf.addImage(canvas.toDataURL('image/png'), 'PNG', 0, 0, imgWidth, imgHeight)
    } else {
      // 多页显示
      let yPosition = 0
      const pageHeight = pdfHeight
      
      while (yPosition < imgHeight) {
        // 创建当前页的canvas
        const pageCanvas = document.createElement('canvas')
        const pageCtx = pageCanvas.getContext('2d')
        
        pageCanvas.width = canvas.width
        pageCanvas.height = (canvas.width * pageHeight) / imgWidth
        
        // 绘制当前页内容
        pageCtx.drawImage(
          canvas,
          0, (yPosition * canvas.width) / imgWidth,
          canvas.width, pageCanvas.height,
          0, 0,
          canvas.width, pageCanvas.height
        )
        
        if (yPosition > 0) {
          pdf.addPage()
        }
        
        pdf.addImage(pageCanvas.toDataURL('image/png'), 'PNG', 0, 0, imgWidth, pageHeight)
        yPosition += pageHeight
      }
    }
    
    // 恢复原始状态
    showInputIdx.value = originalShowInputIdx
    
    // 保存PDF
    const fileName = `旅行行程规划_${new Date().toLocaleDateString().replace(/\//g, '-')}.pdf`
    pdf.save(fileName)
    
  } catch (error) {
    console.error('导出PDF时发生错误:', error)
    alert('导出PDF时发生错误，请重试')
  } finally {
    isExporting.value = false
  }
}

// 创建用于导出的内容容器
function createExportContainer() {
  const container = document.createElement('div')
  container.style.cssText = `
    position: absolute;
    left: -9999px;
    top: 0;
    width: 800px;
    background: #ffffff;
    padding: 40px;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  `
  
  // 添加标题
  const title = document.createElement('h1')
  title.textContent = '旅行行程规划'
  title.style.cssText = `
    text-align: center;
    font-size: 36px;
    color: #2d3a4b;
    margin-bottom: 40px;
    letter-spacing: 2px;
    font-weight: bold;
  `
  container.appendChild(title)
  
  // 创建便签网格
  const notesGrid = document.createElement('div')
  notesGrid.style.cssText = `
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 20px;
    margin-bottom: 20px;
  `
  
  dayPlans.value.forEach((dayPlan, idx) => {
    const dayNote = createDayNoteElement(idx + 1, dayPlan)
    notesGrid.appendChild(dayNote)
  })
  
  container.appendChild(notesGrid)
  return container
}

// 创建单个便签元素
function createDayNoteElement(dayNumber, plans) {
  const noteDiv = document.createElement('div')
  const colorIndex = (dayNumber - 1) % 3
  const backgrounds = ['#fffbe6', '#f0f7ff', '#fff0f6']
  
  noteDiv.style.cssText = `
    background: ${backgrounds[colorIndex]};
    border-radius: 20px;
    border: 2px solid #e0e0e0;
    padding: 20px;
    min-height: 200px;
    break-inside: avoid;
  `
  
  // 添加标题
  const header = document.createElement('div')
  header.textContent = `Day ${dayNumber}`
  header.style.cssText = `
    background: #fff;
    border-radius: 10px;
    padding: 8px 16px;
    margin-bottom: 16px;
    font-weight: bold;
    font-size: 18px;
    color: #2d3a4b;
    text-align: center;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  `
  noteDiv.appendChild(header)
  
  // 添加行程列表
  plans.forEach((plan, index) => {
    const planItem = document.createElement('div')
    planItem.textContent = `${index + 1}. ${plan}`
    planItem.style.cssText = `
      background: #fff;
      border-radius: 8px;
      margin-bottom: 8px;
      padding: 8px 12px;
      font-size: 14px;
      color: #3a4a5b;
      box-shadow: 0 1px 3px rgba(0,0,0,0.1);
      line-height: 1.4;
    `
    noteDiv.appendChild(planItem)
  })
  
  return noteDiv
}


</script>

<style scoped>
.plan-hint {
  position: absolute;
  right: 32px;
  top: 18px;
  color: #bbb;
  font-size: 13px;
  z-index: 2;
  line-height: 1.7;
}
.trip-planner-container {
  padding: 32px 32px 0 32px;
  background: #ffffff;
  min-height: 100vh;
  position: relative;
}
h1 {
  text-align: center;
  font-size: 2.3rem;
  color: #2d3a4b;
  margin-bottom: 18px;
  letter-spacing: 2px;
}
.input-row {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-bottom: 18px;
  font-size: 22px;
  font-weight: bold;
}
.input-row input[type="number"] {
  width: 90px;
  height: 44px;
  font-size: 22px;
  padding: 4px 10px;
  border-radius: 10px;
  border: 1.5px solid #b0c4de;
  margin-left: 12px;
  margin-right: 8px;
  background: #fafdff;
  box-shadow: 0 1px 4px rgba(180,200,220,0.08);
}
.hint {
  color: #888;
  font-size: 15px;
  margin-left: 8px;
}
.export-section-bottom {
  display: flex;
  justify-content: center;
  margin-top: 40px;
  margin-bottom: 32px;
}
.export-btn {
  background: linear-gradient(135deg, #4CAF50 0%, #45a049 100%);
  color: white;
  border: none;
  padding: 12px 24px;
  font-size: 16px;
  font-weight: bold;
  border-radius: 25px;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 4px 15px rgba(76, 175, 80, 0.3);
  letter-spacing: 1px;
}
.export-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, #45a049 0%, #3d8b40 100%);
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(76, 175, 80, 0.4);
}
.export-btn:disabled {
  background: #cccccc;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}
.tip-text {
  color: #7b8fa6;
  font-size: 19px;
  margin-bottom: 22px;
  font-weight: bold;
  text-align: center;
}
.notes-row {
  display: flex;
  flex-wrap: wrap;
  gap: 40px;
  justify-content: center;
}
.day-note {
  background: #fffbe6;
  border-radius: 24px;
  box-shadow: 0 4px 18px 0 rgba(180,200,220,0.13);
  border: none;
  width: 260px;
  min-height: 380px;
  padding: 20px 16px 24px 16px;
  display: flex;
  flex-direction: column;
  align-items: center;
  position: relative;
  margin-bottom: 24px;
  transition: box-shadow 0.2s, transform 0.2s;
}
.day-note-yellow { background: #fffbe6; }
.day-note-blue { background: #f0f7ff; }
.day-note-pink { background: #fff0f6; }
.day-note:hover {
  box-shadow: 0 8px 32px 0 rgba(120,160,200,0.18);
  transform: translateY(-4px) scale(1.03);
}
.note-header {
  background: #fff;
  border-radius: 10px;
  padding: 8px 22px;
  margin-bottom: 16px;
  font-weight: bold;
  font-size: 1.2rem;
  color: #2d3a4b;
  box-shadow: 0 1px 4px rgba(180,200,220,0.08);
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
}
.icon-btn {
  background: #f3f3f3;
  border: none;
  color: #bbb;
  font-size: 1.1em;
  border-radius: 6px;
  margin-left: 6px;
  padding: 2px 8px;
  cursor: pointer;
  transition: background 0.2s, color 0.2s;
}
.icon-btn:hover {
  background: #e0e0e0;
  color: #1976d2;
}
.plan-list {
  width: 100%;
  min-height: 40px; /* Ensure there is space for dragging into empty lists */
}
.plan-item {
  background: #fff;
  border-radius: 8px;
  margin-bottom: 8px;
  padding: 8px 12px;
  box-shadow: 0 1px 4px rgba(180,200,220,0.08);
  font-size: 1.08rem;
  color: #3a4a5b;
  cursor: grab;
  border: 1px solid #e0e0e0;
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.plan-item:last-child {
  margin-bottom: 0;
}
.del-item-btn {
  background: none;
  border: none;
  color: #bbb;
  font-size: 1.2em;
  margin-left: 8px;
  cursor: pointer;
  transition: color 0.2s;
}
.del-item-btn:hover {
  color:rgba(235, 140, 150, 0.85);
}
.add-row {
  display: flex;
  align-items: center;
  justify-content: space-between; /* 水平分布 */
  gap: 6px;
  margin-bottom: 8px;
  width: 100%;
  padding: 0 8px; /* 添加左右内边距 */
}
.plan-input-wrapper {
  flex: 1;
  position: relative; /* Needed for absolute positioning of the dropdown */
}
.add-row input {
  width: 100%;
  padding: 6px 8px;
  border-radius: 6px;
  border: 1px solid #bbb;
  font-size: 0.9rem; /* 调小字体 */
}
.add-row button {
  background: #f3f3f3;
  border: none;
  border-radius: 6px; /* 调整圆角 */
  padding: 6px 12px;  /* 调整内边距 */
  font-size: 0.8rem;  /* 调小字体 */
  color: #2d3a4b;
  cursor: pointer;
  transition: background 0.2s, color 0.2s;
  margin-left: 4px; /* 添加间距 */
}
.add-row button:hover {
  background: #1976d2;
  color: #fff;
}
.spots-dropdown {
  position: absolute;
  background: #fff;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  max-height: 150px;
  overflow-y: auto;
  z-index: 10;
  width: 100%;
  margin-top: 4px;
}
.spots-dropdown .spot-item {
  padding: 8px 12px;
  cursor: pointer;
  font-size: 0.9rem;
  color: #333;
  display: flex;
  align-items: center;
}
.spots-dropdown .spot-item:hover {
  background: #f0f0f0;
}
.confirm-text {
  letter-spacing: 0.2em;
  font-weight: bold;
}
.add-btn {
  margin-top: auto;
  font-size: 32px;
  background: linear-gradient(135deg, #e3f0fa 60%, #c3d7de 100%);
  border: 2.5px solid #7bb1d1;
  color: #3a6ea5;
  cursor: pointer;
  transition: color 0.2s, border-color 0.2s, background 0.2s, box-shadow 0.2s;
  width: 56px;
  height: 56px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  box-sizing: border-box;
  line-height: 1;
  padding: 0;
  box-shadow: 0 2px 8px rgba(120,160,200,0.10);
}
.add-btn:hover {
  color: #fff;
  border-color: #1976d2;
  background: linear-gradient(135deg, #7bb1d1 60%, #1976d2 100%);
  box-shadow: 0 4px 16px rgba(25,118,210,0.18);
}
.day-note-yellow { background: #fffbe6; }
.day-note-blue { background: #f0f7ff; }
.day-note-pink { background: #fff0f6; }
.add-btn-yellow {
  background: linear-gradient(135deg, #fffbe6 60%, #ffe6b3 100%);
  border-color: #ffe6b3;
  color: #e6a23c;
}
.add-btn-yellow:hover {
  color: #fff;
  border-color: #e6a23c;
  background: linear-gradient(135deg, #ffe6b3 60%, #e6a23c 100%);
}
.add-btn-blue {
  background: linear-gradient(135deg, #f0f7ff 60%, #b3e0ff 100%);
  border-color: #b3e0ff;
  color: #409eff;
}
.add-btn-blue:hover {
  color: #fff;
  border-color: #409eff;
  background: linear-gradient(135deg, #b3e0ff 60%, #409eff 100%);
}
.add-btn-pink {
  background: linear-gradient(135deg, #fff0f6 60%, #ffb3c6 100%);
  border-color: #ffb3c6;
  color: #e35d6a;
}
.add-btn-pink:hover {
  color: #fff;
  border-color: #e35d6a;
  background: linear-gradient(135deg, #ffb3c6 60%, #e35d6a 100%);
}
.edit-plan-input {
  flex: 1;
  padding: 6px 8px;
  border-radius: 6px;
  border: 1px solid #bbb;
  font-size: 0.9rem;
  max-width: 150px;
}
.editable-plan {
  cursor: pointer;
  transition: color 0.2s;
}
.editable-plan:hover {
  color: #1976d2;
}

.clear-all-btn {
  background: rgba(235, 140, 150, 0.15);
  color: #7b8fa6;
  border: 1px solid rgba(235, 140, 150, 0.3);
  border-radius: 8px;
  padding: 8px 16px;
  font-size: 0.9rem;
  font-weight: 500;
  cursor: pointer;
  margin-left: 16px;
  transition: all 0.2s ease;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
}

.clear-all-btn:hover {
  background: rgba(235, 140, 150, 0.25);
  color: #e35d6a;
  border-color: rgba(235, 140, 150, 0.5);
  transform: translateY(-1px);
}

@media (max-width: 900px) {
  .notes-row {
    gap: 18px;
  }
  .day-note {
    width: 98vw;
    min-width: 220px;
    max-width: 98vw;
  }
}
@media (max-width: 600px) {
  .notes-row {
    flex-direction: column;
    align-items: center;
    gap: 12px;
  }
  .day-note {
    width: 98vw;
    min-width: 180px;
    max-width: 98vw;
    padding: 12px 6px 18px 6px;
  }
  .note-header {
    font-size: 1rem;
    padding: 6px 10px;
  }
  .trip-planner-container {
    padding: 20px 16px 0 16px;
  }
  .export-btn {
    padding: 10px 20px;
    font-size: 14px;
  }
  h1 {
    font-size: 1.8rem;
  }
}
</style>
