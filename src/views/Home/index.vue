<script setup lang="ts">
// TODO 上一步、下一步 (历史记录、队列实现)
// TODO 屏幕尺寸发生变化时，canvas需要重绘（节流）
import { onBeforeUnmount, onMounted, ref, watch } from "vue";
import { useWindowSize } from "@vueuse/core";
// import { fabric } from "fabric";
import { fabric } from "fabric-with-erasing";
import {
  Undo2,
  MousePointer2,
  Hand,
  Pencil,
  Type,
  Squircle,
  Redo2,
  Eraser,
  Square,
  Circle,
  Triangle,
} from "lucide-vue-next";
import { Button } from "@/components/ui/button";
import { Slider } from "@/components/ui/slider";
import type { ShapeName, ToolBarItemName } from "./types";

const { width, height } = useWindowSize();

let canvas: fabric.Canvas | null = null;

const isDragging = ref(false);
const lastPosX = ref(0);
const lastPosY = ref(0);

// 是否展示画笔参数工具栏
const isShowPencilToolbar = ref(false);
// 是否展示形状工具栏
const isShowShapeToolbar = ref(false);

/**
 * @description 初始化canvas编辑器
 */
function initCanvasEditor() {
  canvas = new fabric.Canvas("canvasRef", {
    width: width.value,
    height: height.value,
    backgroundColor: "#f1f5f9",
  });
  canvas.on("mouse:wheel", canvasMouseWheelHandler);
}

/**
 * @description 画布上的鼠标滚轮事件，实现画布缩放
 * @param opt
 */
function canvasMouseWheelHandler(opt: fabric.IEvent<any>) {
  let delta = opt.e.deltaY;
  let zoom = canvas!.getZoom();
  zoom *= 0.999 ** delta;
  if (zoom > 20) zoom = 20;
  if (zoom < 0.01) zoom = 0.01;
  canvas!.setZoom(zoom);
  // console.log(Math.ceil(zoom * 100));

  opt.e.preventDefault();
  opt.e.stopPropagation();
}

/**
 * @description alt + 鼠标组合 拖动时的鼠标移动处理函数
 * @param opt
 */
function canvasMouseMoveHandler(opt: fabric.IEvent<any>) {
  if (!isDragging.value) return;

  const e = opt.e;
  const vpt = canvas!.viewportTransform;

  vpt![4] += e.clientX - lastPosX.value;
  vpt![5] += e.clientY - lastPosY.value;

  canvas?.requestRenderAll();
  lastPosX.value = e.clientX;
  lastPosY.value = e.clientY;
}

/**
 * @description alt + 鼠标组合 鼠标按下的事件处理函数
 * @param opt
 */
function canvasMouseDownHandler(opt: fabric.IEvent<any>) {
  const evt = opt.e;
  // if (evt.altKey) {
  isDragging.value = true;
  canvas!.selection = false;
  lastPosX.value = evt.clientX;
  lastPosY.value = evt.clientY;
  // }
}

/**
 * @description alt + 鼠标组合 鼠标抬起事件处理函数
 */
function canvasMouseUpHandler() {
  canvas?.setViewportTransform(canvas.viewportTransform!);
  isDragging.value = false;
  canvas!.selection = true;
}

/**
 * @description 光标模式
 */
function mouseToggleHandler() {
  if (canvas) {
    canvas.isDrawingMode = false;
    canvas.defaultCursor = "default";
    canvas.off("mouse:down", canvasMouseDownHandler);
    canvas.off("mouse:up", canvasMouseUpHandler);
    canvas.off("mouse:move", canvasMouseMoveHandler);
    canvas.off("mouse:down", createITextHandler);
    // shape
    canvas.off("mouse:down", shapeMouseDownHandler);
    canvas.off("mouse:move", shapeMouseMoveHandler);
    canvas.off("mouse:up", shapeMouseUpHandler);
  }
}

/**
 * @description 🖐🏻模式
 */
