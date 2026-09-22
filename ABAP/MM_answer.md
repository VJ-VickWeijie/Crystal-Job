思考了 45s

先给结论：NO_PRICE_FROM_PO = 'X' 不是造成 PB00 消失、P101 出现的直接原因。 它只阻止系统从“上一张采购凭证”继承价格条件；它不会禁止从 Info Record/合同等主数据进行正常定价，也不会禁止系统执行 ZRM201 定价过程。SAP Note 580225 对这一点写得很明确：即使设成 X，信息记录中的条件仍然可能被带入。

你这个案例里，我会把注意力从 NO_PRICE_FROM_PO 转到 POITEM-PO_PRICE、POITEM-CALCTYPE、POCOND-CHANGE_ID、P101 的自动确定来源，以及你们自定义公式 900。

1. NO_PRICE_FROM_PO 到底做了什么

可以把采购订单定价来源粗略理解成：

BAPI_PO_CREATE1
       │
       ├─ 定价过程 ZRM201
       │     ├─ Info Record
       │     ├─ Contract / SA
       │     ├─ Condition Record
       │     └─ 自动条件
       │
       ├─ 上一张 PO 的价格
       │
       └─ 你通过 POCOND 手工传入的条件

当：

no_price_from_po = 'X'

它主要切断的是：

上一张 PO
    ↓
当前 PO

不是：

Info Record
Condition Record
POCOND
定价过程

SAP 的说明是：如果设置 NO_PRICE_FROM_PO = X，不从上一份 document 复制 conditions；但 information record 的 conditions 仍然可以复制/确定。

因此：

NO_PRICE_FROM_PO = 'X'

≠ 不自动定价

≠ 只接受你传入的 PB00

≠ 不出现 P101

这一点非常重要。


---

2. 为什么你没传 P101，它却自己出来？

你给出的 ZRM201：

Step  Counter   Condition
50              P101
51    1         PB00
51    2         PBXX

这里已经暴露出一个很重要的信息：

P101 本身就在 ZRM201 定价过程中，而且排在 PB00/PBXX 前面。

所以你虽然只向：

lt_cond
lt_condx

传了 PB00，但 BAPI 并不是：

“lt_cond 有什么，我就创建什么”

而更像：

① 建立 PO Item
② 确定 Pricing Schema ZRM201
③ 执行自动定价
④ P101/PB00/PBXX 等根据各自规则确定
⑤ 再处理 POCOND 的 Insert / Update / Delete
⑥ 重新计算、检查条件有效性
⑦ 得出最终 KOMV/PRCD_ELEMENTS

SAP Note 580225 也明确说明，在 BAPI_PO_CREATE1 创建 PO 时，POCOND-CHANGE_ID 可以作用于系统自动确定出来的条件。

所以出现：

你传：PB00

最终：
P101

并不代表 BAPI 把 "PB00" 字符串改成了 "P101"。

更可能是：

P101 = SAP Pricing Engine 自动确定
PB00 = 你传入了，但后续没有成功成为最终有效条件

这两个过程是分开的。


---

3. 我现在最怀疑的第一个点：POITEM-PO_PRICE

这个字段你现在没有贴出来，但它对这个问题非常关键。

SAP Note 580225 对它的定义基本是：

PO_PRICE = SPACE
正常自动确定条件。
只有当系统无法找到条件时，NET_PRICE 才可能被采用。

PO_PRICE = '1'
把 POITEM-NET_PRICE 当 Gross Price，
写到定价过程中的 Base Price Condition。

PO_PRICE = '2'
把 POITEM-NET_PRICE 当 Net Price，
写到定价过程中的 Base Price Condition，
并删除其他相关条件。

而且要：

lt_poitemx-po_price = 'X'.

这和你的 P101 有什么关系？

关键就在：

> Base Price Condition 到底是谁？



标准 SAP 通常大家熟悉：

PB00
PBXX

但你们是：

ZRM201

而且里面额外出现：

Step 50 P101
Step 51 PB00
Step 51 PBXX

所以我非常建议你立即检查：

lt_poitem-po_price
lt_poitem-net_price
lt_poitem-calctype

特别是有没有：

lt_poitem-po_price = '1'.

或者：

lt_poitem-po_price = '2'.

如果系统把 P101 配置成了你们这个 Schema 实际使用的主价格条件，那么有可能发生：

