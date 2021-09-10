# [llux](https://github.com/llxiaoyuan/llux)
## ollvm de-obfuscator
## 移除ollvm中的控制流平坦化、不透明谓词

### `ll`取自`ollvm`，`ux`取自`lux`，寓意混淆之中的光亮和线索

![image](https://user-images.githubusercontent.com/36320938/131329596-d1879d97-de90-4dee-a267-13c70a66741c.png)
![image](https://user-images.githubusercontent.com/36320938/131329631-6143b224-fe21-42df-bfeb-149d5040f61f.png)
![image](https://user-images.githubusercontent.com/36320938/131329641-7015a02c-f148-423a-a369-ee4799f39558.png)
![image](https://user-images.githubusercontent.com/36320938/131329649-125eeb73-9219-4336-8068-28e93bd0d5cf.png)
![image](https://user-images.githubusercontent.com/36320938/131240568-c1e65560-58f1-4879-99e4-3d593643f0fd.png)

<br />

### Description

> #### * 实现了在`x86_64架构下`,`开启代码优化后`,`控制流平坦化`以及`不透明谓词`的移除
> #### * 实现原理`不同于符号执行`，无需条件约束，在遇到BCF增加的死循环块的时候，依然可以完成路径遍历
> #### * 针对ollvm进行了充分地优化，速度对于符号执行框架angr速度较快，且可以处理复杂样本
> #### * 暂未开源，因为这样会使得混淆对抗太过于简单，这里只提供了工具方便使用

<br />
<br />

<a href="#Usage_1">`Usage 1 基本使用`</a>

<a href="#Usage_2">`Usage 2 多个分发器的处理`</a>

<a href="#Usage_3">`Usage 3 bcf的处理`</a>

<a id="Usage_1"/>

### Usage 1

> #### 我们有如下代码，使用函数注解的形式开启了ollvm的全部的混淆功能，使用较低的bcf概率 `-mllvm -bcf_prob=3`，在开启较高概率时，可以对部分条件进行手动patch，最终在程序运行结束之后会提示一些无法到达的基本块，手动删除即可，参考`Usage 3`
> #### * fla：控制流平坦化
> #### * sub：指令替换
> #### * split：基本块拆分
> #### * bcf：伪造控制流

![Image](https://user-images.githubusercontent.com/36320938/131239661-a9ed5ab9-c0b3-452f-aac1-a4309f5e1ab2.png)

<br />
<br />

> #### 编译后的CFG（控制流程图）

![Image](https://user-images.githubusercontent.com/36320938/131239668-a9eeea44-7b4f-40d0-a451-a40963e8b653.png)

<br />
<br />

> #### 在要去除混淆的函数入口下断（硬件断点），并运行到

![Image](https://user-images.githubusercontent.com/36320938/131239674-41fa70bc-92a9-46c9-8aac-69b564355b38.png)

<br />
<br />

> #### dump堆栈内存

![Image](https://user-images.githubusercontent.com/36320938/131239684-9f5fee7b-5601-40db-b801-1ef13b7079ab.png)

<br />
<br />

> #### dump模块内存

![Image](https://user-images.githubusercontent.com/36320938/131239688-48c28159-d7a1-4904-aa6b-e2013992c4c9.png)

<br />
<br />

> #### 编写subroutine.txt

![Image](https://user-images.githubusercontent.com/36320938/131239689-b1ff4cff-b04a-40fe-a847-9cc903f8a54b.png)

<br />
<br />

> #### 将dump的内存和subroutine.txt按如下方式布局

![Image](https://user-images.githubusercontent.com/36320938/131239694-fdaf15c3-bbe0-4a18-8922-28c7070824ae.png)

<br />
<br />

> #### 程序在短暂运行后会在当前路径下生成code.asm

![image](https://user-images.githubusercontent.com/36320938/131244309-2552dd1f-236e-4d82-8d5c-03c744d500fa.png)

<br />
<br />

> #### 拿到vs进行编译

![Image](https://user-images.githubusercontent.com/36320938/131239704-cf2f96c1-6560-4c8f-96b3-8c2cf0b607d6.png)

<br />
<br />

> #### 经过对代码进行一定的修改，使得其可以通过编译（修改后的代码上传到了Release）

![Image](https://user-images.githubusercontent.com/36320938/131239711-1efd64f4-9593-44e2-b108-7330f520b417.png)

![Image](https://user-images.githubusercontent.com/36320938/131239717-5aafef60-fe29-4d81-95a3-d8715f196b9a.png)

<br />
<br />

> #### 观察到不透明谓词

![Image](https://user-images.githubusercontent.com/36320938/131239726-16d96b8c-cda7-4064-8588-55f2820fd8de.png)

<br />
<br />

> #### 其中是dword_14001DA48，dword_14001DA4C分别是ollvm bcf功能中的int类型的全局变量x和y，地址连续，值恒为0

```shell
 mov    eax,dword_14001DA48
 
 手动修改成如下代码化简逻辑
 
 mov    eax,0
```
```shell
 cmp    dword_14001DA4C, 0ah
 cmovl  eax, r13d
 
 手动修改成如下代码化简逻辑
 
;cmp    dword_14001DA4C, 0ah
 mov    eax, r13d
```
```shell
 mov    eax, 0ff85a711h
 mov    ecx, 4ef55c6dh
 cmovl  eax, ecx
;jmp    1400012f0h
 cmp    eax,0ff85a711h
 je     loc_1400013a1
 jmp    loc_140001459

 手动修改为如下代码化简逻辑

;mov    eax, 0ff85a711h
;mov    ecx, 4ef55c6dh
;cmovl  eax, ecx
;jmp    1400012f0h
;cmp    eax,0ff85a711h
 jl     loc_140001459
 jmp    loc_1400013a1
```

<br />
<br />

> #### 按照上述的方法对代码进行一些调整，得到：

![Image](https://user-images.githubusercontent.com/36320938/131239739-3ce26cc7-0642-4b65-93b3-34316c9fc6f2.png)

<br />
<br />

> #### 使用函数返回值对比验证逻辑的正确性，通过

![Image](https://user-images.githubusercontent.com/36320938/131239742-b89d5ef9-a961-45f9-a1df-b17a1693a2ed.png)

<br />
<br />

> #### 我们在基于`angr`框架的`deflat.py`了进行了测试，结果是失败的

![image](https://user-images.githubusercontent.com/36320938/131325445-ed715066-1afc-4c12-98dc-4124e9428726.png)

<br />
<br />

<a id="Usage_2"/>

## Usage 2 multi-dispatcher

> #### 按照上述操作，我们同样地实现了更加复杂且拥有多个分发器样本的去混淆

![Image](https://user-images.githubusercontent.com/36320938/131239747-24e2cc98-e348-4c60-a282-71f057f3f4f2.png)

<br />
<br />

> #### 编译后的CFG（控制流程图）

![Image](https://user-images.githubusercontent.com/36320938/131239751-5638eb0b-3785-4561-8aa7-438fbfca90db.png)

<br />
<br />

> #### 短暂运行后，我们过滤掉了无法到达的基本块140001cd7、1400025c5（这里是程序逻辑本身就不可达，并不是BCF，因为我们没有进行patch）

![image](https://user-images.githubusercontent.com/36320938/131244335-cfc4ce82-2e8e-4a31-9e5d-d9d93616d987.png)

<br />
<br />

> #### 诸如此类的逻辑会出现不可能到达的基本块，这是因为ollvm的混淆pass会优先于llvm的优化pass

![image](https://user-images.githubusercontent.com/36320938/131240064-9807f90a-74ec-4210-84bd-16587cd2b966.png)

<br />
<br />

> #### 手动删除未到达的基本块140001cd7、1400025c5

<br />
<br />

> #### 手动删除死循环块

![Image](https://user-images.githubusercontent.com/36320938/131239781-d1edd8c9-beff-49f5-91cc-430ee5c22473.png)

<br />
<br />

> #### 手动删除死循环块的引用

![Image](https://user-images.githubusercontent.com/36320938/131239796-573bbe52-0a6d-4e0b-8d0f-c7f90ce4a950.png)

<br />
<br />

> #### 按照上述的方法对代码进行一些调整，得到：

![Image](https://user-images.githubusercontent.com/36320938/131239800-716dec62-9d1b-4d14-9d64-560793521f98.png)

<br />
<br />

> #### 使用函数返回值对比验证逻辑的正确性，通过

![Image](https://user-images.githubusercontent.com/36320938/131239801-db1b33d9-47c2-4187-8a0b-b7179cda2dab.png)

<br />

<a id="Usage_3"/>

## Usage 3 bcf patch

> #### 我们将`Usage 1`中的代码使用默认的`bcf`概率`-mllvm -bcf_prob=30`进行编译，相较于`-mllvm -bcf_prob=3`的概率明显复杂了一些。我们需要寻找类似箭头所指的`bcf块`，特点是有多个前驱，后继为分发器，这种情况是因为编译优化造成的

![Image](https://user-images.githubusercontent.com/36320938/131495997-df5b8748-426e-4bcd-9b8c-efb4fb9231fd.png)

<br />
<br />

> #### `bcf块`内容如下，一般情况两个`cmov`都是条件都是满足的，可以看做`mov`

![Image](https://user-images.githubusercontent.com/36320938/131496021-acf36a3e-0b8b-4922-b456-f1fcd5aefcf7.png)

<br />
<br />

> #### 所以逐一对`bcf块`的所有前驱进行简单的`patch`操作

![Image](https://user-images.githubusercontent.com/36320938/131496109-58f38a41-458c-4a40-909f-7d3228e71695.png)

<br />
<br />

> #### 对`bcf`块进行`nop`操作

![Image](https://user-images.githubusercontent.com/36320938/131496134-3f4ba3fc-85c0-4fa7-b923-a2712ea1257c.png)

<br />
<br />

> #### 按照`Usage 1`中演示的方法进行操作。`llux`对一些基本块进行了合并，并提示了无法到达的基本块

![Image](https://user-images.githubusercontent.com/36320938/131496180-90d8a757-86fa-47d8-b1af-e0124177e376.png)

<br />
<br />

> #### 手动删除因为基本块合并导致重定义的符号

![Image](https://user-images.githubusercontent.com/36320938/131496213-0ccc307a-d75c-4245-8247-d0702fa82caf.png)

<br />
<br />

> #### 并按照之前的方法对代码进行一定的调整，最终得到

![Image](https://user-images.githubusercontent.com/36320938/131496222-8e5b6f16-c9a3-410c-b3b8-fb5c7afd3f28.png)

<br />
<br />

> #### 使用函数返回值对比验证逻辑的正确性，通过

![Image](https://user-images.githubusercontent.com/36320938/131496229-0811f348-fa97-4be1-9100-3c032ece378d.png)

<br />

## Reference
+ [deflat](https://github.com/cq674350529/deflat)
+ [PLCT实验室维护的ollvm分支](https://github.com/isrc-cas/flounder)
+ [利用符号执行去除控制流平坦化](https://security.tencent.com/index.php/blog/msg/112)
+ [Deobfuscation: recovering an OLLVM-protected program](https://blog.quarkslab.com/deobfuscation-recovering-an-ollvm-protected-program.html)

<br />

## Github
https://github.com/llxiaoyuan/llux

## [Stantinko ollvm 浅析](Stantinko.md)