function handToggleHandler() {
  if (canvas) {
    canvas.isDrawingMode = false;
    canvas.defaultCursor = "move";
    canvas.on("mouse:down", canvasMouseDownHandler);
    canvas.on("mouse:up", canvasMouseUpHandler);
    canvas.on("mouse:move", canvasMouseMoveHandler);
  }
}

/**
 * @description 🖌模式
 */
function pencilToggleHandler() {
  // 开启自由绘画模式
  if (canvas) {
    canvas.off("mouse:down", canvasMouseDownHandler);
    canvas.off("mouse:up", canvasMouseUpHandler);
    canvas.off("mouse:move", canvasMouseMoveHandler);

    canvas.isDrawingMode = true;
    updateFreeDrawingBrush(currentColor.value);
  }
}

/**
 * @description 橡皮擦模式
 */
function eraserToggleHandler() {
  if (canvas) {
    canvas.off("mouse:down", shapeMouseDownHandler);
    // 控制图形绘制大小
    canvas.off("mouse:move", shapeMouseMoveHandler);
    canvas.off("mouse:up", shapeMouseUpHandler);
    canvas.isDrawingMode = true;
    canvas.freeDrawingBrush = new (fabric as any).EraserBrush(canvas);
    canvas.freeDrawingBrush.width = 10;
  }
}

/**
 * @description 可编辑文本绘制事件处理函数
 * @param opt
 */
function createITextHandler(opt: fabric.IEvent<any>) {
  const pointer = canvas?.getPointer(opt.e);
  const x = pointer?.x;
  const y = pointer?.y;

  const text = new fabric.IText("双击编辑文本", {
    left: x,
    top: y,
    fontSize: 20,
    fill: "#000",
    stroke: "#000",
    strokeWidth: 0,
    textAlign: "left",
    hasControls: true,
    selectable: true,
  });
  canvas?.add(text);
  text.enterEditing();
  text.selectAll();
  canvas?.setActiveObject(text);
  canvas?.renderAll();
  mouseToggleHandler();
}

/**
 * @description 文本绘制模式
 */
function textToggleHandler() {
  if (!canvas) return;
  canvas.defaultCursor = "text";
  canvas.off("mouse:down", canvasMouseDownHandler);
  canvas.off("mouse:up", canvasMouseUpHandler);
  canvas.off("mouse:move", canvasMouseMoveHandler);

  // NOTE 先移除掉已有的文本监听
  canvas.off("mouse:down", createITextHandler);
  canvas.on("mouse:down", createITextHandler);
}

const colorGroup = [
  {
    bgColor: "#1d4ed8",
    ringColor: "ring-blue-500",
  },
  {
    bgColor: "#4338ca",
    ringColor: "ring-indigo-500",
  },
  {
    bgColor: "#be123c",
    ringColor: "ring-rose-500",
  },
  {
    bgColor: "#15803d",
    ringColor: "ring-green-500",
  },
  {
    bgColor: "#000",
    ringColor: "ring-slate-500",
  },
];
const currentColor = ref("#1d4ed8");
const pencilSize = ref([1]);

/**
 * @description 选择画笔颜色
 * @param colorItem
 */
function selectColorHandler(colorItem: { bgColor: string; ringColor: string }) {
  currentColor.value = colorItem.bgColor;
  updateFreeDrawingBrush(currentColor.value);
}

/**
 * @description 更新画笔实例
 * @param color
 * @param width
 */
function updateFreeDrawingBrush(color: string) {
  if (!canvas) return;
  const newBrush = new fabric.PencilBrush(canvas);
  newBrush.color = color;
  newBrush.width = pencilSize.value[0] ?? 10;
  canvas.freeDrawingBrush = newBrush;
}

const currentShape = ref<ShapeName>("rect");

/**
 * @description 选择不同的形状
 */
function handleShapeChange(e) {
  const target = e.target.closest("[data-shape]");
  if (target) {
    const shape = target.dataset.shape;
    // 根据不同的shape做不同的形状生成处理
    currentShape.value = shape;
    console.log(currentShape.value);
  }
}

