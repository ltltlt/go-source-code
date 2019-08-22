# Golang 看runtime的经验

## pragma(编译指示)

### //go:noinline

禁用内联

### //go:nosplit

不进行栈溢出检测(这个检测会及时发现goroutine栈不够用了而扩张栈空间)

### //go:noescape

禁止逃逸, 自己代码也可以使用，但只能用于只有声明，主体用汇编写的函数

### //go:norace

跳过竞态检测

### //go:linkname localname importpath.name

指示编译器使用importpath.name作为localname的名字

### //go:nowritebarrier(see HACKING.md)

指示编译器如果这个函数包含write barrier时产生一个错误, 并不能阻止write barrier的产生

## register

- FP: frame pointer, 栈帧的基本地址(但弹出一个栈帧时, SP会赋为FP), 函数参数不在FP~SP之间, 而是在FP上, 0(FP)是第一个参数
- SP: 栈顶指针, 访问栈上变量都是SP+...
- PC: program counter, 程序计数器, 下一条指令运行的地址
- SB: static base pointer, 静态符号基址
