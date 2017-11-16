---
title: RAC源码学习<二> Haskell
date: 2017-11-15 16:41:56
categories: 源码阅读
tags: 源码阅读
---

# B10-Haskell趣味指南

为什么读这本书？
> swift 某些方面和他很像        
> 培养函数式编程思想      
> 函子

<!--more-->

## 从零开始

### List 入门

两个List合并操作，通过 ++ 运算子实现。缺点会遍历整个List

```
ghci> [1,2,3,4] ++ [9,10,11,12]
[1,2,3,4,9,10,11,12]
```

用 : 运算子

```
ghci> 'A':"SMALL CAT"
A SMALL CAT
```

按照索引取得 List 中的元素，可以使用 !! 运算子

```
ghci> "Steve Buscemi" !! 6
'B'
```

List 的 > 和 >= 运算符 是卓个元素比较
 
 
<div align=center>
<img src="./list_monster.png" width = "432" height = "218" alt="内存分部图" align=center/>
</div>
 
 
```
ghci> head [5,4,3,2,1] 
5
ghci> tail [5,4,3,2,1] 
[4,3,2,1]
ghci> last [5,4,3,2,1] 
1
ghci> init [5,4,3,2,1] 
[5,4,3,2]
ghci> length [5,4,3,2,1] 
5
```
用 null 去检查 List 是否空，不要用[]

```
ghci> null [1,2,3] 
False 
ghci> null [] 
True
```

reverse将一个 List 反转         
take 返回一个 List 的前几个元素

```
ghci> take 3 [5,4,3,2,1] 
[5,4,3] 
ghci> take 1 [3,9,3] 
[3] 
ghci> take 5 [1,2] 
[1,2] 
ghci> take 0 [6,6,6] 
[]
```

drop 他会删除 List 中的前几个元素         
maximum 返回一个 List 中最大的那个元素。minimun 返回最小的                
sum 返回一个 List 中所有元素的和。product 返回一个 List 中所有元素的和。         
elem 判断一个元素是否在包含于一个 List，通常以中缀函数的形式调 用它。

```
ghci> 4 `elem` [3,4,5,6] 
True 
ghci> 10 `elem` [3,4,5,6] 
False
```
 
Range 是构造 List 方法之一，提供步长设定。避免在Range中使用浮点数。            
取前24个13的倍数

```
take 24 [13,26..]
```
关于 cycle 函数，接受一个 List 做参数并返回一个无线的 List

```
ghci> take 10 (cycle [1,2,3]) 
[1,2,3,1,2,3,1,2,3,1] 
ghci> take 12 (cycle "LOL ") 
"LOL LOL LOL "
```

关于 repeat 函数，接受一个值做参数并返回一个该值的无限 List 

```
ghci> take 10 (repeat 5)
[5,5,5,5,5,5,5,5,5,5]
```
            
关于 replicate 函数，返回包含相同元素的 List

```
ghci> replicate 3 10 
[10,10,10]
```

关于 List Comprehension ,类似数学中的集合表达式

```
ghci> [x*2 | x <- [1..10]] 
[2,4,6,8,10,12,14,16,18,20]

ghci> [x*2 | x <- [1..10], x*2 >= 12] 
[12,14,16,18,20]
```

Tuple 元组 存储不同类型的数据单元         
关于 fst 函数，将返回一个序对的首项        
关于 snd 函数，将返回一个序对的尾项          
上面两个函数仅对序列对(Pair)有效。        

关于 zip 函数，用来生成一组序对(Pair)的 List。

```
ghci> zip [1,2,3,4,5] [5,5,5,5,5] 
[(1,5),(2,5),(3,5),(4,5),(5,5)] 
ghci> zip [1 .. 5] ["one", "two", "three", "four", "five"]
[(1,"one"),(2,"two"),(3,"three"),(4,"four"),(5,"five")]
```

## Typed and Typeclasses

Typed 理解为类型 Typeclasses 理解为通用接口

:t 命令后跟任何可用的表达式，即可得到该表达式的类型。

```
addThree :: Int -> Int -> Int -> Int 
addThree x y z = x + y + z
```

=> 符号，他左边的部分叫做类型约束。相等函数取两个相同类型的值作为参数并返回一个布林值，而这两个参数的类型同在 Eq 类之中

```
ghci> :t (==) 
(==) :: (Eq a) => a -> a -> Bool
```

Show 的成员为可用的字符串。有点像类型转换字符串。常用函数表示 show。它可以取任一 Show 的成员类型并将其转为字符串。         
Read 是和 Show 相反的 Typeclass。ghci通过调用 read 后面跟的那部分来确定其类型。

```
ghci> read "[1,2,3,4]" ++ [3] 
[1,2,3,4,3]
ghci> read "4"  
报错
ghci> read "5" :: Int 
5
```

## 函数的语法

### Pattern matching

__函数的模式匹配__

__List 的模式匹配__

```
sum' :: (Num a) => [a] -> a
sum' [] = 0
sum' (x:xs) = x + sum' xs
```