function createShape(shape: ShapeName, options?: any) {
  if (shape == "rect") {
    return new fabric.Rect({
      fill: currentColor.value,
      // stroke: "green",
      // strokeWidth: 3,
      ...options,
    });
  } else if (shape == "circle") {
    return new fabric.Circle({
      fill: "red",
      radius: 0,
      // stroke: "red",
      // strokeWidth: 3,
      ...options,
    });
  } else if (shape == "triangle") {
    return new fabric.Triangle({
      fill: currentColor.value,
      // fill: "green",
      // stroke: "red",
      // strokeWidth: 3,
      ...options,
    });
  }
}

let tempObj: any = null;

/**
 * @description 图形绘制模式
 */
function shapeToggleHandler() {
  // console.log("zhjixongle");

  if (!canvas) return;

  // 选择图形并创建
  canvas.on("mouse:down", shapeMouseDownHandler);
  // 控制图形绘制大小
  canvas.on("mouse:move", shapeMouseMoveHandler);
  canvas.on("mouse:up", shapeMouseUpHandler);
}

function shapeMouseDownHandler(opt) {
  const pointer = canvas!.getPointer(opt.e);
  lastPosX.value = pointer!.x;
  lastPosY.value = pointer!.y;
  isDragging.value = true;

  tempObj = createShape(currentShape.value, {
    left: lastPosX.value,
    top: lastPosY.value,
    originX: "left",
    originY: "top",
  });

  canvas?.add(tempObj!);
}

function shapeMouseMoveHandler(opt) {
  if (!isDragging.value || !tempObj) return;
  const pointer = canvas!.getPointer(opt.e);
  const width = Math.abs(pointer.x - lastPosX.value);
  const height = Math.abs(pointer.y - lastPosY.value);
  const minX = Math.min(lastPosX.value, pointer.x);
  const minY = Math.min(lastPosY.value, pointer.y);

  if (currentShape.value === "circle") {
    // 圆形以直径为基础，left/top 为左上角
    const radius = Math.max(width, height) / 2;
    // console.log("ininon");

    tempObj.set({
      left: minX,
      top: minY,
      rx: radius, // 椭圆 x 半径
      ry: radius, // 椭圆 y 半径
      width: radius * 2,
      height: radius * 2,
      radius,
    });
  } else {
    tempObj.set({
      left: minX,
      top: minY,
      width: width,
      height: height,
    });
  }

  // 坐标更新
  tempObj.setCoords();
  canvas?.renderAll();
}

function shapeMouseUpHandler() {
  isDragging.value = false;
  tempObj = null;
}

const currentSelectedToolbarItemName = ref<ToolBarItemName>("mouse");

/**
 * @description toolbar item点击事件
 */
function handleToolbarChange(e) {
  const target = e.target.closest("[data-baritem]");
  if (target) {
    const clickedItemName = target.dataset.baritem;
    currentSelectedToolbarItemName.value = clickedItemName;
    isShowPencilToolbar.value =
      currentSelectedToolbarItemName.value == "pencil";
    isShowShapeToolbar.value = currentSelectedToolbarItemName.value == "shape";
  }
}

/**
 * 根据button group不同的name进行handler分发处理
 */
watch(currentSelectedToolbarItemName, (newToolbarItemName) => {
  // console.log(newToolbarItemName);

  if (newToolbarItemName == "mouse") {
    mouseToggleHandler();
  } else if (newToolbarItemName == "hand") {
    handToggleHandler();
  } else if (newToolbarItemName == "pencil") {
    pencilToggleHandler();
  } else if (newToolbarItemName == "text") {
    textToggleHandler();
  } else if (newToolbarItemName == "eraser") {
    eraserToggleHandler();
  } else if (newToolbarItemName == "shape") {
    shapeToggleHandler();
  }
});

/**
 * 调整画笔大小
 */
watch(pencilSize, (newPencilSize) => {
  if (newPencilSize.length) {
    updateFreeDrawingBrush(currentColor.value);
  }
});

