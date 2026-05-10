帮我写一个Deepseek AI聊天的桌面应用程序：
- 不必有多美观，甚至不必美观。它是生产力工具。
- 我强烈不建议进行并发工具调用。这个工具的复杂度在当前情况下已经很灾难了。
- 端点：https://api.deepseek.com/chat/completions
- 请求体需要硬编码的部分：
```json
{
  "model": "deepseek-v4-pro",
  "thinking": {"type": "enabled"},
  "reasoning_effort": "high",
  "stream": true
}
```
- **不进行**任何隐式的上下文压缩。reasoning_content总是完整传递（包括但不仅限于推理内容、工具调用、工具输出），哪怕它不输于agent的。
- **所有的**历史信息都应该能在本地被编辑。所有的
- 支持SSE（云）和stdio（本地）两种MCP。MCP只支持从配置文件中编辑，不支持热编辑。
- 工具启动后会按照以下逻辑进行工具加载：
  - 首先加载本地的配置文件。配置文件中存储了MCP服务器的地址和`approved_tools`、`manually_approved_tools`两个列表。
    - 这两个列表均存储string，内容为`mcp名称::工具名称`，例如`my_mcp::tool_a`。
    - 不允许存在重名的MCP。
  - 尝试连接至MCP。stdio类型的工具，则尝试拉起子进程。不进行重试。如果工具列表为空也视为连接失败。
  - 对于连接成功的MCP，收集它们的tools列表，并与配置文件中的`approved_tools`、`manually_approved_tools`进行对比，筛选出最终的工具列表。
    - 如果一个工具既在`approved_tools`又在`manually_approved_tools`中，则将其从`approved_tools`中移除。
    - 如果有加载成功的MCP，且这两个列表中存在此MCP没有的工具，例如`my_mcp::tool_b`，则从列表中移除这些工具。
    - 如果有加载成功的MCP，且MCP中存在这两个列表里面没有的工具，例如`my_mcp::tool_b`，则将这些工具添加到**不持久化的**`unapproved_tools`列表中。
    - 哪怕这两个列表中包含加载失败的MCP甚至不存在的MCP的工具，也不要将这些工具从列表中移除。
    - 如果运行过程中产生了连接错误（例如子进程崩溃），则将该MCP标记为连接失败。无需考虑MCP的恢复。
    - 用户打开工具管理页面时，能且只能看到加载成功的MCP提供的工具；每个工具可以从从`approved`、`manually_approved`、`unapproved`三个状态中切换。
  - **每次修改**时保存这两个列表。
- 然后进入聊天列表界面。聊天界面左边是对话列表，右边是当前对话的聊天内容。聊天内容的底部是输入框。
  - 每一个聊天都有且只有一个元数据：标题。标题默认为对话创建的时间，可以手动在左边的对话列表中in-place地编辑。
  - 每个聊天都可以复制（dupe）和删除（delete）。每个聊天都保存到磁盘的一个json文件，文件名为`标题.json`。
- 聊天中的每一个消息均为标准schema，显示为标准格式（内容按markdown渲染，role为system/user/assistant/tool）。
  - 使用microlight.js（tool的content）和marked.js（其他）进行高亮
  - 消息有且只有两个元数据
    - `send_to_server`，默认为true。为false时这条消息不发送给服务器。
    - `approved`，只在`assistant`角色上存在，默认为false。
    - 除此之外，没有其他的元数据。
  - tool definitions总是全局的，在上述的MCP与工具加载逻辑中进行管理。消息列中不保存。
  - 有且只有`assistant`有`reasoning_content`和`tool_calls`。
  - `tool`的`content`总是展示原始信息。