重定义一个 sum‘ 函数，它的类型是 `[a] -> a` ,其中 a 满足 Num 的类型约束。下面是模式匹配，第一个是特定类型匹配，最后一个是通用类型匹配。

还有一种 as 模式。如 xs@(x:y:ys)  

```
capital :: String -> String 
capital "" = "Empty string, whoops!"
capital all@(x:xs) = "The first letter of " ++ all ++ " is " ++ [x]
```

使用 as 模式通常就是为了在较大的模式中保留对整体的引用，减少重复性的工作

__Guards__

guard 由跟在函数名及参数后面的竖线标志

```
myCompare :: (Ord a) => a -> a -> Ordering 
a `myCompare` b

| a > b = GT 
| a == b = EQ 
| otherwise = LT
```

__where__

```
bmiTell :: (RealFloat a) => a -> a -> String 
bmiTell weight height
	| bmi <= skinny = "You're underweight, you emo, you!"
	| bmi <= normal = "You're supposedly normal. Pffft, I bet you're ugly!" | bmi <= fat = "You're fat! Lose some weight, fatty!"
	| otherwise = "You're a whale, congratulations!"
	where bmi = weight / height ^ 2
		   skinny = 18.5
		   normal = 25.0
		   fat = 30.0
		   
		   
initials :: String -> String -> String 
initials firstname lastname = [f] ++ ". " ++ [l] ++ "."
	where (f:_) = firstname 
         (l:_) = lastname
```

__let__

let 的格式为 let [bindings] in [expressions]。在 let 中绑定的名字仅对 in 部分可见。

```
cylinder :: (RealFloat a) => a -> a -> a 
cylinder r h =
	let sideArea = 2 * pi * r * h 
	topArea = pi * r ^2 
	in sideArea + 2 * topArea
```

let 绑定本身是个表达式，而 where 绑定则是个语法结构。if 语句也是一个表达式。

__Case expressions__

case ‘表达式’的语法

```
case expression of pattern -> result  
pattern -> result 
pattern -> result 
...
```


## 递归

### 实现 Maximum

主要就是，寻找问题中的关系的过程

```
function Maximum' 
	| [] []
	| [x:xs] if x > Maximum' xs return x else return Maximum' xs
```

固定模式：先定义一个边界条件，再定义个函数，让它从一堆元素中取一个并做点事情后，把余下的元素重新交给这个函数。

## 高阶函数

Haskell 惰性特征

### map 和 filter

### lambda 

```
ghci> map (\(a,b) -> a + b) [(1,2),(3,5),(6,3),(2,6),(2,5)] 
[3,8,9,8,7]
```

### fold

一个 fold 取一个二元函数，一个初始值 (我喜欢管它叫累加值) 和一个 需要折叠的 List。
首先看下 foldl 函数，也叫做左折叠。     

```
sum' :: (Num a) => [a] -> a 
sum' xs = foldl (\acc x -> acc + x) 0 xs
```

右折叠 foldr 的行为与左折叠相似，只是累加值从 List 的右边开始。    

对于左折叠和右折叠的选择，对于使用(++)往 List 后面追加元素的效率比使用(:)低得多。    

foldl1 与 foldr1 的行为与 foldl 和 foldr 相似，只是你无需明确提供 初始值。他们假定 List 的首个 (或末尾) 元素作为起始值，并从旁边的元素 开始折叠。         
scanl 和 scanr 与 foldl 和 foldr 相似，只是它们会记录下累加值的 所有状态到一个 List。也有 scanl1 和 scanr1。

看 $ 函数。它也叫作函数调用符。$ 函数调用符的优先级最低。          
试想有 这个表达式：sum (map sqrt [1..130])。由于低优先级的 $，我们可以将 其改为 sum $ map sqrt [1..130]

### Function composition

