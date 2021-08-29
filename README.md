# llux
## ollvm de-obfuscator

### Description

* 无需条件约束，在遇到BCF增加的死循环块的时候，依然可以完成路径遍历
* 针对ollvm进行了充分地优化，速度对于符号执行框架angr速度较快，且可以处理复杂样本
* 暂未开源，因为这样会使得混淆对抗太过于简单，这里只提供了工具方便使用

### Usage

我们有如下代码，使用函数注解的形式开启了ollvm的全部的混淆功能
* fla：控制流平坦化
* sub：指令替换
* split：基本块拆分
* bcf：伪造控制流

![Image](https://user-images.githubusercontent.com/36320938/131239661-a9ed5ab9-c0b3-452f-aac1-a4309f5e1ab2.png)

使用较低的bcf概率 `-mllvm -bcf_prob=3`，在开启较高概率时，可以对部分条件进行手动patch，最终在程序运行结束之后会提示一些无法到达的基本块，手动删除即可

编译后的CFG（控制流程图）

![Image](https://user-images.githubusercontent.com/36320938/131239668-a9eeea44-7b4f-40d0-a451-a40963e8b653.png)

在要去除混淆的函数入口下断（硬件断点），并运行到

![Image](https://user-images.githubusercontent.com/36320938/131239674-41fa70bc-92a9-46c9-8aac-69b564355b38.png)

dump堆栈内存

![Image](https://user-images.githubusercontent.com/36320938/131239684-9f5fee7b-5601-40db-b801-1ef13b7079ab.png)

dump模块内存

![Image](https://user-images.githubusercontent.com/36320938/131239688-48c28159-d7a1-4904-aa6b-e2013992c4c9.png)

编写subroutine.txt

![Image](https://user-images.githubusercontent.com/36320938/131239689-b1ff4cff-b04a-40fe-a847-9cc903f8a54b.png)

将dump的内存和subroutine.txt按如下方式布局

![Image](https://user-images.githubusercontent.com/36320938/131239694-fdaf15c3-bbe0-4a18-8922-28c7070824ae.png)

程序在短暂运行后会在当前路径下生成code.asm

![Image](https://user-images.githubusercontent.com/36320938/131239696-aee26fb3-1ced-4303-8cb5-ae0cccd5078e.png)

拿到vs进行编译

![Image](https://user-images.githubusercontent.com/36320938/131239704-cf2f96c1-6560-4c8f-96b3-8c2cf0b607d6.png)

经过对代码进行一定的修改，使得其可以通过编译（修改后的代码上传到了Release）

![Image](https://user-images.githubusercontent.com/36320938/131239711-1efd64f4-9593-44e2-b108-7330f520b417.png)

![Image](https://user-images.githubusercontent.com/36320938/131239717-5aafef60-fe29-4d81-95a3-d8715f196b9a.png)

观察到不透明谓词

![Image](https://user-images.githubusercontent.com/36320938/131239726-16d96b8c-cda7-4064-8588-55f2820fd8de.png)

其中是dword_14001DA48，dword_14001DA4C分别是ollvm bcf功能中的int类型的全局变量x和y，地址连续，值恒为0

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

按照上述的方法对代码进行一些调整，得到：

![Image](https://user-images.githubusercontent.com/36320938/131239739-3ce26cc7-0642-4b65-93b3-34316c9fc6f2.png)

使用函数返回值对比验证逻辑的正确性，通过

![Image](https://user-images.githubusercontent.com/36320938/131239742-b89d5ef9-a961-45f9-a1df-b17a1693a2ed.png)

按照上述操作，我们同样地实现了更加复杂样本的去混淆

![Image](https://user-images.githubusercontent.com/36320938/131239747-24e2cc98-e348-4c60-a282-71f057f3f4f2.png)

编译后的CFG（控制流程图）

![Image](https://user-images.githubusercontent.com/36320938/131239751-5638eb0b-3785-4561-8aa7-438fbfca90db.png)

短暂运行后，我们过滤掉了无法到达的基本块140001cd7、1400025c5（这里是程序逻辑本身就不可达，并不是BCF，因为我们没有进行patch）

![Image](https://user-images.githubusercontent.com/36320938/131239761-9a50ec54-8233-4f43-90f1-4de96c2f0775.png)

诸如此类的逻辑会出现不可能到达的基本块，这是因为ollvm的混淆pass会优先与于llvm的优化pass

![image](https://user-images.githubusercontent.com/36320938/131240064-9807f90a-74ec-4210-84bd-16587cd2b966.png)

手动删除未到达的基本块140001cd7、1400025c5

手动删除死循环块

![Image](https://user-images.githubusercontent.com/36320938/131239781-d1edd8c9-beff-49f5-91cc-430ee5c22473.png)

手动删除死循环块的引用

![Image](https://user-images.githubusercontent.com/36320938/131239796-573bbe52-0a6d-4e0b-8d0f-c7f90ce4a950.png)

按照上述的方法对代码进行一些调整，得到：

![Image](https://user-images.githubusercontent.com/36320938/131239800-716dec62-9d1b-4d14-9d64-560793521f98.png)

使用函数返回值对比验证逻辑的正确性，通过

![Image](https://user-images.githubusercontent.com/36320938/131239801-db1b33d9-47c2-4187-8a0b-b7179cda2dab.png)

