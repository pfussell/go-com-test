# go-com-test
How to use Windows COM in Go

Looking at the documentation for DllGetClassObject, the signature is
```
HRESULT __stdcall DllGetClassObject(
  _In_  REFCLSID rclsid,
  _In_  REFIID   riid,
  _Out_ LPVOID   *ppv
);
```

This means you have to pass these three parameters to proc.Call as uintptrs (Call expects all arguments to be uintptrs).
```
package main

import "syscall"

var (
    xaSession      = syscall.NewLazyDLL("XA_Session.dll")
    getClassObject = xaSession.NewProc("DllGetClassObject")
)

func main() {
    // TODO set these variables to the appropriate values
    var rclsid, riid, ppv uintptr
    ret, _, _ := getClassObject.Call(rclsid, riid, ppv)
    // ret is the HRESULT value returned by DllGetClassObject, check it for errors
}
```
Note that you need to set the parameter values correctly, the CLSID and IID may be contained in the accompanying C header file for the library, I don't know this XA_Session library.

The ppv will in this case be a pointer to the COM object that you created. To use COM methods from Go, you can create wrapper types, given you know all the COM methods defined by it and their correct order. All COM objects support the QueryInterface, AddRef and Release functions and then additional, type specific methods.


int ConnectServer(int id)
DisconnectServer()

then what you can do to wrap that in Go is the following:
```

package xasession

import (
    "syscall"
    "unsafe"
)

// NewXASession casts your ppv from above to a *XASession
func NewXASession(ppv uintptr) *XASession {
    return (*XASession)(unsafe.Pointer(ppv))
}

// XASession is the wrapper object on which to call the wrapper methods.
type XASession struct {
    vtbl *xaSessionVtbl
}

type xaSessionVtbl struct {
    // every COM object starts with these three
    QueryInterface uintptr
    AddRef         uintptr
    Release        uintptr
    // here are all additional methods of this COM object
    ConnectServer    uintptr
    DisconnectServer uintptr
}

func (obj *XASession) AddRef() uint32 {
    ret, _, _ := syscall.Syscall(
        obj.vtbl.AddRef,
        1,
        uintptr(unsafe.Pointer(obj)),
        0,
        0,
    )
    return uint32(ret)
}

func (obj *XASession) Release() uint32 {
    ret, _, _ := syscall.Syscall(
        obj.vtbl.Release,
        1,
Let's say your XA_Session object additionally supports these two functions (again, I don't know what it really supports, you have to look that up)

 uintptr(unsafe.Pointer(obj)),
        0,
        0,
    )
    return uint32(ret)
}

func (obj *XASession) ConnectServer(id int) int {
    ret, _, _ := syscall.Syscall(
        obj.vtbl.ConnectServer, // function address
        2, // number of parameters to this function
        uintptr(unsafe.Pointer(obj)), // always pass the COM object address first
        uintptr(id), // then all function parameters follow
        0,
    )
    return int(ret)
}

func (obj *XASession) DisconnectServer() {
    syscall.Syscall(
        obj.vtbl.DisconnectServer,
        1,
        uintptr(unsafe.Pointer(obj)),
        0,
        0,
    )
}
```
