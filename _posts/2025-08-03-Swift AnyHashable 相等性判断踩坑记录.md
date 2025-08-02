---
title: Swift AnyHashable 相等性判断踩坑记录
date: 2025-08-03 00:08:00 +0800
categories: [Swift 能工巧匠集]
tags: [iOS, Swift, 个人博客]
pin: true
---

## 背景

最近在一个业务需求中踩了一个 AhyHashable 相等性判断的一个大坑，抽象成以下 Demo 即可复现：

Demo 的实现非常简单：SwiftUI ScrollView 列表加上一个 Button，点击 Button 之后尝试触发 ScrollView 滚动。

其问题表现为：下列示例代码中的第 41 行无法触发 ScrollView 滚动。

```swift
import SwiftUI
import Combine

struct ItemModel: Identifiable, Equatable {
    var title: String
    var id: String = UUID().uuidString
}

struct Cell: View {
    var item: ItemModel

    var body: some View {
        Rectangle()
            .fill(.cyan)
            .border(.black, width: 1)
            .overlay(
                Text(item.title)
                    .font(.system(size: 15).weight(.bold))
                    .foregroundColor(.black)
            )
    }
}

class ViewModel: ObservableObject {
    @Published var items: [ItemModel]
}

struct ContentView: View {
    @StateObject var viewModel: ViewModel = ViewModel(items: Array(0..<50).map({ .init(title: "Cell-\($0)") }))
    @State var scrollPosition: CollectionView.ScrollPosition?
    var body: some View {
        ScrollViewReader { proxy in
           ScrollView {
              ForEach(viewModel.items) { item in
                Cell(item: item)
                 .frame(minWidth: 50, minHeight: 50)
              }
           }
           
           Button("ScrollToLast"){
             proxy.scrollTo(viewModel.items.last?.id)
           }   
        }
        
    }
}
```

且奇怪的是，将上述的代码更改为以下写法，就木有问题了：

```swift
Button("ScrollToLast"){
  if let id = viewModel.items.last?.id {
    proxy.scrollTo(id)
  }
}   
```

