---
name: writing-contextual-code-comments
description: Use when 任务将新增或修改源代码，无论用户是否明确要求添加或更新注释
---

# Writing Contextual Code Comments

## Overview

基于仓库证据写注释：声明使用目标语言的原生文档格式，关键代码块使用普通注释；说明职责、系统位置与非显然意图，不复述名称或语法。

> 调查上下文 -> 确认语言与项目惯例 -> 为符号说明职责和系统位置 -> 为关键代码块说明原因或约束 -> 验证证据边界与行为未变

本技能只约束注释，不扩大用户授权的代码变更范围。若任务仅要求注释，则不改变签名、可执行语句、行为或无关格式。始终保持注释简洁专业。

## Workflow

1. 阅读仓库说明、声明、接口或基类、调用方、被调用方、测试及相邻注释。列出用户范围内的类型、构造函数、函数或方法、属性和字段，以及它们的参数、非 `void` 返回值和泛型类型参数。
2. 识别语言并检查同模块的文档格式和注释自然语言。项目惯例优先；没有明确惯例时采用该语言生态主流格式及用户指定或仓库主要自然语言。
3. 逐项使用原生文档注释，写明职责或对外契约及其在子系统或工作流中的位置。每个参数、非 `void` 返回值和泛型类型参数都必须使用原生标签或章节记录，名称清晰也不能省略。
4. 检查重要分支、`switch`、循环、状态转换、异常边界、同步点和多步骤业务规则；在代码块前说明原因、维护的规则或选择影响。简单守卫与机械操作无需旁白。
5. 保留准确注释，直接替换过时、重复或误导内容。无法确认时继续调查；关键不确定性仍存在则询问用户，只记录可确认部分。最后核对清单与差异。

## Language Conventions

| Language | 原生格式与回退要点 |
|---|---|
| C# | XML documentation `///`；每个参数用 `<param>`、非 `void` 返回值用 `<returns>`、泛型类型参数用 `<typeparam>`；primary constructor 参数写在类型文档中 |
| Java | Javadoc `/** ... */`；每个参数和泛型类型参数用 `@param`（泛型写作 `@param <T>`），非 `void` 返回值用 `@return` |
| Kotlin | KDoc `/** ... */`；参数和泛型类型参数用 `@param`，非 `Unit` 返回值用 `@return`，primary constructor property 按项目惯例用 `@property` |
| Python | 项目既有 docstring 风格；逐项填写 Args/Parameters、Returns、Type Parameters 等对应章节；实例属性或 property 写在工具认可的 Attributes 等位置 |
| JavaScript / TypeScript | JSDoc / TSDoc `/** ... */`；每个参数用 `@param`、返回值用 `@returns`、泛型用 `@typeParam` 或项目采用的 `@template` |
| Go | declaration 前的 `// Name ...`；没有专用标签时在原生 prose 中逐项说明参数、返回值和泛型类型参数 |
| Rust | rustdoc `///`，模块级 `//!`；按项目风格使用 Arguments、Returns 等段落，并在类型或函数文档中逐项说明泛型参数 |
| C / C++ | 项目采用的 Doxygen `/** ... */` 或 `///` |
| Swift | DocC `///` |
| Ruby | YARD `#` 及必要标签 |
| PHP | PHPDoc `/** ... */` |
| 未列出语言 | 先查项目范例；无范例时采用该生态主流格式，不套用其他语言语法 |

语言或工具不支持独立文档节点时，将说明写在其认可的包含声明位置，不伪造语法。

## Comment Contract

范围内每个符号都有两个必填信息槽：

1. **职责或契约：** 为调用者或所属模块完成什么。
2. **系统位置：** 位于哪个子系统、工作流阶段或边界，为何存在于此。

约束、副作用、失败语义和生命周期按证据选填。字段、属性、构造函数和方法都在覆盖清单内。关键位置注释只填“原因、规则或影响”，放在对应代码块之前；方法摘要不能替代分支内的意图注释。

## Parameter, Return And Generic Contract

名称或类型清晰不能免除文档。每一项都使用目标语言认可的标签或章节，并在该条目自身同时写出两个固定信息槽：**当前操作如何使用它**，以及**它连接的来源、消费方、流程阶段或系统边界**。不能依赖函数摘要替单个条目补足系统关系。

