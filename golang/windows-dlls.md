# Windows Dlls

[原文](https://github.com/golang/go/wiki/WindowsDLLs)

## 调用Windows DLL

Go可以用几种不同的方式调用Windows原生函数

1. 动态加载DLL，然后调用其中的函数。你可以通过`SyscallX`调用函数。
   （`X`是参数的数量。如果函数的传入参数比这要少，例如需要9个传了7个，`Syscall9`还是可以工作，只需要在调用`Syscall9`是第二个参数指定7）

一个用这种方法调用Windows DLL的简单Go程序：

```go
package main

import (
	"fmt"
	"syscall"
	"unsafe"
)

func abort(funcname string, err error) {
    panic(fmt.Sprintf("%s failed: %v", funcname, err))
}

var (
    kernel32, _        = syscall.LoadLibrary("kernel32.dll")
    getModuleHandle, _ = syscall.GetProcAddress(kernel32, "GetModuleHandleW")
    
    user32, _     = syscall.LoadLibrary("user32.dll")
    messageBox, _ = syscall.GetProcAddress(user32, "MessageBoxW")
)

const (
    MB_OK                = 0x00000000
    MB_OKCANCEL          = 0x00000001
    MB_ABORTRETRYIGNORE  = 0x00000002
    MB_YESNOCANCEL       = 0x00000003
    MB_YESNO             = 0x00000004
    MB_RETRYCANCEL       = 0x00000005
    MB_CANCELTRYCONTINUE = 0x00000006
    MB_ICONHAND          = 0x00000010
    MB_ICONQUESTION      = 0x00000020
    MB_ICONEXCLAMATION   = 0x00000030
    MB_ICONASTERISK      = 0x00000040
    MB_USERICON          = 0x00000080
    MB_ICONWARNING       = MB_ICONEXCLAMATION
    MB_ICONERROR         = MB_ICONHAND
    MB_ICONINFORMATION   = MB_ICONASTERISK
    MB_ICONSTOP          = MB_ICONHAND
    
    MB_DEFBUTTON1 = 0x00000000
    MB_DEFBUTTON2 = 0x00000100
    MB_DEFBUTTON3 = 0x00000200
    MB_DEFBUTTON4 = 0x00000300
)

func MessageBox(caption, text string, style uintptr) (result int) {
    var nargs uintptr = 4
    ret, _, callErr := syscall.Syscall9(uintptr(messageBox),
            nargs,
    	    0,
            uintptr(unsafe.Pointer(syscall.StringToUTF16Ptr(text))),
            uintptr(unsafe.Pointer(syscall.StringToUTF16Ptr(caption))),
            style,
            0,
            0,
            0,
            0,
            0)
    if callErr != 0 {
        abort("Call MessageBox", callErr)
    }
    result = int(ret)
    return
}

func GetModuleHandle() (handle uintptr) {
    var nargs uintptr = 0
    if ret, _, callErr := syscall.Syscall(uintptr(getModuleHandle),nargs,0,0,0);callErr != 0 {
        abort("Call GetModuleHandle", callErr)
    } else {
        handle = ret
    }
    return
}

func main() {
    defer syscall.FreeLibrary(kernel32)
    defer syscall.FreeLibrary(user32)
    
    fmt.Printf("Return: %d\n", MessageBox("Done Title", "This test is Done.", MB_YESNOCANCEL))
}

func init() {
    fmt.Print("Starting Up\n")
}
```


2. 使用`syscall.NewProc`代替`syscall.GetProcAddress`。这些基本上是系统调用的一些辅助方法，上面看到的这些，并且只在Windows可用：

[`https://go.dev/src/syscall/dll_windows.go`](https://go.dev/src/syscall/dll_windows.go)

```go
package main

import (
    "fmt"
    "syscall"
    "unsafe"
)

func main() {
    var mod = syscall.NewLazyDLL("user32.dll")
    var proc = mod.NewProc("MessageBoxW")
    var MB_YESNOCANCEL = 0x00000003
    
    ret, _, _ := proc.Call(0,
        uintptr(unsafe.Pointer(syscall.StringToUTF16Ptr("This test is Done."))),
        uintptr(unsafe.Pointer(syscall.StringToUTF16Ptr("Done Title"))),
        uintptr(MB_YESNOCANCEL))
    fmt.Printf("Return: %d\n", ret)
}
```


3. 通过“链接”库，使用“[CGO](https://github.com/golang/go/wiki/cgo)”方法（这个方法可以运行在Linux和Windows）。

例如：

```go
import "C"
...
C.MessageBoxW(...)
```

查看[CGO](https://github.com/golang/go/wiki/cgo)获取更多细节。