从 [SwiftUI.ScrollViewProxy.scrollTo](https://developer.apple.com/documentation/swiftui/scrollviewproxy/scrollto\(_:anchor:\)) 的接口声明可知：前后两种写法，唯一的区别是传递到 `proxy.scrollTo` 函数的 id 参数类型不一样，前者是个可选值，后者不是：

```swift
func scrollTo<ID>(
    _ id: ID,
    anchor: UnitPoint? = nil
) where ID : Hashable
```

- 有问题写法：id 参数类型是 AnyHashable<Optional<String>>
	
- 无问题写法：id 参数类型是 AnyHashable<String>
	

## 撇开迷雾

由于在 ScrollView 的数据源中，其元素的类型是 String，同时通过 proxy.scrollTo(id) 触发的滚动行为，其核心逻辑是判断值 id 是否存在于 ScrollView 的数据源中：如果存在，则触发滚动，如果不存在，则不触发。

由于当 id 是非可选值时能正常触发 SrollView 滚动，而当 id 是可选值的时候却失败了。

难道在 Swift 环境下，AnyHashable<"123"?> 跟 AnyHashable<"123"> 是不相等的？

写个 Demo 验证下，发现确实如此：即使 AnyHashable 中的 Warpper 实际的值是相同的，但如果其中一个是可选值，另一个是非可选值的话，相等性判断也会失败。

### Demo1

![Demo1运行截图](assets/img/posts/2020-08-03-Swift-AnyHashable/0.png)
按理来说，在执行 AnyHashable 的相等性判断时，先检查类型再判断值是否相等也是合理的。

因吹斯听的是，如果将上述 Demo 中 AnyHashable 的包装值类型由 String 改成 Int64 或者 Double 的话，结果完全不一样：即使 Warpper 类型不一致（一个是可选类型，另一个非可选类型），只要值一致，AnyHashable 的相等性判断成立：

### Demo2

<div style="display: flex !important; flex-direction: row !important; gap: 20px; justify-content: center; align-items: flex-start; margin: 20px 0; width: 100%;">
  <div style="flex: 2; text-align: center; min-width: 0;">
    <img src="/assets/img/posts/2020-08-03-Swift-AnyHashable/1.png" alt="Demo2运行截图1" style="max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); display: inline-block;">
  </div>
  <div style="flex: 1; text-align: center; min-width: 0;">
    <img src="/assets/img/posts/2020-08-03-Swift-AnyHashable/2.png" alt="Demo2运行截图2" style="max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); display: inline-block;">
  </div>
</div>

目前为止，基本可以确定问题出现在 AnyHashable 内部实现中。

### AnyHashable 中的相等性判断

#### [AnyHashable.swift](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/AnyHashable.swift)

AnyHashable 的核心功能，也就是类型擦除由 `_AnyHashableBox`协议来提供，其核心方法包括：

- `_isEqual(to` *`box`*`: _AnyHashableBox)`：提供相等性判断
	
- `_hashValue: Int` & `_hash(into` *`hasher`*`: inout Hasher)`：哈希计算
	
- `_unbox<T: Hashable>()`：包装值拆箱
	

```swift
internal protocol _AnyHashableBox {
  var _canonicalBox: _AnyHashableBox { get }

  /// Determine whether values in the boxes are equivalent.
  ///
  /// - Precondition: `self` and `box` are in canonical form.
  /// - Returns: `nil` to indicate that the boxes store different types, so
  ///   no comparison is possible. Otherwise, contains the result of `==`.
  func _isEqual(to box: _AnyHashableBox) -> Bool?
  var _hashValue: Int { get }
  func _hash(into hasher: inout Hasher)
  func _rawHashValue(_seed: Int) -> Int

  var _base: Any { get }
  func _unbox<T: Hashable>() -> T?
  func _downCastConditional<T>(into result: UnsafeMutablePointer<T>) -> Bool
}

@_unavailableInEmbedded
extension _AnyHashableBox {
  var _canonicalBox: _AnyHashableBox {
    return self
  }
}
```

其中相等性判断由协议方法 `_AnyHashableBox._canonicalBox._isEqual(to` *`box`*`:)`提供，默认情况下，`_canonicalBox`返回当前的 `_AnyHashableBox` ：
![](assets/img/posts/2020-08-03-Swift-AnyHashable/3.png)

而提供类型擦除的具体 `Box` 类型，则是在 `AnyHashable.init`构造器中。从其构造器实现可以得到几个发现：

- 当 AnyHashable 的包装值 Wrapper 是 `String`时，使用 `_ConcreteHashableBox` 作为类型擦除的 Box
	
- 如果包装值 Wrapper 遵循 `_HasCustomAnyHashableRepresentation`协议的话，使用自定义的 `Wrapper._toCustomAnyHashable` 作为包装值
	
- 其余情况（Optional<String>，自定义的值类型，引用类型）通过 `_swift_makeAnyHashableUpcastingToHashableBaseType`处理包装值，使用 `_ConcreteHashableBox` 作为类型擦除的 Box
	- 如果包装值 Wrapper 是转换成 AnyObject 的值类型，则将其解包并作为 AnyHashable 的包装值
		
	- 其余类型直接使用 `_ConcreteHashableBox` 储存 Wrapper
		

**特殊处理：** 
<div style="display: flex !important; flex-direction: row !important; gap: 20px; justify-content: center; align-items: flex-start; margin: 20px 0; width: 100%;">
  <div style="flex: 1; text-align: center; min-width: 0;">
    <img src="/assets/img/posts/2020-08-03-Swift-AnyHashable/4.png" alt="Demo2运行截图1" style="max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); display: inline-block;">
  </div>
  <div style="flex: 1; text-align: center; min-width: 0;">
    <img src="/assets/img/posts/2020-08-03-Swift-AnyHashable/5.png" alt="Demo2运行截图2" style="max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); display: inline-block;">
  </div>
</div>

**默认情况：** 
<div style="display: flex !important; flex-direction: row !important; gap: 20px; justify-content: center; align-items: flex-start; margin: 20px 0; width: 100%;">
  <div style="flex: 1; text-align: center; min-width: 0;">
    <img src="/assets/img/posts/2020-08-03-Swift-AnyHashable/6.png" alt="Demo2运行截图1" style="max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); display: inline-block;">
  </div>
  <div style="flex: 1; text-align: center; min-width: 0;">
    <img src="/assets/img/posts/2020-08-03-Swift-AnyHashable/7.png" alt="Demo2运行截图2" style="max-width: 100%; height: auto; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); display: inline-block;">
  </div>
</div>

![](assets/img/posts/2020-08-03-Swift-AnyHashable/8.png)
总结一下，AnyHashable 在构造器中使用两种不同方式处理包装值：

1. 通过 `_ConcreteHashableBox` 储存：例如 Wrapper 是 String/Optional<String> 或者业务自定义的值类型/引用类型
	
2. `_HasCustomAnyHashableRepresentation._toCustomAnyHashable`转发：Wrapper 需要遵循 `_HasCustomAnyHashableRepresentation` 协议
	

我们先来看一下 `_ConcreteHashableBox` 是如何处理相等性判断的，可以看到在进行值判断之前，会先执行类型判断，如果类型转换失败了，那么认为两者是不相等的，代入到 `AnyHashable<"1234"> == AnyHashable<Optional<"1234">>` 这个例子，由于 `String`与 `Optional<String>`不是同一个类型，所以判等失败。

因此上述 Demo1 中的行为可以得到结论：Wrapper 类型不一致导致判等失败。
![](assets/img/posts/2020-08-03-Swift-AnyHashable/9.png)
<br>

#### [\_HasCustomAnyHashableRepresentation.swift](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/AnyHashable.swift)

我们再来看一下 Wrapper 遵循 \_HasCustomAnyHashableRepresentation 协议这种情况，在 Swift 仓库中全局检索这个协议，发现确实有内置类型遵循了这个协议：

1. AnyHashable：不做特殊处理，返回自身
	
2. 基础类型（Intxx & Double & 集合类型）：特殊的 Box 自定义处理相等性判断
	1. Dictionary：\_DictionaryAnyHashableBox
		
	2. Array：\_ArrayAnyHashableBox
		
	3. Set：\_SetAnyHashableBox
		
	4. 浮点数：\_${Self}AnyHashableBox
		
	5. 整数：\_IntegerAnyHashableBox
		

![](assets/img/posts/2020-08-03-Swift-AnyHashable/10.png)![](assets/img/posts/2020-08-03-Swift-AnyHashable/11.png)

![](assets/img/posts/2020-08-03-Swift-AnyHashable/12.png)![](assets/img/posts/2020-08-03-Swift-AnyHashable/13.png)

![](assets/img/posts/2020-08-03-Swift-AnyHashable/14.png)![](assets/img/posts/2020-08-03-Swift-AnyHashable/15.png)

> **本地验证发现：**
> 
> Int64 以及 Optional<Int64> 同样遵循 `_HasCustomAnyHashableRepresentation` 协议

以 `_IntegerAnyHashableBox` 为例：

1. 当 AnyHashable Wrapper 是 Int64 或者 Optional<Int64> 时统一使用 `_IntegerAnyHashableBox` 包装值
	
2. 执行相等性判断时，没有类型转换（类型已经统一），直接比较 Wrapper 的实际值
	

![](assets/img/posts/2020-08-03-Swift-AnyHashable/16.png)
同理，对于浮点数类型的处理也是一样的。这就解释了 Demo2 中的现象：
`AnyHashable<1234>` 与 `AnyHashable<Optional<1234>>`内部在执行相等性比较时，使用了特殊的 `_IntegerAnyHashableBox` 执行处理：**没有执行类型转换，而是直接比较了两者实际的值**。
<br>

### 总结：

1. AnyHashable 在执行相等性判断时，会根据 Warpper 的类型执行不同的处理逻辑
	1. 整型 / 浮点类型 / 集合类型及其 Optional 类型：**直接比较了两者实际的值**
		
	2. 其他类型：**先执行类型判断，如果类型转换失败了，那么认为两者是不相等的。如果类型一致，再判断实际的值**
		
2. `AnyHashable<123> == AnyHashable<Optional<123>>` 判断成立，但 `AnyHashable<"123"> == AnyHashable<Optional<"123">>` 判断不成立，在业务视觉下难以令人理解且十分容易犯错出 Bug。
	

## 怎么解决

对于一开头 Demo 中遇到的问题，可以通过添加一层 Optional Chain（if let）来解决：
![](assets/img/posts/2020-08-03-Swift-AnyHashable/17.png)
