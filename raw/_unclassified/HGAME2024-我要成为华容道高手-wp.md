# 我要成为华容道高手

## 题目简述

题目将华容道状态搜索与 HTTP API 交互结合在一起。服务器为每一轮返回长度为 20 的 $4\times5$ 棋盘字符串；客户端需要找到一组合法移动，以 JSON 数组提交。服务器返回 `next` 时继续解下一局，返回 `win` 时给出 flag。决定性难点是对状态空间做 BFS，因此本站将它放入 `_unclassified`，而不是仅因为有 HTTP 外壳就归入 Web。

## 解题过程

### 恢复棋盘编码与移动协议

方向由 `1` 至 `4` 表示上、右、下、左；棋盘字节则区分空格、占位格、单格块、竖向块、横向块和 $2\times2$ 的曹操块。只有每个块的左上角位置作为可操作块记录，其余部分是 `OTHER`。

```go
package game

const (
	UP = iota + 1
	RIGHT
	DOWN
	LEFT
)

const (
	EMPTY = iota + '0'
	OTHER
	SINGLE
	VERTICAL
	HORIZONTAL
	KING
)

type Step struct {
	Position  int `json:"position"`
	Direction int `json:"direction"`
}

func Up(pos int) int {
	if pos < 4 || pos > 19 {
		return -1
	}
	return pos - 4
}

func Right(pos int) int {
	if pos < 0 || pos > 19 || pos%4 == 3 {
		return -1
	}
	return pos + 1
}

func Down(pos int) int {
	if pos < 0 || pos > 15 {
		return -1
	}
	return pos + 4
}

func Left(pos int) int {
	if pos < 0 || pos > 19 || pos%4 == 0 {
		return -1
	}
	return pos - 1
}

func AllBlocks(layout [20]byte) []int {
	var blocks []int
	for i, value := range layout {
		switch value {
		case SINGLE, VERTICAL, HORIZONTAL, KING:
			blocks = append(blocks, i)
		}
	}
	return blocks
}
```

对单格块，移动就是检查相邻格是否为 `EMPTY`，然后交换两个字节。复合块可递归地拆成两个横向或竖向的单格移动：

```go
func Move(layout [20]byte, step Step, blockType byte) [20]byte {
	if layout == [20]byte{} {
		return layout
	}

	moveSingle := func(board [20]byte, position, direction int) [20]byte {
		var destination int
		switch direction {
		case UP:
			destination = Up(position)
		case RIGHT:
			destination = Right(position)
		case DOWN:
			destination = Down(position)
		case LEFT:
			destination = Left(position)
		default:
			return [20]byte{}
		}
		if destination == -1 || board[destination] != EMPTY {
			return [20]byte{}
		}
		board[position], board[destination] =
			board[destination], board[position]
		return board
	}

	if blockType == SINGLE {
		return moveSingle(layout, step.Position, step.Direction)
	}

	switch blockType {
	case VERTICAL:
		switch step.Direction {
		case UP:
			return Move(
				Move(layout, step, SINGLE),
				Step{Down(step.Position), UP},
				SINGLE,
			)
		case DOWN:
			return Move(
				Move(layout, Step{Down(step.Position), DOWN}, SINGLE),
				step,
				SINGLE,
			)
		case RIGHT, LEFT:
			return Move(
				Move(
					layout,
					Step{Down(step.Position), step.Direction},
					SINGLE,
				),
				step,
				SINGLE,
			)
		}

	case HORIZONTAL:
		switch step.Direction {
		case UP, DOWN:
			return Move(
				Move(
					layout,
					Step{Right(step.Position), step.Direction},
					SINGLE,
				),
				step,
				SINGLE,
			)
		case RIGHT:
			return Move(
				Move(layout, Step{Right(step.Position), RIGHT}, SINGLE),
				step,
				SINGLE,
			)
		case LEFT:
			return Move(
				Move(layout, step, SINGLE),
				Step{Right(step.Position), LEFT},
				SINGLE,
			)
		}

	case KING:
		switch step.Direction {
		case UP:
			return Move(
				Move(layout, step, HORIZONTAL),
				Step{Down(step.Position), UP},
				HORIZONTAL,
			)
		case DOWN:
			return Move(
				Move(
					layout,
					Step{Down(step.Position), DOWN},
					HORIZONTAL,
				),
				step,
				HORIZONTAL,
			)
		case RIGHT:
			return Move(
				Move(
					layout,
					Step{Right(step.Position), RIGHT},
					VERTICAL,
				),
				step,
				VERTICAL,
			)
		case LEFT:
			return Move(
				Move(layout, step, VERTICAL),
				Step{Right(step.Position), LEFT},
				VERTICAL,
			)
		}
	}

	return [20]byte{}
}
```

