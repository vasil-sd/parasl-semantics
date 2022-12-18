---
title: Language Type
---

```k
module TYPE
  imports DOMAINS

  syntax TypeInt ::= "int"
  syntax Type ::= TypeInt

  syntax TypeBool ::= "bool"
  syntax Type ::= TypeBool

  // syntax TypeArray ::= Type "[" Int "]"
  // syntax Type ::= TypeArray

  syntax Type ::= "#top" | "#bot"
endmodule
```