- 对话进行的核心为continue操作，并且有一个默认打开，可以关闭的`auto_continue`开关。具体而言：
  - 如果最后一条消息是user，那么按照经典地ai客户端的行为进行生成。
    - 在生成完毕后，如果`auto_continue`打开，立即自动进行一次continue操作。
  - 如果最后一条消息是assistant，并且它没有进行`tool_calls`，则**不进行任何操作**。
  - 如果最后一条消息是assistant，且它有不少于一个`tool_calls`，按照以下规则进行判定：
    - 如果有不少于一个`tool_calls`是无效（包含schema错误和工具不存在两种情况）的，则红色高亮无效的`tool_calls`五秒钟，并且**不进行任何操作**。
    - 如果全部的`tool_calls`都是有效的：
      - 如果此消息的`tool_calls`均为`approved_tools`或本消息的`approved`为true，则立即调用其**第一个**工具，并在调用完毕后，如果`auto_continue`打开，**无论调用是否成功**，立即自动进行一次continue操作。
      - 否则，**不进行任何操作**。
      - 换言之，用户的审批依赖`approved`字段。与传统的MCP客户端不同，用户大可以**修改tool_calls的内容后审批**。
    - 如果最后的消息为`tool`，则向上回溯最后一条`assistant`消息，并记录回溯过程中的所有消息。
      - 如果不能找到这样的`assistant`消息，或回溯过程中遇到了`tool`以外的消息，则**不进行任何操作**。
      - 如果回溯过程中至少存在这样的一对消息，它们的`tool_call_id`相同，则红色高亮无效的`tool`五秒钟，并且**不进行任何操作**。
      - 如果回溯过程中的所有消息中有存在至少一条这样的消息，这条消息的`tool_call_id`不在回溯到的`assistant`消息的`tool_calls`中（包含回溯到的`assistant`消息没有进行工具调用的情况），则红色高亮无效的`tool`五秒钟，并且**不进行任何操作**。
      - 对于回溯到的`assistant`消息：
        - 如果有不少于一个`tool_calls`是无效的，则红色高亮无效的`tool_calls`五秒钟，并且**不进行任何操作**。
        - 如果全部的`tool_calls`都是有效的：
          - 如果此消息的`tool_calls`均为`approved_tools`或本消息的`approved`为true，则立即调用其**第一个没有被调用过的**工具，并在调用完毕后，如果`auto_continue`打开，**无论调用是否成功**，立即自动进行一次continue操作。
          - 否则，如果其全部申请的工具均已调用完成，那么按照经典地ai客户端的行为进行生成。如果`auto_continue`打开，在生成完成后，立即自动进行一次continue操作。
          - 否则（有未审批的工具，因为bug或者审批解除导致进入了当前状态），则红色高亮无效的`assistant`消息五秒钟，并且**不进行任何操作**。
- 对话的右上角有一个开关，切换`只读-可写-sudo`模式和一个`校验`按钮。
- 所有的消息都可以在sudo模式下被删除，不进行任何校验。
- 所有的消息都可以在可写模式下被删除。具体来说：
  - 如果删除的消息是`user`或者`system`消息，这是最简单的情况 :) 删了就删了，没什么大不了的。
  - 如果删除的消息是`tool`消息，则向上回溯最后一条`assistant`消息，并记录回溯过程中的所有消息。
    - 如果不能找到这样的`assistant`消息，或回溯过程中遇到了`tool`以外的消息，则直接删除当前消息。
    - 对于回溯到的`assistant`消息，如果其`tool_calls`中存在被删除的消息的`tool_call_id`，则将其从`tool_calls`中移除。
      - 如果此次删除使得它丢失了全部的`tool_calls`，从回溯到的`assistant`消息开始向下搜索直到末尾或者搜索到第一条不是`tool`的消息：
        - 如果找到了这样的消息，并且它是`assistant`消息，则将当前的`assistant`消息的`reasoning_content`（如果有）添加到这条消息的头部，添加一个换行符（如果没有那就不添加），并删除搜索路径中全部的消息（全都是tool消息）。
        - 如果没有找到这样的消息（到达了末尾或者搜索到第一条不是`tool`的消息），则直接删除回溯到的`assistant`消息以及搜索路径中全部的消息（全都是tool消息）。
  - 如果删除的消息是`assistant`消息，则从当前消息开始向下搜索直到末尾或者搜索到第一条不是`tool`的消息：
    - 如果找到了这样的消息，并且它是`assistant`消息，则将被删除的消息的`reasoning_content`（如果有）添加到这条消息的头部，添加一个换行符（如果没有那就不添加），并删除搜索路径中全部的消息（全都是tool消息）以及当前的`assistant`消息。
    - 如果没有找到这样的消息（到达了末尾或者搜索到第一条不是`tool`的消息），则直接删除当前的`assistant`消息以及搜索路径中全部的消息（全都是tool消息；assistant消息本身不删除）。
- 所有的消息都可以在sudo模式下被编辑，不进行任何校验。
- 所有的消息都可以在可写模式下被编辑。具体来说：
  - 如果被编辑的消息不是`assistant`消息，这是最简单的情况 :) 编辑了就编辑了，没什么大不了的。
  - 如果被编辑的消息是`assistant`消息，如果编辑的内容涉及到`tool_calls`，则按照以下规则进行校验：
    - 如果编辑后的`tool_calls`中存在无效的工具调用（包含schema错误、工具不存在和存在`tool_call_id`冲突三种情况），则红色高亮无效的工具调用五秒钟，但不丢失当前的半成品编辑状态。
    - 如果这次编辑带来了新的`tool_calls`，则立即在当前消息下依次插入每一个带来的新的`tool_calls`，结果显示为“error”。