POITEM-NET_PRICE
       ↓
PO_PRICE = 1/2
       ↓
Base Price Condition
       ↓
P101

此时你会觉得：

> “我明明 POCOND 传的是 PB00，为什么出来的是 P101？”



实际上 P101 可能根本不是 POCOND 创建的，而是 PO_PRICE/自动 Pricing 创建的。


---

4. 第二个重点：你的 CHANGE_ID

这个字段极其重要。

SAP 对 BAPI 的标准逻辑是：

I = Insert
U = Update
D = Delete

而创建 PO 时：

U

也可以修改“刚刚由 SAP 自动定价确定出来”的 Condition；如果对应 Condition 不存在，则会尝试新增。SAP 官方示例就是：系统自动确定 PB00=90，然后在 POCOND 中用 CHANGE_ID='U' 把 PB00 改成 100。

所以如果你的目标是：

> “不管系统自动找到什么 PB00，我最终就是要 PB00 = 我的值。”



我反而更倾向先测试：

ls_cond-itm_number = '00010'.
ls_cond-cond_type  = 'PB00'.
ls_cond-cond_value = xxx.
ls_cond-currency   = xxx.
ls_cond-cond_p_unt = xxx.
ls_cond-change_id  = 'U'.

而不是简单：

CHANGE_ID = 'I'

原因是如果 PB00 本身能够被自动 determination 找到：

自动 PB00
+
你 Insert PB00

SAP 有可能形成两条 Condition，其中一条 inactive，甚至产生你之前说的：

P101 + PB00

或者：

PB00 + PB00

SAP 甚至有专门的 KBA 讨论 BAPI_PO_CREATE1 传 POCOND 后出现重复条件的问题。

所以这里我要看到你实际：

lt_cond-change_id
lt_condx-change_id

是什么。


---

5. 第三个非常可疑的地方：你这个 900

你给出的 T683S 有一个非常有价值的信息：

Step 51 Counter 1 PB00   KOFRM = 900
Step 51 Counter 2 PBXX   KOFRM = 900

这个我建议你重点查。

KOFRM = 900 是：

> Alternative calculation type for condition value



也就是一个 VOFM 自定义 Condition Value Formula。

所以你们的 PB00 并不是纯标准行为。

实际过程很可能是：

PB00
 ↓
进入 Pricing
 ↓
执行 Formula 900
 ↓
根据某些条件修改：
   XKOMV-KBETR
   XKOMV-KWERT
   XKOMV-KINAK
   ...
 ↓
PB00 有可能 inactive / 0 / 被重新处理

这个地方，我认为比 NO_PRICE_FROM_PO 更值得怀疑。

因为你现在有一个非常明显的特征：

> 有时候只有 P101；

有时候 P101 + PB00。



如果单纯是：

NO_PRICE_FROM_PO = X

它应该表现得比较稳定。

但是如果 Formula 900 里面根据：

Vendor
Material
Plant
Purchasing Org
Info Record
PO Type
某个自定义字段
某个前置 Condition

进行判断，那么：

PO A
P101
PB00 → Formula 900 → inactive/delete

PO B
P101
PB00 → Formula 900 → 保留

这就非常符合你现在说的：

> 不是必然发生。



所以请直接去：

VOFM
→ Formulas
→ Condition Value
→ 900

或者 SE38/SE80 看对应生成程序。

重点搜索：

XKOMV
KOMV
KINAK
KWERT
KBETR
KSCHL
P101
PB00

如果 900 是你们自开发公式，我认为这是目前的高优先级嫌疑点。


---

6. 为什么有时 P101 + PB00，有时只有 P101？

结合你现在提供的信息，我认为完整流程很可能类似这样：

BAPI_PO_CREATE1
                    │
                    ▼
          确定 Schema ZRM201
                    │
           ┌────────┴────────┐
           │                 │
         Step 50          Step 51
          P101          PB00 / PBXX
           │                 │
     自动找到价格？      自动找到价格？
           │                 │
          YES               YES/NO
           │                 │
           ▼                 ▼
         P101             PB00
                             │
                             ▼
                       Formula 900
                             │
                    ┌────────┴───────┐
                    │                │
                  有效             无效
                    │                │
                    ▼                ▼
              P101 + PB00          P101

与此同时：

