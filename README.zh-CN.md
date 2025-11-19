# json-repair-rust（中文说明）

Rust 实现的极简 JSON “修复”库，通过 [PyO3](https://github.com/PyO3/pyo3) 和 [maturin](https://github.com/PyO3/maturin) 暴露给 Python 使用。

与很多“智能修复”库不同，它只做极少数、**明确定义的结构性修复**，拒绝一切“猜测用户想要什么”的魔法行为。

## 这个库会做什么（能力）

- **只修复简单、结构性错误**（典型 LLM / 日志输出场景）：
  - 自动闭合被截断的对象/数组  
    例如：`{"a": 1` 在 EOF 停止时，被视为 `{"a": 1}`。
  - 允许对象/数组中的尾逗号  
    例如：`{"a": 1,}`、`[1, 2,]`。

- **支持并忽略注释**：
  - 行注释：`// ...`、`# ...`
  - 块注释：`/* ... */`

- **数字行为与 Python 保持一致**：
  - 普通整数 → Python `int`
  - 浮点数 → Python `float`
  - 超过 `i64` 范围的大整数，会退回给 Python 的 `int()` 处理，保持**任意精度**，不丢数据。

- **保留未知转义和损坏的 `\u` 序列**：
  - 未知转义（例如 `"\w"`）会被保留为文本 `\\w`，不会静默丢掉反斜杠。
  - 损坏的 Unicode 转义（例如 `"\u123z"`），会以文本 `"\\u123z"` 形式保留，不会只剩一个 `\u`。

## 这个库不会做什么（刻意不做）

- **不会“猜”用户的意图**：
  - 不会凭空补全缺失的 key / 冒号 / 引号。
  - 不会把任意一坨文本“尽量解释成 JSON”。
  - 真正的结构性错误（如缺少 `:`）会直接抛出 `ValueError`，而不是“帮你猜”。

- **不会悄悄修改数据**：
  - 不会把 `None`、`NaN`、`Infinity` 等非 JSON 标准值自动转换成合法 JSON。
  - 不会吞掉未知转义或损坏的 `\u` 后面的字符。

- **不会尝试解析多个顶层文档**：
  - 只解析**一个**顶层 JSON 值。
  - 输入多余内容会被忽略，而不是尝试拼接多个文档。

## 为什么要这样设计（设计哲学）

- **确定性优先于“聪明”启发式**  
  JSON 修复工具的职责是修复少量、明确、结构性的错误，而不是替上游的 LLM / 日志系统背锅，更不能把垃圾输入“凑合修好”。  
  一旦 Parser 开始猜测，行为就变得不可预测，bug 也会被掩盖。

- **数据安全优先于“方便”**  
  静默丢字符、乱改转义、自动转换奇怪的字面量，看似“好用”，本质上是**数据损坏**。  
  这个库的策略是：
  - 对明确安全的问题（截断、尾逗号、注释）进行修复；
  - 对其它模糊情况直接报错，让调用方自己决定如何处理。

- **简单 + 高性能**  
  解析器是一个小而直接的递归下降实现：
  - 不搞复杂状态机；
  - 不在热路径上做不必要的堆分配；
  - 暴露给 Python 的就是一个函数：`repair_json(str) -> Any`。

## Python 使用示例

```python
from json_repair_rust import repair_json

data = repair_json('{"a": 1, "b": [1,2,],}')
assert data == {"a": 1, "b": [1, 2]}
```

大整数示例：

```python
from json_repair_rust import repair_json

data = repair_json('{"id": 123456789012345678901234567890}')
assert isinstance(data["id"], int)
```

损坏转义示例（不会丢数据）：

```python
from json_repair_rust import repair_json

data = repair_json(r'{"path": "C:\\Windows", "weird": "\\u123z"}')
assert data["path"] == r"C:\Windows"
assert data["weird"] == r"\u123z"
```

## 本地构建（开发者）

```bash
pip install maturin
maturin develop

python -c "from json_repair_rust import repair_json; print(repair_json('{\"x\": 1,}'))"
```

## 适用场景

适合这些场景：

- LLM 生成的 JSON 不时多一个尾逗号或被截断；
- 日志/配置中混杂注释和轻微格式错误；
- 希望在 **不改变语义、不掩盖错误** 的前提下，对 JSON 做有限修复。

不适合这些场景：

- 你希望“无论输入多烂都能给你一个 JSON”；  
- 想用它来“纠正”上游系统的设计问题，而不是修复少量流式/截断错误。