- 可以在sudo模式下可以在任意消息前/后插入任何消息，不进行任何校验。
- 可以在可写模式下可以在任意消息前/后插入消息。具体来说：
  - 首先，不允许在`tool`消息前插入任何不是`tool`的消息。不允许在`tool`消息后插入任何不是`tool`的消息，除非这个`tool`消息的下一个消息不是`tool`消息或没有下一个消息。
  - 如果插入的消息是`tool`消息，则向上回溯最后一条`assistant`消息，并记录回溯过程中的所有消息；同时向下追溯最后一条不是`tool`的消息，并记录追溯过程中的所有消息。
    - 如果不能找到这样的`assistant`消息，或回溯过程中遇到了`tool`以外的消息，则红色高亮希望插入的消息五秒钟，但不丢失当前的半成品编辑状态。
    - 对于回溯到的`assistant`消息，如果其`tool_calls`中存在被插入的消息的`tool_call_id`，
      - 如果在追溯到的消息列表中存在此`tool_call_id`，则红色高亮此`tool_call_id`五秒钟，但不丢失当前的半成品编辑状态。
      - 否则，插入成功。
    - 向这条`assistant`消息的`tool_calls`中插入被插入的消息的`tool_call_id`，并要求选择schema，但按全空填写其参数。
- 可以手动校验整个对话的完整性。校验内容包括但不限于：
  - 每一个`tool`消息都能找到一个`assistant`消息，并且这个`assistant`消息的`tool_calls`中包含这个`tool`消息的`tool_call_id`。
  - 每一个`assistant`消息中每一个工具调用都有一个对应的`tool`消息，并且这个`tool`消息的`tool_call_id`与之相同。
- 对话总是在进行任何变动后立即保存到磁盘。对于工具调用的结果，也立即保存到磁盘。
- 软件启动时，从磁盘加载对话列表。