POCOND PB00
     ↓
CHANGE_ID = ?
     ↓
和自动定价结果 Merge

因此你看到：

情况 A

P101
PB00

意味着：

P101 自动 determination 成功
PB00 也最终被保留

情况 B

P101

意味着：

P101 自动 determination 成功

但 PB00：
可能没真正 Insert/Update 成功
或被重新定价处理
或 Formula 900 让其失效
或存在 condition exclusion
或后续 enhancement 修改了条件

而不是：

NO_PRICE_FROM_PO 把 PB00 转换成 P101

后者我基本可以排除。


---

7. 我建议你现在检查这 6 个字段

先不要大范围 Debug。

在调用 BAPI 前直接看：

lt_poitem-po_price
lt_poitem-net_price
lt_poitem-calctype

以及：

lt_cond-change_id
lt_cond-cond_st_no
lt_cond-cond_count

再看：

lt_condx-change_id
lt_condx-cond_type
lt_condx-cond_value
lt_condx-itm_numberx

尤其是：

CHANGE_ID
PO_PRICE
CALCTYPE

这是第一轮最有价值的三个。


---

8. CALCTYPE 也不能忽略

SAP Help 对 POITEM-CALCTYPE 的解释是：

A = 不重新确定条件

B = 完全重新定价

C = 保留 manual pricing elements，
    其他条件重新确定

而且：

POITEMX-CALCTYPE = 'X'

才生效。

如果你某些项目：

CALCTYPE = 'B'

而另一些不是，

那么完全可以造成：

同一套 POCOND

出现不同结果。

因为 B 会触发新的 price determination。

这个和你的：

> “有些时候 P101 + PB00，有些时候只有 P101”



也高度吻合。


---

9. 我建议你做一个非常简单的四组测试

这个测试能非常快地定位到底是谁影响 PB00。

Test	NO_PRICE_FROM_PO	POCOND PB00	CHANGE_ID	观察

A	X	不传	-	系统自动产生什么
B	SPACE	不传	-	是否因为上一张 PO 产生差异
C	X	PB00	U	PB00 是否稳定存在
D	X	PB00	I	是否出现 P101+PB00/重复


尤其 Test A 非常重要。

如果：

不传 POCOND

就已经：

P101

那么答案已经很明确：

> P101 100% 来自自动 Pricing，不是你的 POCOND，也不是 NO_PRICE_FROM_PO 把 PB00 改成 P101。



然后再做 Test C。

如果：

POCOND PB00
CHANGE_ID = U

还是只有：

P101

下一步就直接 Debug Pricing / Formula 900。


---

10. 还有一个隐藏重点：不要只查 T683S

你现在查 T683S 是正确的第一步，但还不够。

P101 需要继续查它的 Condition Type 定义以及 Access Sequence。

也就是用：

M/06

打开：

P101
PB00
PBXX

比较：

Condition class
Calculation type
Condition category
Access sequence
Manual entries
Header/item condition
Plus/minus
Group condition

最关键的是：

P101 有没有 Access Sequence

如果有：

P101
 ↓
Access Sequence
 ↓
Condition Table
 ↓
Condition Record

那你就继续查：

Vendor
Material
Purchasing Org
Plant
Validity Date

很可能马上就能解释：

> 为什么这一张 PO 自动出现 P101。




---

我的判断优先级

根据你现在给出来的代码和 ZRM201，我目前会这样排查：

优先级 ①：检查 POITEM-PO_PRICE / CALCTYPE

lt_poitem-po_price
lt_poitem-calctype

特别是 PO_PRICE 有没有 1/2。

优先级 ②：检查 POCOND-CHANGE_ID

你的 PB00 最好确认是否：

CHANGE_ID = 'U'

以及：

lt_condx-change_id = 'X'

SAP Note 580225 明确支持创建 PO 时用 U 修改自动确定的 PB00。

优先级 ③：查 P101 的 Access Sequence

看它到底从哪里自动 determination 出来的。

优先级 ④：查 Formula 900

因为：

PB00 → KOFRM 900
PBXX → KOFRM 900

这是你们自定义逻辑，极有可能决定 PB00 最终是否有效。

优先级 ⑤：查 Condition Exclusion / Enhancement

如果前面都没问题，再看：

BAdI ME_PROCESS_PO_CUST
Exit
Enhancement
Condition Exclusion


