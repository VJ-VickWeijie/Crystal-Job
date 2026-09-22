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