---
title: Values
---

```k
requires "type.md"

module BIT
  import DOMAINS

  syntax Int ::= #bitsInInt(Int) [function, functional]

  rule #bitsInInt(N) => 8
    requires N >=Int -128 andBool N <=Int 127

  // TODO: add other cases, look at domains.md and try to optimize
  // it using absInt and log2Int standard functions
endmodule

module VALUE-SYNTAX
  import TYPE-SYNTAX
  import INNER-TYPE-SYNTAX
  import DOMAINS

  syntax OuterValueInt ::= Int
  syntax OuterValue ::= OuterValueInt

  syntax OuterInitializersList ::= List{OuterValue, ","}
                                 | "{" OuterInitializersList "}" [bracket]

  syntax InnerValueInt ::= Int
  syntax InnerValue ::= InnerValueInt

  syntax InnerTypedValue ::= value(InnerType, InnerValue)

  syntax KResult ::= InnerTypedValue

  syntax InnerTypedValue ::= #toInnerValue(OuterValue) [function]
  // TODO: add other cases

  syntax Bool ::= #validInnerTypedValue(InnerTypedValue) [function, functional]

endmodule

module VALUE
  import BIT
  imports VALUE-SYNTAX
  imports TYPE

  rule #toInnerValue(V:OuterValue) => value(int(#bitsInInt({V}:>Int)), {V}:>Int)
    requires isOuterValueInt(V)

  rule #validInnerTypedValue(value(_Type:InnerTypeInt, _Value:InnerValueInt)) => true
  rule #validInnerTypedValue(_) => false [owise]

  configuration <k> #toInnerValue($PGM:OuterValue) </k>

endmodule
```