---

所以针对你最开始的两个问题，可以比较明确地回答：

> 是不是 NO_PRICE_FROM_PO 影响？



它会影响“是否从上一张 PO 继承价格”，但不是 PB00 被 P101 替代的直接机制。而且即使 NO_PRICE_FROM_PO = X，Info Record 的条件仍然可以进入当前 PO。

> 为什么有时候 P101+PB00，有时候只有 P101？



这反而说明 P101 和 PB00 很可能分别来自两条不同的定价路径：P101 自动 determination，PB00 是自动确定或 POCOND merge；PB00 是否最终保留又受到 CHANGE_ID / PO_PRICE / CALCTYPE / Formula 900 / exclusion 等影响。

其中你这个案例里，KOFRM = 900 是我特别想追下去的线索。

如果你下一条把 lt_poitem / lt_poitemx / lt_cond / lt_condx 实际 append 的那段代码贴出来，尤其是 PO_PRICE、CALCTYPE、CHANGE_ID、COND_ST_NO、COND_COUNT，我可以直接按照 BAPI 的实际 Pricing 流程帮你逐字段判断，基本可以把问题缩小到具体哪一个字段或 Formula 900。



思考了 52s

你这组字段一出来，我会把排查优先级明显调整。

现在最可疑的不是 NO_PRICE_FROM_PO，也不是 PO_PRICE/CALCTYPE，而是 CHANGE_ID = 'U' + POCONDX 对 Step/Counter 的标识。

SAP 对 BAPI_PO_CREATE1 的定价说明中，U 的标准场景是：系统先自动确定出某个条件，例如 PB00=90，然后你通过 POCOND-CHANGE_ID = 'U' 把它改成 100。相反，I 的标准语义是额外插入一个条件。

所以你的情况非常可能是下面这个流程：

BAPI 开始创建 PO
        │
        ▼
执行 ZRM201 自动定价
        │
        ├── P101 自动确定成功
        │
        └── PB00？
              │
        ┌─────┴─────┐
        │           │
      找到了       没找到
        │           │
        ▼           ▼
 POCOND U       POCOND U
 找到 PB00      没有 PB00 可更新
        │           │
        ▼           ▼
 修改 PB00       什么也没插进去
        │           │
        ▼           ▼
 P101 + PB00       P101

这与你描述的“不是每次都发生，有时候 P101+PB00，有时候只有 P101”高度吻合。


---

还有一个我现在很关注的问题：你的 POCONDX 看起来可能没填完整

你告诉我的：

lt_cond-cond_st_no = 51.
lt_cond-cond_count = 1.
lt_cond-cond_type   = 'PB00'.
lt_cond-change_id   = 'U'.

但是你列出的 lt_condx 只有：

lt_condx-change_id   = 'X'.
lt_condx-cond_type   = 'X'.
lt_condx-cond_value  = 'X'.
lt_condx-itm_numberx = 'X'.

如果你实际代码里确实没有下面这些：

lt_condx-cond_st_no  = '051'.
lt_condx-cond_st_nox = 'X'.
lt_condx-cond_count  = 'X'.

那我建议你首先补上。

BAPIMEPOCONDX 里确实存在 COND_ST_NO、COND_ST_NOX，而 COND_COUNT 在 X 结构里本身就是 update indicator。

你的 ZRM201：

Step  Counter Condition
50            P101
51    1       PB00
51    2       PBXX

所以你实际上是在告诉 BAPI：

我要操作：
Item xxxx
Step 51
Counter 1
PB00

这个信息最好完整地给进去。

我建议你的结构至少长这样：

CLEAR: ls_cond, ls_condx.

ls_cond-itm_number = '000010'.
ls_cond-cond_st_no = '051'.
ls_cond-cond_count = '01'.
ls_cond-cond_type  = 'PB00'.
ls_cond-cond_value = lv_price.
ls_cond-currency   = lv_waers.
ls_cond-change_id  = 'U'.

APPEND ls_cond TO lt_cond.


ls_condx-itm_number  = '000010'.
ls_condx-cond_st_no  = '051'.

ls_condx-itm_numberx = 'X'.
ls_condx-cond_st_nox = 'X'.
ls_condx-cond_count  = 'X'.
ls_condx-cond_type   = 'X'.
ls_condx-cond_value  = 'X'.
ls_condx-currency    = 'X'.
ls_condx-change_id   = 'X'.

