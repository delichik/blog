---
title: LMDB - 低gc压力缓存
published: 2024-06-24
description: ""
tags: ["Golang", "缓存"]
category: 开发
draft: false
---

# LMDB - 低gc压力缓存

在Golang中创建一个缓存库最头痛的问题就是gc，大家也集思广益地想出了各种避免的办法，
今天看到了一个比较巧妙的思想，就想要实现一下试试看。

## 原理

Golang中的保存一个变量，第一个的想法都是直接new一个，然后将地址保存在内存中的map里面，
这样的问题就是gc每次都会去扫描map中的指针指向的值，在map中东西特别多的时候，gc效率将会严重下降。

这个方法的实现中，我也是简单的将指针保存在map中，不一样的是，指针不是new出来的，而是变量的地址。
这个方法中，向Set一个变量而不是指针，然后将变量复制到lmdb中的数组里，然后取出其在数组中的变量的地址，
再放入map中保存，后续使用时就直接从map中取出这个指针就行。

由于每个指针指向的值实际上保留在数组中，gc扫描将不会去检查指针指向的值，又因为指针本身实际是简单的uintptr，
整个map将直接不被扫描，效率将大大提高

## 实例源代码

```golang
package lmdb

const defaultFragmentCount = 128
const fragmentSize = 128

type LMDBIndex struct {
	Fragment uint
	Index    uint
}

type Item[T any] struct {
	realItem T
	deleted  bool
}

type Items[T any] struct {
	items        []Item[T]
	deletedCount uint
}

type LMDB[K comparable, V any] struct {
	length  uint
	m       map[K]LMDBIndex
	storage []Items[V]
}

func NewLMDB[K comparable, V any]() *LMDB[K, V] {
	return &LMDB[K, V]{
		length:  0,
		m:       make(map[K]LMDBIndex),
		storage: make([]Items[V], 0, defaultFragmentCount),
	}
}

func (m *LMDB[K, V]) Get(key K) (*V, bool) {
	index, ok := m.m[key]
	if !ok {
		return nil, false
	}

	t := m.storage[index.Fragment].items[index.Index]
	return &t.realItem, true
}

func (m *LMDB[K, V]) getNextIndex() LMDBIndex {
	for fi, fragment := range m.storage {
		if fragment.deletedCount == 0 {
			continue
		}

		for ii := range fragmentSize {
			if ii >= len(fragment.items) {
				fragment.items = append(fragment.items, Item[V]{deleted: true})
				m.storage[fi] = fragment
				return LMDBIndex{
					Fragment: uint(fi),
					Index:    uint(ii - 1),
				}
			}
			item := fragment.items[ii]
			if !item.deleted {
				continue
			}
			return LMDBIndex{
				Fragment: uint(fi),
				Index:    uint(ii - 1),
			}
		}
	}

	m.storage = append(m.storage, Items[V]{
		items:        make([]Item[V], 1, fragmentSize),
		deletedCount: fragmentSize,
	})

	return LMDBIndex{
		Fragment: uint(len(m.storage) - 1),
		Index:    0,
	}
}

func (m *LMDB[K, V]) Set(key K, value V) {
	index, ok := m.m[key]
	if ok {
		m.storage[index.Fragment].items[index.Index] = Item[V]{
			realItem: value,
			deleted:  false,
		}
	} else {
		index = m.getNextIndex()
		m.m[key] = index
		fragment := m.storage[index.Fragment]
		fragment.items[index.Index] = Item[V]{
			realItem: value,
			deleted:  false,
		}
		m.storage[index.Fragment] = fragment
	}
	m.length++
}

func (m *LMDB[K, V]) Delete(key K) {
	index, ok := m.m[key]
	if !ok {
		return
	}
	fragment := m.storage[index.Fragment]
	fragment.items[index.Index] = Item[V]{
		deleted: true,
	}
	fragment.deletedCount++
	m.storage[index.Fragment] = fragment
	m.length--
	delete(m.m, key)
}

func (m *LMDB[K, V]) Len() uint {
	return m.length
}

type TestStruct struct {
	Str string
	Num int
}
```