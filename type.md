---
title: Система типов, синтаксис внутренний и внешний
---

# Введение

Зачем нужны несколько синтаксисов?

1. Внешний - удобен пользователю: много синтаксического сахара, удобных конструкций и тд.
   Внутренний - удобен разработчику языка, компилятора, интерпретатора и тд:
   однородный, простой и пр.

2. Можно сделать несколько внутренних синтаксисов, которые приближены (вплоть до 1-к-1)
   к AST и IR и для них операционные семантики, что позволит иметь формальную модель представления
   программы на разных стадиях компиляции и у нас будет возможность хорошо
   протестировать разные слои/подсистемы компилятора/интерпретатора и тд.
   Особенно, когда есть чётко определённая трансляционная семантика: как одно
   представление отображается на другое.

# Общий декларативный модуль для типов

Тут просто декларируем сорты, которые будем потом использовать.

И функцию трансляции из внешнего синтаксиса во внутренний

```k
module TYPE-SYNTAX
  syntax InnerType
  syntax OuterType
  syntax InnerType ::= #toInnerType(OuterType) [function, functional]
endmodule
```

# Внутренний синтаксис для типов данных

```k
module INNER-TYPE-SYNTAX
  import TYPE-SYNTAX
  import ID-SYNTAX
  import LIST
```

## Целочисленные

```k
  syntax InnerTypeInt ::= int(intBits:Int)
```

`intBits` - это именованный параметр количества бит.


```k
  syntax InnerType ::= InnerTypeInt
```

## С плавающей точкой

```k
  syntax InnerTypeFloat ::= float(expBits:Int, mantBits:Int)
```

```k
  syntax InnerType ::= InnerTypeFloat
```

## Массивы

```k
  syntax InnerTypeArray ::= array(arrayElementType:InnerType, arraySize:Int)
```

```k
  syntax InnerType ::= InnerTypeArray
```

## Вектора

```k
  syntax InnerTypeVector ::= vector(vectorElementType:InnerType, vectorSize:Int)
```

```k
  syntax InnerType ::= InnerTypeVector
```

## Сигнатуры функций

```k
  syntax InnerParam ::= param(paramName:Id, paramType:InnerType)
  syntax InnerParams ::= List{InnerParam,","}
  syntax InnerFuncType ::= func(funcParams:InnerParams, returnType:InnerType)
  syntax InnerType ::= InnerFuncType
```

## Структуры

```k
  syntax InnerFieldName ::= "#undef" > Id
  syntax InnerStructField ::= field(fieldName:InnerFieldName, fieldType:InnerType)
  syntax InnerStructFields ::= List{InnerStructField,","}
  syntax InnerStructType ::= struct(structFields:InnerStructFields)
  syntax InnerType ::= InnerStructType
```

## Ссылки

Ссылочные типы явно или неявно используются там, где нужно передавать сложные
типы, например, массивы. Мало в каких языках используется передача по значению
сложных типов.

```k
  syntax InnerRefType ::= ref(refType:InnerType)
```

`refType` - это тип значения, который будет храниться в соотв. ячейке.

Переменные типа `ref` будут хранить, как значения, только номера ячеек.

По-сути, когда мы объявляем переменную типа структуры, мы объявляем переменную
типа ссылки на структуру:

```
a: {x : int, y: int};
a: ref(struct(field(x, int), field(y, int)));
```

## Дополним нашу систему типов

```k
  syntax InnerType ::= "#top"
                     | "#bot"
```

`#top` - это супертип всех типов, все типы являются его подтипами.

`#bot` - это подтип всех типов, все типы для него супертипы.

Эти типы будут полезны для расширения/сужения типов переменных, параметров и пр.

```k
endmodule
```

# Внешний синтаксис

Это то, что пользователь (программист) будет использовать для написания программ.

```k
module OUTER-TYPE-SYNTAX
  import TYPE-SYNTAX
  import DOMAINS
```

## Целочисленный тип

```k
  syntax OuterTypeInt ::= int(intBits:Int)
                        | "int" [macro]
                        | "char" [macro]
  rule int => int(32)
  rule char => int(8)

  syntax OuterType ::= OuterTypeInt
```

## Плавающая точка