APPEND ls_condx TO lt_condx.

尤其注意这里：

ls_condx-cond_st_no  = '051'.
ls_condx-cond_st_nox = 'X'.
ls_condx-cond_count  = 'X'.

这里 COND_COUNT 在 X 结构里不是填 01，而是：

COND_COUNT = 'X'

因为它表示对应字段有效。BAPIMEPOCOND 中真正的 Condition Counter 才是 NUMC 类型的 01。


---

但是，即使把 X 表补完整，U 仍然有一个根本问题

这是我认为你现在这个问题里最重要的一点。

你设置：

CHANGE_ID = 'U'.

本质上是在说：

> SAP，你自动定价出来以后，找到这个 PB00，然后帮我修改它。



并不是：

> SAP，无论如何都给我创建一个 PB00。



SAP Note 580225 给的 U 示例本身就是这个逻辑：

系统自动确定 PB00 = 90
        ↓
POCOND:
PB00 = 100
CHANGE_ID = U
        ↓
把已有 PB00 改成 100



所以现在要问的关键问题其实变成：

> 在出问题的 PO 中，BAPI 自动 Pricing 阶段到底有没有产生 PB00？



这比研究 NO_PRICE_FROM_PO 更重要。


---

我建议你马上做一个非常有效的测试

拿一笔“最后只有 P101”的数据。

Test 1：完全不传 POCOND/POCONDX

也就是暂时：

* pocond  = lt_cond
* pocondx = lt_condx

让 BAPI 完全自动定价。

看看创建出来到底是什么。

如果结果还是：

P101

没有：

PB00

那基本已经破案了。

说明：

系统自动定价
↓
只产生 P101
↓
没有产生 PB00
↓
你的 CHANGE_ID='U'
↓
找不到 PB00 可以 Update
↓
最终仍然只有 P101

此时 POCOND 根本没有“创建 PB00”。


---

Test 2：把 U 临时改成 I

同一笔测试数据：

ls_cond-change_id = 'I'.

其他保持：

COND_ST_NO = '051'
COND_COUNT = '01'
COND_TYPE  = 'PB00'

如果最后变成：

P101
PB00

那么问题基本可以确定：

不是 NO_PRICE_FROM_PO。

而是：

> 这笔 PO 的自动 Pricing 没产生 PB00，而你的 U 只能修改已有的 PB00；改成 I 以后 PB00 才真正被插入。



CHANGE_ID='I' 正是 SAP 给出的“额外插入条件”的标准用法。


---

但是生产代码不要直接无脑把 U 改成 I

因为另一种情况已经存在：

自动 Pricing 已经找到了 PB00
+
你又 CHANGE_ID = I

那么存在产生：

PB00 自动
PB00 手工

甚至重复 Condition 的风险。

SAP 目前的 KBA 也明确把 BAPI PO 的“Unexpected condition / Expected condition not added / Duplicate condition”等归类为 Purchasing BAPI Pricing 常见问题。

所以：

U

适用于：

自动已经存在 PB00
→ 修改 PB00

而：

I

适用于：

本来不存在 PB00
→ 新增 PB00

这是两种不同语义。


---

现在 P101 的身份也越来越清楚了

你的定价过程：

ZRM201

050       P101
051 / 01  PB00
051 / 02  PBXX

因此我现在不会认为：

PB00 → P101

发生了“转换”。

更准确的是：

Pricing Engine
│
├── Step 50
│     └── 得到了 P101
│
└── Step 51
      ├── PB00
      └── PBXX

P101 和 PB00 是两条独立 Condition。

所以最后只有：

P101

真正应该追的问题是：

> 为什么这一次 Step 51/Counter 1 没有产生 PB00？



而不是：

> 为什么 PB00 变成了 P101？



这是两个完全不同的调查方向。


---

为什么某些 PO 自动有 PB00，某些没有？

现在应该开始比较“成功 PO”和“失败 PO”的自动定价来源。

尤其比较：

Vendor
Material
Purchasing Organization
Purchasing Group
Plant
Company Code

Document Date
Pricing Date

Info Record
Contract
Scheduling Agreement
Source of Supply

特别是 PIR。

因为即使：

no_price_from_po = 'X'.

