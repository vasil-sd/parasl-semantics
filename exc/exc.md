---
title: Language Syntax
---

```k
requires "type.md"

module INNER-SYNTAX
  imports DOMAINS

  syntax Value ::= Int
                | Bool

  syntax InnerExpr ::= Value
                    > "(" InnerExpr ")"           [bracket]
                    > "!" InnerExpr
                    > left:
                      InnerExpr "/" InnerExpr     [left]
                    | InnerExpr "%" InnerExpr     [left]
                    | InnerExpr "*" InnerExpr     [left]
                    > left:
                      InnerExpr "-" InnerExpr     [left]
                    | InnerExpr "+" InnerExpr     [left]
                    > non-assoc:
                      InnerExpr "==" InnerExpr    [non-assoc]
                    | InnerExpr "!=" InnerExpr    [non-assoc]
                    | InnerExpr "<" InnerExpr     [non-assoc]
                    | InnerExpr ">=" InnerExpr    [non-assoc]
                    > left:
                      InnerExpr "&&" InnerExpr    [left]
                    > InnerExpr "||" InnerExpr    [left]

  syntax InnerExpr ::= simplify(InnerExpr)        [function, total]
endmodule

module EXC-SYNTAX
  imports TYPE
  imports INNER-SYNTAX
  imports ID-SYNTAX

  syntax Result ::= "(" Type "," InnerExpr ")"

  syntax KResult ::= Result

  syntax Var ::= Id

  syntax Expr ::= Result
                | Int
                | Bool
                | Var
                > "(" Expr ")"            [bracket]
                > "!" Expr                [strict]
                > left:
                  Expr "/" Expr           [strict, left]
                | Expr "%" Expr           [strict, left]
                | Expr "*" Expr           [strict, left]
                > left:
                  Expr "-" Expr           [strict, left]
                | Expr "+" Expr           [strict, left]
                > non-assoc:
                  Expr "==" Expr          [strict, non-assoc]
                | Expr "!=" Expr          [strict, non-assoc]
                | Expr "<" Expr           [strict, non-assoc]
                | Expr ">=" Expr          [strict, non-assoc]
                | Expr ">" Expr           [strict, non-assoc]
                | Expr "<=" Expr          [strict, non-assoc]
                > left: Expr "&&" Expr    [strict(1), left]
                > left: Expr "||" Expr    [strict(1), left]

  syntax Block ::= "{" "}"
                | "{" Stmt "}"

  syntax Stmt ::= Type Var "=" Expr ";"                 [strict(3)]
                | Var "=" Expr ";"                      [strict(2)]
                | "while" "(" Expr ")" Block            [strict(1)]
                | Block
                | "if" "(" Expr ")" Block               [macro]
                > "if" "(" Expr ")" Block "else" Block  [avoid, strict(1)]

  syntax Stmt ::= Stmt Stmt                             [right]

  // macro expantion rules
  rule if ( B ) S => if ( B ) S else { }
endmodule

module CONFIG
  imports TYPE
  imports EXC-SYNTAX
  imports MAP

  configuration
  <T>
    <k> $PGM:Stmt </k>
    <vars> .Map </vars>
  </T>
endmodule

module SYNTAX-ERROR
  imports EXC-SYNTAX
  syntax Error ::= AssignmentTypingError(KItem, Type, Type)
                 | AlreadyDeclared(KItem, Var)
                 | ExprBinTypingError(KItem, Type, Type, Type)
                 | ExprEqTypingError(KItem, Type, Type)
                 | ExprUnTypingError(KItem, Type, Type)
                 | IfTypingError(KItem, Type, Type)
                 | AssertIsAlwaysFalse(KItem)
endmodule

module SIMPLIFY
  imports TYPE
  imports INNER-SYNTAX
  imports SYNTAX-ERROR
  imports INT
  imports BOOL

  rule simplify(A:Int + B:Int) => A +Int B
  rule simplify(A:Int + 0) => A
  rule simplify(0 + B:Int) => B
  rule simplify(A:Int - B:Int) => A -Int B
  rule simplify(A:Int - 0) => A

  rule simplify(A:Int * B:Int) => A *Int B
  rule simplify(_:Int * 0) => 0
  rule simplify(0 * _:Int) => 0
  rule simplify(A:Int / B:Int) => A /Int B
    requires B =/=Int 0
  rule simplify(0 / B:Int) => 0
    requires B =/=Int 0
  rule simplify(A:Int % B:Int) => A %Int B
    requires B =/=Int 0
  rule simplify(0 % B:Int) => 0
    requires B =/=Int 0

  rule simplify(A:Int == B:Int) => A ==Int B
  rule simplify(A:Bool == B:Bool) => A ==Bool B
  rule simplify(A:Int != B:Int) => A =/=Int B
  rule simplify(A:Bool != B:Bool) => A =/=Bool B

  rule simplify(A:Int < B:Int) => A <Int B
  rule simplify(A:Int >= B:Int) => A >=Int B

  rule simplify(A:Bool && B:Bool) => A andBool B
  rule simplify(false && _:Bool) => false
  rule simplify(true && B:Bool) => B
  rule simplify(A:Bool || B:Bool) => A orBool B
  rule simplify(false || B:Bool) => B
  rule simplify(true || _:Bool) => true
  rule simplify(! A:Bool) => notBool A
  rule simplify(! ! A:Bool) => A

  rule simplify(A) => A
    [owise]
endmodule

module EXC
  imports EXC-SYNTAX
  imports CONFIG
  imports SIMPLIFY
  imports SYNTAX-ERROR
  imports ID
  imports BOOL
  imports INT
  imports K-EQUAL

  // statements semantics
  rule { } => .                                       [structural]
  rule { S } => S                                     [structural]

  rule S1:Stmt S2:Stmt => S1 ~> S2                    [structural]

  rule while ( E ) S => if ( E ) { S while ( E ) S }  [structural]

  // Id declaration semantics
  rule <k> T:Type V:Var = (T, _) #as Val ; => . ...</k>
       <vars> Vs => Vs [V <- Val] </vars>
    requires notBool(V in_keys(Vs))

  rule <k> _:Type V:Var = _ ; #as S => AlreadyDeclared(S, V)  ...</k>
       <vars> ... V |-> _ ... </vars>
    [owise]

  // assignment semantics
  rule <k> V:Var = (T, Val) ; => . ...</k>
       <vars>... V |-> (T, (_ => Val)) </vars>

  rule <k> V:Var = (T1, _) ; #as S => AssignmentTypingError(S, T1, T2) ...</k>
       <vars>... V |-> (T2, _) </vars>
    [owise]

  rule <k> V:Var => Val ... </k>
       <vars>... V |-> Val ... </vars>

  // type conversions
  rule V:Int => (int, V)
  rule V:Bool => (bool, V)

  // inner arithmetic expressions
  rule (int, V1) + (int, V2) => (int, simplify(V1 + V2))
  rule (int, V1) - (int, V2) => (int, simplify(V1 - V2))
  rule (int, V1) * (int, V2) => (int, simplify(V1 * V2))
  rule (int, V1) / (int, V2) => (int, simplify(V1 / V2))
  rule (int, V1) % (int, V2) => (int, simplify(V1 % V2))
  rule (T:Type, V1) == (T, V2) => (bool, simplify(V1 == V2))
  rule (T:Type, V1) != (T, V2) => (bool, simplify(V1 != V2))
  rule (int, V1) < (int, V2) => (bool, simplify(V1 < V2))
  rule (int, V1) >= (int, V2) => (bool, simplify(V1 >= V2))
  rule (int, V1) > (int, V2) => (bool, simplify(V2 < V1))
  rule (int, V1) <= (int, V2) => (bool, simplify(V2 >= V1))
  rule (bool, V1) && (bool, V2) => (bool, simplify(V1 && V2))
  rule (bool, V1) || (bool, V2) => (bool, simplify(V1 || V2))
  rule ! (bool, V1) => (bool, simplify(! V1))

  // handling typing errors
  rule (T1, _) + (T2, _)  #as E => ExprBinTypingError(E, int, T1, T2)  [owise]
  rule (T1, _) - (T2, _)  #as E => ExprBinTypingError(E, int, T1, T2)  [owise]
  rule (T1, _) % (T2, _)  #as E => ExprBinTypingError(E, int, T1, T2)  [owise]
  rule (T1, _) / (T2, _)  #as E => ExprBinTypingError(E, int, T1, T2)  [owise]
  rule (T1, _) * (T2, _)  #as E => ExprBinTypingError(E, int, T1, T2)  [owise]
  rule (T1, _) == (T2, _) #as E => ExprEqTypingError(E, T1, T2)        [owise]
  rule (T1, _) != (T2, _) #as E => ExprEqTypingError(E, T1, T2)        [owise]
  rule (T1, _) < (T2, _)  #as E => ExprBinTypingError(E, int, T1, T2)  [owise]
  rule (T1, _) >= (T2, _) #as E => ExprBinTypingError(E, int, T1, T2)  [owise]
  rule (T1, _) > (T2, _)  #as E => ExprBinTypingError(E, int, T1, T2)  [owise]
  rule (T1, _) <= (T2, _) #as E => ExprBinTypingError(E, int, T1, T2)  [owise]
  rule (T1, _) && (T2, _) #as E => ExprBinTypingError(E, bool, T1, T2) [owise]
  rule (T1, _) || (T2, _) #as E => ExprBinTypingError(E, bool, T1, T2) [owise]
  rule ! (T, _)           #as E => ExprUnTypingError(E, bool, T)       [owise]

  // if statement simplifications
  rule if ( (bool, true) ) S else _  => S
  rule if ( (bool, false) ) _ else S => S

  rule if ( (T, _) ) _ else _ #as S => IfTypingError(S, bool, T)
    [owise]

  // while statement simplifications
  rule while ( (bool, false) ) _ => .

  rule while ( (int, _) ) _ #as S => IfTypingError(S, bool, int)
    [owise]
endmodule
```