```k
  syntax OuterTypeFloat ::= "double"
                          | "float"

  syntax OuterType ::= OuterTypeFloat
```

## Массивы

```k
  syntax OuterTypeArray ::= OuterType "[" Int "]"
  syntax OuterType ::= OuterTypeArray
```

## Вектора

```k
  syntax OuterTypeVector ::= "vector" "<" OuterType "," Int ">"
  syntax OuterType ::= OuterTypeVector
```

## Структуры

```k
  syntax OuterTypeStructField ::= Id ":" OuterType
                                | Id [macro, klabel(outerTypeStructFieldMacro)]

  rule (Name:Id):OuterTypeStructField => (Name : #top):OuterTypeStructField

  syntax OuterTypeStructFields ::= List{OuterTypeStructField, ","}
  syntax OuterTypeStruct ::= "{" OuterTypeStructFields "}"
  syntax OuterType ::= OuterTypeStruct
```

## Функции

```k
  syntax OuterTypeFuncParam ::= Id ":" OuterType
                              > Id [macro,klabel(outerTypeFuncParamMacro)]

  rule (Name:Id):OuterTypeFuncParam => (Name : #top):OuterTypeFuncParam

  syntax OuterTypeFuncParams ::= List{OuterTypeFuncParam, ","}
  syntax OuterTypeFunc ::= "(" OuterTypeFuncParams ")" ":" OuterType
                         > "(" OuterTypeFuncParams ")" [macro]

  rule ( (P:OuterTypeFuncParams) ):OuterTypeFunc =>
    ((P) : #bot):OuterTypeFunc

  syntax OuterType ::= OuterTypeFunc
```

## Дополнительные типы

```k
  syntax OuterType ::= "#top" | "#bot"
endmodule
```

# Основной модуль

```k
module TYPE
  import TYPE-SYNTAX
  import INNER-TYPE-SYNTAX
  import OUTER-TYPE-SYNTAX

  rule #toInnerType(int(N)) => int(N)
  rule #toInnerType(double) => float(11, 52)
  rule #toInnerType(float)  => float(8, 23)

  rule #toInnerType(#top) => #top
  rule #toInnerType(#bot) => #bot

  rule #toInnerType(OT:OuterType [N]) => array(#toInnerType(OT), N)
  rule #toInnerType(vector<OT, N>) => vector(#toInnerType(OT), N)

  syntax InnerStructField ::= #toInnerStructField(OuterTypeStructField) [function, functional]
  rule #toInnerStructField(N : T) => field(N, #toInnerType(T))

  syntax InnerStructFields ::= #toInnerStructFields(OuterTypeStructFields) [function, functional]

  rule #toInnerStructFields(OF:OuterTypeStructField, Rest:OuterTypeStructFields) =>
       #toInnerStructField(OF), #toInnerStructFields(Rest)
  rule #toInnerStructFields(.OuterTypeStructFields) => .InnerStructFields

  rule #toInnerType({Fields}) => struct(#toInnerStructFields(Fields))

  syntax InnerParam ::= #toInnerParam(OuterTypeFuncParam) [function, functional]
  rule #toInnerParam(N : T) => param(N, #toInnerType(T))

  syntax InnerParams ::= #toInnerParams(OuterTypeFuncParams) [function, functional]
  rule #toInnerParams(P, Rest) => #toInnerParam(P), #toInnerParams(Rest)
  rule #toInnerParams(.OuterTypeFuncParams) => .InnerParams

  rule #toInnerType( ( Params:OuterTypeFuncParams ) : (RT:OuterType) ) =>
       func(#toInnerParams(Params), #toInnerType(RT))

endmodule
```

# Немного тестов

```{.k .test}
module TYPE-TEST
  import TYPE

  configuration <k> #toInnerType($PGM:OuterType) </k>
endmodule
```

```{.sh}
#!/bin/bash

kompile --md-selector 'k' --syntax-module OUTER-TYPE-SYNTAX --main-module TYPE-TEST type.md
cat > test1.type <<END
(x,y)
END
krun test1.type

cat > test2.type <<END
{x:int, y:int}
END
krun test2.type

cat > test3.type <<END
{x:int, y:(a,b)}
END
krun test3.type

```
