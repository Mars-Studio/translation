## 第七章 错误处理
没有人愿意处理错误。处理错误很乏味，还容易出出错，并且也没有计划程序如何正确运行有乐趣。但是，错误处理非常重要，无论你多么不喜欢，软件因为薄弱的错误处理而失败要更糟糕。

庆幸的是，OCaml提供了强大的工具来可靠地处理错误，且把痛处降至最低。本章我们会讨论OCaml中的几种处理错误的方法，并且给出了一些如何设计接口以简化错误处理的建议。

开始，我们先介绍OCaml中报告错误的两种基本方法：带错误的返回值和异常。
>> Error-Aware return types

### 带错误的返回值
OCaml中抛出一个错误最好的方法就是把这个错误包含到返回值中。考虑`List`模块中`find`函数的类型：
```ocaml
# List.find;;
- : 'a list -> f:('a -> bool) -> 'a option = <fun>
```
返回类型中的`option`表明此函数在查找一个相符的元表时可能会失败：
```ocaml
# List.find [1;2;3] ~f:(fun x -> x >= 2) ;;
- : int option = Some 2
# List.find [1;2;3] ~f:(fun x -> x >= 10) ;;
- : int option = None
```
在函数返回值中包含错误信息就要求调用者显式处理它，允许调用者来决定是从这个错误中恢复还是将其向前传播。

下面看一下`computer_bounds`函数。该函数接收一个列表和一个比较函数，通过查找最大和最小的列表元素返回列表的上下边界。提取列表中最大和最小元素用的是`List.hd`和`List.last`，它们在遇到空列表时会返回`None`：
```ocaml
# let compute_bounds ~cmp list =
let sorted = List.sort ~cmp list in
match List.hd sorted, List.last sorted with
| None,_ | _, None -> None
| Some x, Some y -> Some (x,y)
;;
val compute_bounds : cmp:('a -> 'a -> int) -> 'a list -> ('a * 'a) option =
<fun>
```
`match`语句用以处理错误分支，将`hd`或`last`返回的`None`传递给`computer_bounds`的返回值。

另一方面，下面的`find_mismatches`函数中，计算过程中碰到的错误不会传递给函数的返回值。`find_mismatches`接收两个哈希表作为参数，在一个表中查找与另一个表中对应的数据不同的键值。因此，在一个表中找不到一个键值不属于错误：
```ocaml
# let find_mismatches table1 table2 =
Hashtbl.fold table1 ~init:[] ~f:(fun ~key ~data mismatches ->
match Hashtbl.find table2 key with
| Some data' when data' <> data -> key :: mismatches
| _ -> mismatches
)
;;
val find_mismatches : ('a, 'b) Hashtbl.t -> ('a, 'b) Hashtbl.t -> 'a list =
<fun>
```
使用`option`编码错误凸显了这样一个事实：一个特定的结果，如在列表中没有找到某元素，是一个错误还是一个合理的结果，这一点并不明确。这依赖于你程序中更大的上下文，是一个通用库无法预知的。带错误的返回值的优势就在于两种情况它都适用。

#### 编码错误和结果
`option`在报告错误方面不总是有足够的表达力。特别是当你把一个错误编码成`None`时，就没有地方来说明错误的性质了。

`Result.t`就是为了解决此不足的。类型定义如下：
```ocaml
module Result : sig
   type ('a,'b) t = | Ok of 'a
                    | Error of 'b
end
```
`Result.t`本质上是一个参数化的`option`，给错误实例赋于存储其它信息的能力。像`Some`和`None`一样，构造器`Ok`和`Error`由`Core.Std`提升到顶层作用域。因此，我们可以这样写：
```ocaml
# [ Ok 3; Error "abject failure"; Ok 4 ];;
- : (int, string) Result.t list = [Ok 3; Error "abject failure"; Ok 4]
```
而不需要先打开`Result`模块。

#### `Error`和`Or_error`
`Result.t`给了你完全的自由来选择错误值的类型，但通常规范错误类型是有用的。别的不说，它会简化自动化通用错误处理模式的工具函数。

但选什么类型呢？把错误表示为字符串会更好呢？还是像XML这样更加结构化的表达？或者其它的什么？

`Core`的答案是`Error.t`类型，它试图在效率、方便性和对错误表达的控制力之间取得一个很好的平衡。

效率问题一开始并不明显。但生成错误消息是很昂贵的。一个值的ASCII表达可能相当耗时，特别是它包含不易转换的数值数据时。

`Error`使用惰性求值避开了这个问题，`Error.t`允许你推迟错误生成除非你需要它，这就意味着很多时候你根本不需要去构造它。当然你也可以直接从一个字符串构造一个错误：
```ocaml
# Error.of_string "something went wrong";;
- : Error.t = something went wrong
```
但你也可以从一个形实转换程序（thunk）构造`Error.t`，即，一个接收单独一个`unit`类型参数的函数：
```ocaml
# Error.of_thunk (fun () ->
    sprintf "something went wrong: %f" 32.3343);;
- : Error.t = something went wrong: 32.334300
```
这时，我们就可以从`Error`的惰性求值获益，因为除非`Error.t`被转换为字符串，否则形实转换程序不会被调用。


#### bind and Other Error Handling Idioms

### Exceptions
#### Helper Functions for Throwing Exceptions
#### Exception Handlers
#### Cleaning Up in the Presence of Exceptions
#### Catching Specific Exceptions
#### Backtraces
#### From Exceptions to Error-Aware Types and Back Again

### Choosing an Error-Handling Strategy