我知道关于continue和approve有一点难懂，我解释一下设计意图：
假设我在项目开发了一半时要求“阅读日志并删除临时文件”，现在生成了两次call：
`remove-dir /tmp/`
`read-file /tmp/log.txt`
其中remove-dir是manually_approved_tools。此时生成是成功的，但是哪怕勾选了`auto_continue`，也会在continue时进入无操作分支。
此时我作为用户就会理解，是因为这次的tool里面有潜在危险操作。比如说我发现了第一个命令会错误地删除dir文件夹而不是里面的文件。
我就可以把它修正为remove-files /tmp/*，并且勾选approve，然后再点一下continue。
相当于continue是send的加强版：它不止能对ai发送消息，还能激活mcp工具，甚至让ai基于修订后的mcp操作继续工作（继续专注于阅读日志、开发项目，而不是把注意力放在临时文件上）

---

几点修正与说明：
1，Send按钮和Continue按钮应该是同一个按钮；或者严格来说这里不存在传统意义上的Send操作。
Continue按钮的前端逻辑应该是，如果对话框的内容非空，则先将其作为用户消息添加到对话中，然后按照之前设计的Continue逻辑进行处理。
前端上，不要把continue按钮做在右栏上方，放在右栏下方输入区右边本来应该放在Send按钮的地方即可。
2，只读模式的意思是不允许深度编辑，但是允许进行continue操作。也就是说在只读模式下，用户可以进行Continue操作（包括1中提到的Send+Continue），但是不允许编辑消息内容、删除消息、插入消息等一切修改对话内容的操作。
3，校验的逻辑为（你写的太简单了，我重新严谨定义一下）：
- 对于每一条assistant消息开始，向下追溯，直到寻找到消息列表末尾或者寻找到第一条不是tool的消息前。我们称之为“消息组”。
  - 显然地，每个消息组都由一条作为组首的assistant消息和零或若干条tool消息组成。
- 如果没有寻找到消息列表末尾（称为不是最后一个消息组）：
  - tool_call的长度应该与此Assistant消息下的tool_call的长度相同；否则，高亮该`assistant`消息五秒钟。
- 对于组首的所有assistant消息：
  - 所有的这些消息中的tool_calls的tool_call_id均不应重复。否则，高亮这些重复的tool_call_id五秒钟。
- 对于组首的每个assistant消息：
  - 其tool_calls中不应该存在无效的工具调用（包含schema错误、工具不存在这两种情况）。如果存在，则高亮这些无效的工具调用五秒钟。
- 对于剩下的所有tool消息：
  - 不应该能够找到这样的一个tool消息，其id与之相同。如果找到了，则高亮这个tool消息五秒钟。
  - 其id应该在组首的assistant消息的tool_calls中；否则，高亮这个tool消息五秒钟。
  - 在组首的assistant消息中的tool_call中对应的tool的schema应该与其schema一致；否则，高亮这个tool消息五秒钟。
校验不通过时也能够执行Continue操作；换言之，校验是纯粹可选的渲染工具。
如果上述过程未产生红色高亮，则校验按钮应该被绿色高亮五秒钟。

4，我给出几个edge-case，其中记号为
Assistant<tool_name1(id1)1,tool_name2(id2)...>或者Assistant<>如果没有
Tool<tool<tool_name(id)>
User<>
System<>

场景1 复杂未损坏对话场景
1. User<>
2. Assistant<tool_a(id1),tool_b(id2)>
3. Tool<tool_a(id1)> [send_to_server=false]
4. Tool<tool_b(id2)>
5. Assistant<tool_c(id3),tool_d(id4)>
6. Tool<tool_d(id4)>

Continue操作构建的上下文应该是：
1 2 4 5 6，此时服务端很有可能返回error。这是预期行为的一部分，用户如果取消勾选`send_to_server`，那用户自己对完整性负责。
在可写模式下：
2后只能插入Tool消息。尽管2后本身不构成限制，但是2后就是3前，此时受到“不允许在`tool`消息前插入任何不是`tool`的消息”的限制。
3后只能插入Tool消息，因为其下一条消息(4)是Tool消息；
5前允许插入任何消息。因为虽然5前就是4后，但是对于4来说，命中“不允许在`tool`消息后插入任何不是`tool`的消息，除非这个`tool`消息的下一个消息不是`tool`消息”的除非部分。
6后允许插入任何消息。因为6后没有消息了，此时命中“或没有下一个消息”。
如果尝试在1后插入Tool消息，则会红色高亮这个Tool消息五秒钟，但是不丢失当前的半成品编辑状态。
如果尝试在3后插入Tool消息，此时回溯到的Assistant消息是2。
- 如果尝试插入的是<tool_a(id3)>，那么此时2应该变成Assistant<tool_a(id1),tool_b(id2),tool_a(id3)>，插入成功；其中tool_a(id3)的参数无视schema地为全空。
  - 我知道id3在5中被使用了。这不重要。
  - 如果此时校验，消息2和消息5中的tool_a(id3)和tool_a(id3)会因为id重复而被红色高亮五秒钟。
- 如果尝试插入的是<tool_a(id2)>，因为id2在消息2的tool_calls中已经存在了，所以此时2应该被红色高亮五秒钟，但是不丢失当前的半成品编辑状态。
  - 我知道部分模型允许不同tool使用相同id的。这不重要。
如果尝试在6后插入Tool<tool_c(id3)>，此时回溯到的Assistant消息是5，向下追溯到的消息是末尾。5中已经存在了tool_c(id3)了，所以不编辑消息5；追溯过程中没有遇到id为id3的消息了，所以插入成功；此时消息5仍然是Assistant<tool_c(id3),tool_d(id4)>，新插入的消息是Tool<tool_c(id3)>。
校验当前对话，应该校验成功。

场景2 复杂损坏对话场景
1. User<>
2. Tool<tool_x(idx)>
3. Assistant<tool_a(id1),tool_b(id2)>
4. Tool<tool_a(id1)>
5. Tool<tool_c(id3)>
6. Assistant<tool_d(id4)>
7. Tool<tool_d(id4)>
8. Tool<tool_b(id2)>
9. Assistant<>
10. Tool<tool_e(id5)>
11. Assistant<tool_f(id6)>
12. Tool<tool_g(id7)>
13. Tool<tool_f(id6)>
14. Assistant<tool_h(id8)>
15. User<>

Continue操作构建的上下文应该是：
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15，此时服务端很有可能返回error。这是预期行为的一部分，用户如果进行sudo操作，那用户自己对完整性负责。
在可写模式下：
如果删除消息2，此时回溯终止于消息1（非tool消息），删除成功。共删除一条消息，即消息2本身；不发生其他任何行为。
如果删除消息9，其向下记录消息10。此时因为消息10存在，因此消息9被删除，但内容合并至消息11；消息10被删除；共删除两条消息，即消息9和消息10；消息11的reasoning_content被更新为“消息9的reasoning_content + 换行 + 消息10的reasoning_content”。
如果删除消息10，回溯到消息9，消息10删除成功，但**并未发生删除行为**。此时消息9与11**不**合并。共删除一条消息，即消息10本身；不发生其他任何行为。
如果编辑消息3但不编辑其tool_calls，编辑成功。
如果编辑消息3并且编辑了其tool_calls，哪怕只是为tool_b的参数增加一个空格。此时因为**工具不存在**，编辑失败。但不退出编辑状态（半成品保留）。
如果在消息11前插入Tool<tool_f(id6)>，回溯到消息9，插入成功；消息9变为Assistant<tool_f(id6)>；消息10不变。
校验当前对话，应该校验失败。
删除消息13，回溯到消息11，消息13删除成功，此时消息11丢失了全部的Tool；但向下搜索找到的消息是15，不能进行合并。
- 链路上的tool消息全部删除：删除消息12和13；
- 不在链路上的消息14不受到影响
- 不删除消息11，但编辑消息11为Assistant<>，reasoning_content不变。