函数的组合 . (f#g)(x) = f(g(x)); 函数组合是右结合的。表达式 f(g(z(x))) 与 (f . g . z) x 等价。

## 模组 (Modules)

:m 装载模组

```
ghci> :m Data.List
ghci> :m Data.List Data.Map Data.Set
仅装载 Data.List 模组 nub 和 sort 
import Data.List (nub，sort)
装载 Data.List 模组时，把 nub 除掉
import Data.List hiding (nub)
避免重名函数装载 在调用模组函数时，需要指定作用域 Data.Map.filter
import qualified Data.Map
可以用别名，调用 filter 直接 M.filter
import qualified Data.Map as M

```

### Data.List

intersperse 取一个元素与 List 作参数，并将该元素置于 List 中每对元素的中间                
       
```          
ghci> intersperse '.' "MONKEY" 
"M.O.N.K.E.Y" 
ghci> intersperse 0 [1,2,3,4,5,6] 
[1,0,2,0,3,0,4,0,5,0,6]          
```       

intercalate取两个List作参数.它会将第一个 List 交叉插入第二个 List中间，并返回一个 List.
```              
ghci> intercalate " " ["hey","there","guys"] 
"hey there guys" 
ghci> intercalate [0,0,0] [[1,2,3],[4,5,6],[7,8,9]] 
[1,2,3,0,0,0,4,5,6,0,0,0,7,8,9]
```                                

这里 transpose 函数可以反转一组 List 的 List。

```                   
ghci> transpose [[1,2,3],[4,5,6],[7,8,9]] 
[[1,4,7],[2,5,8],[3,6,9]] 

ghci> transpose ["hey","there","guys"] 
["htg","ehu","yey","rs","e"]       
```                                           
 
这里concat 把一组 List 连接为一个 List。

```                
ghci> concat ["foo","bar","car"] 
"foobarcar" 

ghci> concat [[3,4,5],[2,3,4],[2,1,1]] 
[3,4,5,2,3,4,2,1,1]
```             
 


concatMap 函数与 map 一个 List 之后再 concat 它等价。

```             
ghci> concatMap (replicate 4) [1..3] 
[1,1,1,1,2,2,2,2,3,3,3,3]
```          

__比较多的函数介绍__

## 构造我们自己的 Types 和 Typeclasses
### Algebraic Data Types

定义一个自己的 Types

`data Bool = False | True`        

data 表示我们要定义一个新的类型。= 的左端标明类型的名称即 Bool， = 的右端就是值构造子 (Value Constructor)，它们明确了该类型可能的值。| 读作 “或”，所以可以这样阅读该声明：Bool 类型的值可以是 True 或 False。 类型名和值构造子的首字母必大写。
 
### Record Syntax

```             
至少 data 声明的时候看上去有点变扭，没有属性名。
data Person = Person String String Int Float String String deriving (Show)
这样好多了
data Person = Person { firstName :: String , 
							lastName :: String , 
							age :: Int , 
							height :: Float , 
							phoneNumber :: String , 
							flavor :: String } deriving (Show)
```           

### Type parameters

泛型的概念

```                
data Car = Car { company :: String , model :: String , year :: Int } deriving (Show)

改成

data Car a b c = Car { company :: a , model :: b , year :: c } deriving (Show)

```             

不要在 data 声明中添加类型约束。
类型构造子 = 值构造子

```                                
data Vector a = Vector a a a deriving (Show)          
```               

### Derived instances              

类型类理解为接口，定义约束类的统一行为特征。                                           

```         
data Person = Person { firstName :: String , lastName :: String , age :: Int } deriving (Eq, Show, Read)
```                     

其中 deriving 就是在值构造子这边声明他行为继承与 Eq。          
Show 和 Read 是互对的行为，需要注意的是 Read 转化为特定类型的时候需要制定类型，或有明确的类型推断提供。不可以 `read "Just 't'" :: Maybe a` 这样的。 


### Type synonyms

别名 `type String = [Char]` 比较好理解。                            

对比一下,更加易读                        
 
```                               
type PhoneNumber = String 
type Name = String 
type PhoneBook = [(Name,PhoneNumber)]

inPhoneBook :: Name -> PhoneNumber -> PhoneBook -> Bool 
inPhoneBook name pnumber pbook = (name,pnumber) `elem` pbook

VS

inPhoneBook :: String -> String -> [(String ,String)] -> Bool

```                          

### Recursive data structures

递归定义数据结构,以 List 为例子         

```     
data List a = Empty | Cons a (List a) deriving (Show, Read, Eq, Ord)
data List a = Empty | Cons { listHead :: a, listTail :: List a} deriving (Show, Read, Eq, Ord)
con 就是指 : 
伪代码
data List a = Empty | a : (List a) deriving (Show, Read, Eq, Ord)
```                   

ﬁxity 指定了他应该是 leftassociative 或是 right-associative，还有他的优先级。             

```          
infixr 5 :-:
data List a = Empty | a :-: (List a) deriving (Show, Read, Eq, Ord)
```          

定义自己的 typeclass 相当于 interface。          

```                        
class Eq a where
	(==) :: a -> a -> Bool 
	(/=) :: a -> a -> Bool 
	x == y = not (x /= y) 
	x /= y = not (x == y)
```                                

class Eq a where 表示定义一个新的 typeclass 叫做 Eq，a 是一个泛型。                   

```                   
instance Eq TrafficLight where 
	Red == Red = True 
	Green == Green = True 
	Yellow == Yellow = True 
	_ == _ = False
```                               

instance 就是去实现特定类型的接口行为，他的行为可能直接的实现方式是 deriving(Eq)。        

### yes-no typeclass

主要是接口的定义和具体类的实现。模仿了 JavaScript 弱类型的语言对 Bool 值的转化。        

```         
class YesNo a where
	yeasno :: a -> Bool

instance YesNo Int where
	yesno 0 = False
	yesno _ = True

```          

### Functor typeclass

Functor 的定义 ，f 是一个__类型构造子__，他接受一个类型。比如 [a] 已经是一个具体的类型，其中 [] 是一个类型构造子。                          

```      
class Functor f where 
	fmap :: (a -> b) -> f a -> f b
```            

### Kind




## 输入与输出





 


 
 















 
 
 
 
 
 
 
 
 
 
 
 
 