`[20]byte{}` 在这里作为非法移动的哨兵值。递归移动时，前一个子步失败会把零数组传到后一个子步，使整个复合操作失败。

### 对状态空间做 BFS

每个棋盘状态都可直接作为 Go `map` 的键。BFS 从初始布局出发，对每个可操作块枚举四个方向；用 `record` 保留前驱布局和到达当前布局的动作，命中终局后便可反向回溯。

```go
package main

import (
	"slices"

	"solver/game"
)

type record struct {
	lastLayout [20]byte
	action     game.Step
}

func solve(layoutInput string) []game.Step {
	store := make(map[[20]byte]record)
	var layout [20]byte
	copy(layout[:], layoutInput)

	queue := make([][20]byte, 0, 100)
	queue = append(queue, layout)
	store[layout] = record{}

	var winLayout [20]byte
	found := false

	for len(queue) > 0 && !found {
		current := queue[0]
		queue = queue[1:]

		for _, position := range game.AllBlocks(current) {
			for _, direction := range []int{
				game.UP, game.RIGHT, game.DOWN, game.LEFT,
			} {
				action := game.Step{
					Position:  position,
					Direction: direction,
				}
				next := game.Move(current, action, current[position])
				if next == [20]byte{} {
					continue
				}
				if _, seen := store[next]; seen {
					continue
				}

				store[next] = record{current, action}
				if next[13] == game.KING {
					winLayout = next
					found = true
					break
				}
				queue = append(queue, next)
			}
			if found {
				break
			}
		}
	}

	if !found {
		return nil
	}

	var steps []game.Step
	for {
		r, ok := store[winLayout]
		if !ok || r.action.Direction == 0 {
			break
		}
		steps = append(steps, r.action)
		winLayout = r.lastLayout
	}
	slices.Reverse(steps)
	return steps
}
```

棋盘按行优先展开，索引 13 是底部中间出口处的曹操块左上角，因此 `next[13] == KING` 是终局判定。BFS 第一次命中终局时的路径同时也是最短解。

### 调用题目 API

官方解答使用了如下协议：

1. `GET /api/newgame` 返回 `gameId` 和 `layout`。
2. 对 `layout` 调用 `solve`，得到由 `{"position":...,"direction":...}` 组成的 JSON 数组。
3. 将数组 POST 到 `/api/submit/{gameId}`。
4. 响应为 `next` 时，用 `game_stage.layout` 继续下一轮；为 `win` 时读取 `flag`。

交互骨架如下，复现时将已失效的赛时主机替换为当前题目地址：

```go
gameID, layout := newGame("http://target.example/api/newgame")

for {
	steps := solve(layout)
	if steps == nil {
		panic("no solution")
	}

	response := submit(
		"http://target.example/api/submit/"+strconv.Itoa(gameID),
		steps,
	)
	switch response.Status {
	case "next":
		layout = response.GameStage.Layout
	case "win":
		fmt.Println(response.Flag)
		return
	default:
		panic(response.Status)
	}
}
```

官方 PDF 未展示最终 flag 字符串，且赛时 API 已不再可用；因此本文只保留可完整理解与复现的状态转移、BFS 和协议流程，不虚构一个无法验证的 flag。

## 方法总结

- 混合题应按决定性障碍分类。HTTP 只是传输棋盘与答案的外壳，真正的求解核心是无权状态图的最短路搜索。
- 用固定长度数组作状态键，可以便捷地去重；同时保留前驱和动作，可在不为每个队列元素复制整条路径的情况下回溯。
- 复合棋子移动可拆成多个单格交换，但必须让整个操作具有“全部成功或整体失败”的语义，否则会将半移动状态加入搜索空间。
- 自动化解题时要同时实现本地求解和 API 状态机；服务器返回下一局时，不能错用上一局的布局或步骤。
