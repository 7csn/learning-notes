# 注意
    大部分情况下，行前空格会被忽略

# 标题
    1. 标题行前置 # 号，# 号个数表示标题级别，分 1 - 6 级
    2. 另外，可使用全 = 号行和上一行组成 1 级标题，使用全 - 号行和上一行组成 2 级标题

> 案例：
>> # 一级标题
>> ## 二级标题
>> ### 三级标题
>> #### 四级标题
>> ##### 五级标题
>> ###### 六级标题

# 分隔线
    由 * 号、- 号、_ 号 组成，三个或以上，可混用，中间可以加入空格

> 案例：
>> ---
>> ***
>> ___

# 分割虚线
    [========]

> 案例：
>> [========]

# 段落
    行尾使用两个空格表示换行

> 案例：
>> 文本后接两个空格  
>> 文本后不接两个空格
>> 文本

### 字体
    文本两侧各加单个 * 号或 _ 号表示斜体，两个则表示粗体，三个表示粗斜体  

> 案例：
>> *斜体*
>> _斜体_
>> **粗体**
>> __粗体__
>> ***粗斜体***
>> ___粗斜体___

### 复选框
    1. 未选中：
        1. - [ ] 选项
        2. * [ ] 选项
        3. + [ ] 选项
    2. 已选中：
        1. - [x] 选项
        2. * [x] 选项
        3. + [x] 选项

> 案例：
>> * [ ] 未选中
>> * [x] 已选中
>> - [ ] 选项列表
>>     + [x] 选项1
>>     + [ ] 选项2

### 删除线
    文本两侧各加上两个 ~ 号

> 案例：
>> ~~删除线~~

### 脚注
    [^脚注标记]
    [^脚注标记]: 脚注说明

> 案例：
>> 我要学习[^PHP]
>> [^PHP]: 世界上最好的 web 语言。

# 列表
    1. 无序列表，前置 * 号、+ 号或 - 号和空格
    2. 有序列表，数字（值不重要）后接 . 号和空格
    3. 列表嵌套，子列表前补四个空格

> 案例：
>> * 无序列表
>> * 无序列表
>> + 无序列表
>> + 无序列表
>> - 无序列表
>> - 无序列表
>> 1. 有序列表
>>     - 无序列表
>>         1. 有序列表
>>         2. 有序列表
>>     - 无序列表
>> 2. 有序列表
>>     * 无序列表
>>     * 无序列表

# 区块
    段落前置 > 号，多个表多层嵌套

> 案例：
>> 区块1
>>> 区块2
>>> * 无序列表
>>>     > 区块
>>> * 无序列表
>>>> 区块3

# 代码
    1. 代码片段两侧加 ` 号
    2. 代码区块
        1. 各行前置4个空格
        2. 用 ``` 包裹代码块，需换行，可以指定语言类型

> 案例：
>> 代码片段：
>>
>> `echo 'Hello world!';`  
>>  
>> 代码区块：  
>>>     function welcome() {
>>>         return 'Hello world!';
>>>     }
>>> ```php
>>> echo 'Hello world!';`
>>> ```
    
# 链接
    1. 常规链接
        1. [链接名称](链接地址)
        2. 链接地址
        3. <链接地址>
    2. 高级链接  
        [显示名][链接变量]  
        [链接标记]：链接地址

> 案例：
>> [百度](https://www.baidu.com)
>>
>> https://www.baidu.com
>>
>> <https://www.baidu.com>
>>
>> [谷歌][google]
>> [google]: https://www.google.com

# 图片
    1. ![alt 属性文本](图片地址)
    2. ![alt 属性文本](图片地址 "可选标题")
    3. 使用 HTML 的 img 标签

> 案例：
>> ![百度无 title](https://ss0.bdstatic.com/-4o3dSag_xI4khGkpoWK1HF6hhy/wisegame/wh%3D68%2C68/sign=5e987e7a7d8b4710ce7af5cafbe2f5c5/574e9258d109b3de78bb6233c3bf6c81800a4c61.jpg)
>> ![百度有 title](https://ss0.bdstatic.com/-4o3dSag_xI4khGkpoWK1HF6hhy/wisegame/wh%3D68%2C68/sign=5e987e7a7d8b4710ce7af5cafbe2f5c5/574e9258d109b3de78bb6233c3bf6c81800a4c61.jpg "百度图标")
>> <img width="50" src="https://ss0.bdstatic.com/-4o3dSag_xI4khGkpoWK1HF6hhy/wisegame/wh%3D68%2C68/sign=5e987e7a7d8b4710ce7af5cafbe2f5c5/574e9258d109b3de78bb6233c3bf6c81800a4c61.jpg" alt="百度有 title" title="百度图标">

# 表格
    用 | 分隔左右单元格，首行为 head，次行为分隔行，其余为 body  
    次行单元格内容以 - 号为主，左置 : 号表示列左对齐；右置 : 号表示列右对齐；左右置 : 表示列居中。

> 案例：
>> | 左对齐 | 右对齐 | 居中对齐 |
>> | :- | -: | :-: |
>> | 单元格 | 单元格 | 单元格 |
>> | 单元格 | 单元格 | 单元格 |

# 转义
    前置 \ 符以解除特殊符号特性，特殊符号：\ ` * _ {} [] () # + - . !

