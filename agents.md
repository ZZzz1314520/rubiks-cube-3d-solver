# Developer Agents for Rubik's Cube 3D Solver

本文档定义了参与开发和维护该魔方项目的 AI 协同代理（Agents）分工与协作规范，以确保项目在无构建工具（Single-file HTML）的架构下保持清晰和稳定。

---

## 代理角色定义 (Agent Roles)

### 1. 3D 渲染与物理交互代理 (Three.js & Interaction Agent)
*   **职责范围**：
    *   维护 [index.html](file:///d:/share/code/mofang/index.html) 中的 Three.js 场景初始化、相机视角控制 (`updateCamera`, `onWheel`)。
    *   管理魔方 3D 模型渲染，包括立方体网格 (`createCube`)、贴纸贴附 (`addSticker`)。
    *   处理物理旋转和动画逻辑 (`rotateLayer`, `rotateCoord`)。
    *   优化拖拽手势交互，通过 Raycaster 计算旋转方向 (`onPointerDown`, `onPointerMove`, `inferMoveFromDrag`, `fallbackFaceMove`)。
*   **技术栈**：Three.js (WebGL), 3D 几何学, 线性代数, 屏幕到世界坐标映射。

### 2. 求解逻辑与状态校验代理 (Solver Logic & Validation Agent)
*   **职责范围**：
    *   对接 `cubejs` 求解核心，维护魔方的逻辑状态模型 (`cubeModel`)。
    *   管理打乱算法 (`scrambleCube`, `randomScramble`) 与求解算法 (`solveCube`)。
    *   负责状态双向同步与校验 (`assertVisualState`, `validateMoveDefinitions`)，确保视觉表现与求解器数组绝对一致。
    *   管理撤销栈 (`undoLastMove`)。
*   **技术栈**：Kociemba 两阶段算法, 魔方状态表 (Facelet Strings), 矩阵变换。

### 3. UI/UX 交互与控制流代理 (UI/UX & Controller Agent)
*   **职责范围**：
    *   维护控制面板（CSS/HTML 部分）以及响应式布局（适配移动端和桌面端）。
    *   管理播放/暂停、单步前进/后退、点击特定步骤跳转的状态控制流 (`runSolve`, `playSolveFrom`, `stepSolveNext`, `stepSolvePrev`, `navigateToSolveStep`)。
    *   管理操作忙碌锁 (`isBusy`, `isSolving`, `solveNavigationBusy`)。
    *   绑定键盘快捷键 (`onKeyDown`) 与各种按钮点击事件。
*   **技术栈**：HTML5, CSS Grid/Flexbox, 浏览器 DOM 事件流, 异步控制流 (Promises/Async-Await)。

---

## 协同工作流程 (Workflow Guidelines)

由于项目是**单文件架构**（All-in-one `index.html`），各 Agent 在修改文件时需严格遵守以下规则以避免冲突：

1.  **修改隔离性**：
    *   涉及 DOM 增删时，需同步修改脚本顶部的 `ui` 对象引用。
    *   修改 3D 交互时，不要改动 `cubeModel` 相关的逻辑求解属性，反之亦然。
2.  **状态防抖与锁机制**：
    *   任何涉及动画的动作必须检查 `isBusy`。
    *   任何涉及自动解算/播放的动作必须受 `isSolving` 和 `solvePaused` 约束。
3.  **强制校验**：
    *   修改任何转动轴向（如 `FACE_DATA`）后，必须确保在 boot 时通过 `validateMoveDefinitions` 校验，并在每一步操作后调用 `assertVisualState` 检查是否同步。