- **参数：** 逐个说明它在当前操作中的职责，以及它从哪个系统边界进入、影响哪个流程阶段或与哪个协作者交互。有证据时补充单位、范围、可空性、所有权或生命周期。
- **返回值：** 每个非 `void` 返回值都说明由谁或哪个流程阶段消费、代表什么系统状态；有证据时补充空值、失败、哨兵值或所有权语义。不得只写“返回结果”或复述返回类型。
- **泛型类型参数：** 逐个说明它在抽象、扩展点、适配器或系统边界中的职责；有证据时补充类型约束、能力要求、协变/逆变、生命周期或所有权。不得写“`T` 表示类型 `T`”。

`void`、`Unit` 或语言等价的无返回值函数不创建虚假返回标签；将可观察副作用及其系统影响写入函数职责。无法确认特殊语义时只写可验证作用，继续调查并在必要时询问。

## Example

```csharp
/// <summary>
/// 在支付获批后协调库存预留，是结账工作流与库存端口之间的边界。
/// </summary>
/// <typeparam name="TInventory">
/// 结账预留阶段使用的库存边界适配器类型，负责提供库存检查与提交能力。
/// </typeparam>
internal sealed class ReservationCoordinator<TInventory>
    where TInventory : IInventoryGateway
{
    /// <summary>提供库存检查与预留提交能力。</summary>
    private readonly TInventory _inventory;

    /// <summary>
    /// 创建结账库存预留协调器，并将其连接到库存端口。
    /// </summary>
    /// <param name="inventory">为预留流程提供库存检查与提交能力的库存端口。</param>
    public ReservationCoordinator(TInventory inventory)
    {
        _inventory = inventory;
    }

    /// <summary>
    /// 标识库存请求应路由到的仓库边界；调用预留前必须完成初始化。
    /// </summary>
    public required string WarehouseId { get; init; }

    /// <summary>为已获批订单预留库存；库存不足时不提交部分预留。</summary>
    /// <param name="order">由结账流程交付、需要在目标仓库完成库存预留的订单。</param>
    /// <returns>供结账流程消费的异步预留结果，表示已接受或库存不足。</returns>
    public async Task<ReservationResult> ReserveAsync(Order order)
    {
        var stock = await _inventory.CheckAsync(WarehouseId, order.Lines);

        // 保持预留全有或全无，避免订单与库存状态分叉。
        if (!stock.Covers(order.Lines))
            return ReservationResult.InsufficientStock();

        return await _inventory.ReserveAsync(WarehouseId, order);
    }
}
```

## Quick Reference

| 对象 | 交付形状 |
|---|---|
| 每个范围内符号 | 原生文档格式 + 职责或契约 + 系统位置 |
| 参数、返回值、泛型 | 每项必写 + 当前职责 + 系统流程或边界中的作用 |
| 每个关键代码块 | 就近说明原因、规则或影响 |
| 不确定信息 | 继续调查；仍不确定则询问，不猜测 |
| 最终差异 | 注释不引入额外行为变化或无关格式化；保留用户授权的代码改动 |

## Common Mistakes

- 用 C# 普通 `//` 或 Python 声明前 `#` 代替原生文档格式。
- 只注释类型，漏掉范围内的方法、属性或字段。
- 因名称清晰而省略参数、非 `void` 返回值或泛型类型参数，或只复述名称和类型。
- 只在方法摘要写规则，关键分支没有就近意图注释。
- 写“获取值”“执行处理”等标识符转述，或逐行解释机械操作。
- 根据名称虚构持久化、失败语义或业务含义；应回到接口、调用和测试取证。
- 在旧注释旁叠加新说法；应直接替换过时或重复内容。

## Verification Gate

交付前确认：

- 清单中每个符号都有正确原生格式，属性、字段、构造函数和方法没有遗漏。
- 每个参数、非 `void` 返回值和泛型类型参数都已逐项记录，并说明当前职责及其在系统流程、子系统或边界中的作用。
- 每个符号说明职责或契约及系统位置；关键代码块就近说明原因、规则或影响。
- 格式和自然语言符合项目惯例；所有结论可由代码、调用、测试或仓库术语支持。
- 不确定信息已经继续调查或询问用户，没有补写看似合理但无证据的含义。
- 注释没有引入用户授权范围外的行为变化或无关格式化。

任一项不满足，继续调查或修订后再交付。