# HTML 元素
    目前支持的元素有 <kbd> <b> <i> <em> <sup> <sub> <s> <u> <br> <abbr> 等

> 案例：
>> 键盘文本：<kbd>Ctrl</kbd> 
>> <b>粗体</b>  
>> <i>斜体</i>  
>> <em>斜体</em>  
>> 上标：X<sup>3</sup>  
>> 下标：H<sub>2</sub>o  
>> <s>删除线</s>  
>> <u>下划线</u>  
>> 换行：<br>
>> 缩写：<abbr title="Hyper Text Markup Language" >HTML</abbr>

# HTML 特殊符号

> 案例：
>> 单双引号：&apos; &quot;  
>> 空格：&nbsp;  
>> & 符：&amp;
>> 大于小于号：&lt; &gt;  
>> 加减号：&plusmn;  
>> 乘除：&times; &divide;  
>> 间隔号：&middot;  
>> 左右书名号：&laquo; &raquo;  
>> 版权：&copy;  
>> 商标及注册商标：&trade; &reg;  
>> 欧元英镑人民币：&euro; &pound; &yen;  
>> 温度：&ordm;C  
>> 个位分数（难用）：&frac35;  
>> 次方：100&sup3;  
>> 西班牙反叹号：&iexcl;  
>> 欧洲段落符：&para;  
>> 分节符/分段符：&sect; 

# 科学公式 TeX(KaTeX)
    由 $$ 包裹

# 流程图
    ```
    graph TD
    图块-->图块
    ID-->图块
    ...
    ID->END[结束]
    ```
> 案例：
>> ```
>> graph TD
>> A[开始]-->B{是否登录}
>> B-->C[用户管理]
>> C-->D{是否注册}
>> D-->|是|F(注册)
>> D-->|否|E(输入用户名密码)
>> E-->G[登录]
>> F-->H[注册完成]
>> H-->G
>> G-->I[完善个人信息]
>> I-->END[结束]
>> ```

# 甘特图
    ```mermaid
    gantt
    title 图表名称
    dateForma 日期格式
    section 任务1
    需求分析1:日期, 持续天数
    section 任务2
    需求分析2:日期
    ...
    ```

> 案例：
>> ```mermaid
>> gantt
>> title 图表名称
>> dateFormat YYYY-MM-DD
>> section 任务1
>> 需求分析1:2018-01-11, 8d
>> section 任务2
>> 需求分析2:2018-01-19, 15d
>> section 任务3
>> 需求分析3:2018-01-26, 15d
>> ```

# 绘制流程图 Flowchart

```flow
st=>start: 用户登陆
op=>operation: 登陆操作
cond=>condition: 登陆成功 Yes or No?
e=>end: 进入后台

st->op->cond
cond(yes)->e
cond(no)->op
```

# 绘制序列图 Sequence Diagram

```seq
Andrew->China: Says Hello
Note right of China: China thinks\nabout it
China-->Andrew: How are you?
Andrew->>China: I am good thanks!
```
