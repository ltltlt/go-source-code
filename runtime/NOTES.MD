# Golang pragma(编译指示)

## //go:noinline

禁用内联

## //go:nosplit

不进行栈溢出检测(这个检测会及时发现goroutine栈不够用了而扩张栈空间)

## //go:noescape

禁止逃逸, 自己代码也可以使用，但只能用于只有声明，主体用汇编写的函数

## //go:norace

跳过竞态检测

## //go:linkname localname importpath.name

指示编译器使用importpath.name作为localname的名字

## //go:nowritebarrier(see HACKING.md)

指示编译器如果这个函数包含write barrier时产生一个错误, 并不能阻止write barrier的产生