onMounted(() => {
  initCanvasEditor();
});

onBeforeUnmount(() => {
  if (canvas) {
    canvas.off("mouse:down", canvasMouseDownHandler);
    canvas.off("mouse:move", canvasMouseMoveHandler);
    canvas.off("mouse:up", canvasMouseUpHandler);

    canvas.dispose();
    canvas = null;
  }
});
</script>

<template>
  <main>
    <canvas id="canvasRef"></canvas>
    <div
      v-if="isShowPencilToolbar"
      class="flex flex-col bg-white p-4 absolute bottom-20 left-[47%] gap-5"
    >
      <div class="flex gap-3">
        <div
          class="w-6 h-6 rounded-full"
          :class="`${
            currentColor == item.bgColor ? item.ringColor + ' ring-4' : ''
          }`"
          :style="{ backgroundColor: item.bgColor }"
          v-for="(item, idx) in colorGroup"
          :key="idx"
          @click="selectColorHandler(item)"
        ></div>
      </div>
      <div>
        <Slider v-model:modelValue="pencilSize" :max="5" :step="1" />
      </div>
    </div>
    <div
      v-if="isShowShapeToolbar"
      class="bg-white px-4 py-2 flex gap-3 absolute bottom-20 left-[48%]"
      @click="handleShapeChange"
    >
      <Button
        variant="ghost"
        size="icon"
        data-shape="rect"
        :class="currentShape == 'rect' && 'bg-slate-100'"
      >
        <Square />
      </Button>
      <Button
        variant="ghost"
        size="icon"
        data-shape="triangle"
        :class="currentShape == 'triangle' && 'bg-slate-100'"
      >
        <Triangle />
      </Button>
      <Button
        variant="ghost"
        size="icon"
        data-shape="circle"
        :class="currentShape == 'circle' && 'bg-slate-100'"
      >
        <Circle />
      </Button>
    </div>
    <div class="toolbar fixed bottom-3 left-1/2 -translate-x-1/2">
      <div class="flex items-center gap-3">
        <div class="bg-white border py-[7px] px-2 rounded-lg">
          <Button variant="ghost" size="icon" disabled>
            <Undo2 />
          </Button>
          <Button variant="ghost" size="icon" disabled>
            <Redo2 />
          </Button>
        </div>
        <div
          class="bg-white border p-2 rounded-lg"
          @click="handleToolbarChange"
        >
          <Button
            variant="ghost"
            size="icon"
            data-baritem="mouse"
            :class="currentSelectedToolbarItemName == 'mouse' && 'bg-slate-100'"
          >
            <MousePointer2 />
          </Button>
          <Button
            variant="ghost"
            size="icon"
            data-baritem="hand"
            :class="currentSelectedToolbarItemName == 'hand' && 'bg-slate-100'"
          >
            <Hand />
          </Button>
          <Button
            variant="ghost"
            size="icon"
            data-baritem="pencil"
            :class="
              currentSelectedToolbarItemName == 'pencil' && 'bg-slate-100'
            "
          >
            <Pencil />
          </Button>
          <Button
            variant="ghost"
            size="icon"
            data-baritem="text"
            :class="currentSelectedToolbarItemName == 'text' && 'bg-slate-100'"
          >
            <Type />
          </Button>
          <Button
            variant="ghost"
            size="icon"
            data-baritem="shape"
            :class="currentSelectedToolbarItemName == 'shape' && 'bg-slate-100'"
          >
            <Squircle />
          </Button>
          <Button
            variant="ghost"
            size="icon"
            data-baritem="eraser"
            :class="
              currentSelectedToolbarItemName == 'eraser' && 'bg-slate-100'
            "
          >
            <Eraser />
          </Button>
        </div>
      </div>
    </div>
  </main>
</template>

<style scoped>
.data-\[state\=on\]\:bg-accent[data-state="on"] {
  border: 1px solid #ccc;
}
</style>
