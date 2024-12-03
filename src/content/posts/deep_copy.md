---
title: Deep Copy
published: 2024-06-24
description: ""
tags: ["Golang"]
category: 开发
draft: false
---

# Deep Copy

为解决循环指针而设计的深度拷贝，目前 `chan`/`map`/`func` 还没有实现。

## 源码

```golang
package main

import (
	"reflect"
)

func DeepCopy[T any](src, dst *T) {
	srv := reflect.ValueOf(src)
	srv = srv.Elem()
	drv := reflect.ValueOf(dst)
	drv = drv.Elem()
	addrMap := map[uint64]reflect.Value{}
	handleStruct(srv, drv, addrMap)
}

func genKey(src reflect.Value) uint64 {
	switch src.Kind() {
	case reflect.Pointer, reflect.Chan, reflect.Map, reflect.UnsafePointer, reflect.Func, reflect.Slice:
		return uint64(src.Pointer())<<6 + uint64(src.Kind())<<1
	default:
		return uint64(src.UnsafeAddr())<<6 + uint64(src.Kind())<<1 + 1
	}
}

func handlePointer(src, dst reflect.Value, addrMap map[uint64]reflect.Value) {
	src = src.Elem()
	addr := genKey(src)
	ndst, ok := addrMap[addr]
	if !ok {
		ndst = reflect.New(src.Type()).Elem()
		addrMap[addr] = ndst
		switch src.Kind() {
		case reflect.Struct:
			handleStruct(src, ndst, addrMap)
		case reflect.Interface:
			handleInterface(src, ndst, addrMap)
		case reflect.Pointer:
			handlePointer(src, ndst, addrMap)
		case reflect.Array:
			handleArray(src, ndst, addrMap)
		case reflect.Slice:
			handleSlice(src, ndst, addrMap)
		case reflect.Chan:
		case reflect.Func:
		case reflect.Map:
		default:
			ndst.Set(src)
		}
	}
	dst.Set(ndst.Addr())
}

func handleStruct(src, dst reflect.Value, addrMap map[uint64]reflect.Value) {
	addrMap[genKey(src)] = dst
	for i := 0; i < src.NumField(); i++ {
		srcf := src.Field(i)
		addr := genKey(srcf)
		dstf, ok := addrMap[addr]
		if !ok {
			ndstf := dst.Field(i)
			addrMap[addr] = ndstf
			switch srcf.Kind() {
			case reflect.Struct:
				handleStruct(srcf, ndstf, addrMap)
			case reflect.Interface:
				handleInterface(srcf, ndstf, addrMap)
			case reflect.Pointer:
				handlePointer(srcf, ndstf, addrMap)
			case reflect.Array:
				handleArray(srcf, ndstf, addrMap)
			case reflect.Slice:
				handleSlice(srcf, ndstf, addrMap)
			case reflect.Chan:
			case reflect.Func:
			case reflect.Map:
			default:
				ndstf.Set(srcf)
			}
		} else {
			dst.Field(i).Set(dstf)
		}
	}
}

func handleInterface(src, dst reflect.Value, addrMap map[uint64]reflect.Value) {
	src = src.Elem()
	switch src.Kind() {
	case reflect.Struct:
		handleStruct(src, dst, addrMap)
	case reflect.Interface:
		handleInterface(src, dst, addrMap)
	case reflect.Pointer:
		handlePointer(src, dst, addrMap)
	case reflect.Array:
		handleArray(src, dst, addrMap)
	case reflect.Slice:
		handleSlice(src, dst, addrMap)
	case reflect.Chan:
	case reflect.Func:
	case reflect.Map:
	default:
		dst.Set(src)
	}
}

func handleArray(src, dst reflect.Value, addrMap map[uint64]reflect.Value) {
	for i := 0; i < src.Len(); i++ {
		srcf := src.Index(i)
		addr := genKey(srcf)
		dstf, ok := addrMap[addr]
		if !ok {
			ndstf := dst.Index(i)
			addrMap[addr] = ndstf
			switch srcf.Kind() {
			case reflect.Struct:
				handleStruct(srcf, ndstf, addrMap)
			case reflect.Interface:
				handleInterface(srcf, ndstf, addrMap)
			case reflect.Pointer:
				handlePointer(srcf, ndstf, addrMap)
			case reflect.Array:
				handleArray(srcf, ndstf, addrMap)
			case reflect.Slice:
				handleSlice(srcf, ndstf, addrMap)
			case reflect.Chan:
			case reflect.Func:
			case reflect.Map:
			default:
				ndstf.Set(srcf)
			}
		} else {
			dst.Index(i).Set(dstf)
		}
	}
}

func handleSlice(src, dst reflect.Value, addrMap map[uint64]reflect.Value) {
	addrMap[genKey(src)] = dst
	dst.Grow(src.Len() - dst.Len())
	dst.SetLen(src.Len())
	for i := 0; i < src.Len(); i++ {
		srcf := src.Index(i)
		addr := genKey(srcf)
		dstf, ok := addrMap[addr]
		if !ok {
			ndstf := dst.Index(i)
			addrMap[addr] = ndstf
			switch srcf.Kind() {
			case reflect.Struct:
				handleStruct(srcf, ndstf, addrMap)
			case reflect.Interface:
				handleInterface(srcf, ndstf, addrMap)
			case reflect.Pointer:
				handlePointer(srcf, ndstf, addrMap)
			case reflect.Array:
				handleArray(srcf, ndstf, addrMap)
			case reflect.Slice:
				handleSlice(srcf, ndstf, addrMap)
			case reflect.Chan:
			case reflect.Func:
			case reflect.Map:
			default:
				ndstf.Set(srcf)
			}
		} else {
			dst.Index(i).Set(dstf)
		}
	}
}
```