SAP 仍然会从 Information Record 获取条件；这个参数主要阻止的是从之前的采购凭证复制价格条件。

所以完全可能出现：

PO A

存在有效 PIR/Condition Record：

自动定价
↓
P101
PB00
↓
你的 U 找到 PB00
↓
P101 + PB00

PO B

没有有效 PB00 来源：

自动定价
↓
P101
没有 PB00
↓
你的 U 找不到 PB00
↓
只剩 P101

这就非常符合你现在观察到的现象。


---

Formula 900 我现在会降一级优先级

上一轮我比较怀疑：

PB00 → KOFRM 900

现在看到你的 CHANGE_ID='U' 后，我会把优先级调整成：

① CHANGE_ID = U
② POCONDX Step/Counter 是否完整
③ 自动 Pricing 为什么有时没有 PB00
④ P101 从哪里自动确定
⑤ Formula 900

原因是 Formula 900 属于：

Alternative condition value formula

它主要参与条件值计算。

除非你们 Formula 900 里面写了比较激进的自定义逻辑，否则如果现象是：

> PB00 整行完全不存在



那么 U 找不到目标，比 Formula 900 更符合现象。


---

你现在最好这样 Debug

不用从 BAPI_PO_CREATE1 第一行一路 F5。

先拿：

> 一笔最终只有 P101 的案例。



然后在 BAPI 内 Pricing 完成以后观察内部 Condition Table。

在 S/4 的采购订单处理过程中可以重点寻找：

TKOMV
XKOMV
MMPUR_KOMV

或者使用/观察：

MEPO_DOC_TKOMV_GET

SAP 自己的相关 KBA 也把 MEPO_DOC_TKOMV_GET 列为采购订单定价条件分析相关对象。

你只要找到 POCOND 被应用之前 的那个时间点。

然后搜索：

KSCHL = 'PB00'

如果找不到 PB00

那基本可以确认：

自动 Pricing 没产生 PB00
+
CHANGE_ID U 没有对象可更新

这就是主因。

如果已经存在 PB00

那么继续单步：

PB00
↓
POCOND U
↓
后续 Pricing
↓
Formula 900
↓
最终 PB00 消失

这时我们才去查：

KOFRM 900
Condition exclusion
KINAK
Enhancement / BAdI


---

我现在对你的案例判断

基于你目前给出的所有信息，我认为最有可能的流程已经从：

NO_PRICE_FROM_PO

转到了：

ZRM201
                   │
          自动 Condition Determination
                   │
           ┌───────┴────────┐
           │                │
         P101             PB00
        Step50          Step51/1
           │                │
        经常存在        有时存在
                            │
                  ┌─────────┴─────────┐
                  │                   │
               存在                不存在
                  │                   │
                  ▼                   ▼
            CHANGE_ID U          CHANGE_ID U
                  │                   │
           Update PB00          没有对象可 Update
                  │                   │
                  ▼                   ▼
           P101 + PB00             P101

这个模型目前和你描述的现象匹配度非常高。

另外，你的 POCONDX 我建议立即确认这三个：

ls_condx-cond_st_no  = '051'.
ls_condx-cond_st_nox = 'X'.
ls_condx-cond_count  = 'X'.

如果目前确实没填，先补上再测。

最后做两个 A/B 测试就很容易把问题钉死：

同一笔“失败数据”

A：不传 POCOND
→ 看 SAP 自己有没有 PB00

B：PB00 + CHANGE_ID='I'
→ 看 PB00 是否稳定出现

如果结果分别是：

A → P101

B → P101 + PB00

那我会认为已经基本证明：NO_PRICE_FROM_PO 不是主因；主因是 PB00 在该场景没有被自动确定，而你使用的 U 只是更新，不是在强制创建 PB00。

如果你愿意继续沿这个方向查，下一步最有价值的不是再贴 BAPI 调用代码，而是给我一组 “成功产生 P101+PB00 的 PO”和“一笔只有 P101 的 PO” 的以下字段：LIFNR、MATNR、EKORG、WERKS、INFNR（如果有）、采购凭证类型，敏感值可以打码；以及 M/06 中 P101、PB00 的 Access Sequence。我可以继续帮你把“为什么自动定价一会儿有 PB00、一会儿没有”沿 Access Sequence 一层层定位下去。