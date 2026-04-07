# Sudoku 游戏领域对象重构


**一、`Sudoku` / `Game` 职责概述**

- `Sudoku`（领域对象）
	- 职责：持有 9x9 `grid`（number[][]），提供 `getGrid()`、`guess(move)`（在指定格子设置值或清空）、校验/冲突检测、胜利检测、`clone()`（返回独立副本）、`toJSON()`、`toString()`。
	- 不负责：历史管理、UI 更新、持久化策略。

- `Game`（聚合根 / 会话）
	- 职责：持有当前 `Sudoku`（`present`），以及历史快照 `past`（数组）、重做栈 `future`（数组）、`initial`（开局）。实现 `guess(move)`（通过委托给 `Sudoku` 并记录历史）、`undo()`、`redo()`、`canUndo()`、`canRedo()`、`toJSON()`（包含 present/past/future/initial 的序列化数据）、以及注册变更回调（可选）。

**二、`Move` 的定义**

- 结构：`{ row: number, col: number, value: number | null }`
- 类型：值对象（value object）。理由：没有唯一标识符，两个内容相同的 `Move` 被视为相同；只用于描述一次变更。将其作为轻量数据传递而非持久化实体。

**三、历史（history）存储策略**

- 存储内容：`past` 与 `future` 存储的是 `Sudoku` 的快照（即 `grid` 的深拷贝），不是 `Move` 日志。
- 理由：实现简单、语义清晰、撤销/重做直接通过恢复快照完成，避免对每种操作实现逆向逻辑。
- 代价：内存消耗（9x9 的数字数组），对本作业规模可接受；若扩展应考虑差分（delta）。

**四、复制（copy）策略**

- 对 `grid` 使用深拷贝。实现方法优先使用 `JSON.parse(JSON.stringify(grid))` 或手写复制函数，以避免共享引用。
- 必须深拷贝的时机：
	- 在将 `present` 推入 `past` 前；
	- 在返回 `getGrid()` 给 UI 时以防止 UI 侧直接修改内部状态；
	- 在序列化/反序列化时重建对象。
- 若误用浅拷贝会导致：历史快照与当前状态共享引用，后续修改会不可逆地污染历史，撤销失效。

**五、序列化 / 反序列化**

- `Sudoku.toJSON()` 输出：`{ grid: number[][] }`。
- `createSudokuFromJSON(json)`：接收上述结构，返回 `Sudoku` 实例（重建方法/校验器）。
- `Game.toJSON()` 输出：
	{
		present: { grid },
		past: [ { grid }, ... ],
		future: [ { grid }, ... ],
		initial: { grid }
	}
- 说明：只序列化纯数据（数字矩阵）。不序列化函数、回调或运行时状态（例如注册的 UI 回调）。

**六、外表化（调试表示）**

- `Sudoku.toString()`：返回 9 行文本，每行 9 个字符或空格，使用 `.` 表示空格，方便在控制台查看。
- `Sudoku.toJSON()`：便于机器可读的存档与恢复。

**七、Undo / Redo 行为细则**

- `guess(move)`：将 `present` 的深拷贝 push 到 `past`，清空 `future`，然后在 `present` 上应用 `move`。
- `undo()`：若 `past` 非空，则将当前 `present` 深拷贝 push 到 `future`，并把 `past.pop()` 的快照设为新的 `present`。
- `redo()`：若 `future` 非空，则将当前 `present` 深拷贝 push 到 `past`，并把 `future.pop()` 的快照设为新的 `present`。
- 新开局或 `reset()`：清空 `past`/`future`，设置 `initial = present`。

**八、同步到 UI 的策略**

- 领域对象为数据与行为的唯一来源。每次领域状态变更后，`Game` 可触发已注册的变更回调，UI 在回调中用 `getSudoku().getGrid()` 完成渲染数据更新。

**九、导出到工厂/接口**

在 `src/domain/index.js` 需要导出的函数：

- `export function createSudoku(input)` — input 为 `number[][]`，返回实现了 `getGrid()` / `guess()` / `clone()` / `toJSON()` / `toString()` 的对象。
- `export function createSudokuFromJSON(json)` — 从 `{ grid }` 重建 `Sudoku`。
- `export function createGame({ sudoku })` — 接受 `Sudoku` 实例并返回实现 `getSudoku()` / `guess()` / `undo()` / `redo()` / `canUndo()` / `canRedo()` / `toJSON()` 的 `Game` 实例。
- `export function createGameFromJSON(json)` — 从 `Game.toJSON()` 恢复完整 `Game`（含 `past`/`future`/`initial`）。

