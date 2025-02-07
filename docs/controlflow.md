# 5. Control Flow

Welcome to Chapter 5 of the "Implementing a language with LLVM" tutorial. Parts 1-4 described the implementation of the simple Kaleidoscope language and included support for generating LLVM IR, followed by optimizations and a JIT compiler. Unfortunately, as presented, Kaleidoscope is mostly useless: it has no control flow other than call and return. This means that you can't have conditional branches in the code, significantly limiting its power. In this episode of "build that compiler", we'll extend Kaleidoscope to have an if/then/else expression plus a simple `for` loop.


## 5.1 If/Then/Else

Extending Kaleidoscope to support if/then/else is quite straightforward. It basically requires adding support for this "new" concept to the lexer, parser, AST, and LLVM code emitter. This example is nice, because it shows how easy it is to "grow" a language over time, incrementally extending it as new ideas are discovered.

Before we get going on "how" we add this extension, lets talk about "what" we want. The basic idea is that we want to be able to write this sort of thing:

```c++
def fib(x)
  if x < 3 then
    1
  else
    fib(x-1)+fib(x-2);
```

In Kaleidoscope, every construct is an expression: there are no statements. As such, the if/then/else expression needs to return a value like any other. Since we're using a mostly functional form, we'll have it evaluate its conditional, then return the `then` or `else` value based on how the condition was resolved. This is very similar to the C "?:" expression.

The semantics of the if/then/else expression is that it evaluates the condition to a boolean equality value: 0.0 is considered to be false and everything else is considered to be true. If the condition is true, the first subexpression is evaluated and returned, if the condition is false, the second subexpression is evaluated and returned. Since Kaleidoscope allows side-effects, this behavior is important to nail down.

Now that we know what we "want", lets break this down into its constituent pieces.


### 5.1.1 Lexer Extensions for If/Then/Else

The lexer extensions are straightforward. First we add new enum values for the relevant tokens:

```c++
// control
tok_if = -6, tok_then = -7, tok_else = -8,
Once we have that, we recognize the new keywords in the lexer. This is pretty simple stuff:

...
if (IdentifierStr == "def") return tok_def;
if (IdentifierStr == "extern") return tok_extern;
if (IdentifierStr == "if") return tok_if;
if (IdentifierStr == "then") return tok_then;
if (IdentifierStr == "else") return tok_else;
return tok_identifier;
```


### 5.1.2 AST Extensions for If/Then/Else

To represent the new expression we add a new AST node for it:

```c++
/// IfExprAST - Expression class for if/then/else.
class IfExprAST : public ExprAST {
  ExprAST *Cond, *Then, *Else;
public:
  IfExprAST(ExprAST *cond, ExprAST *then, ExprAST *_else)
    : Cond(cond), Then(then), Else(_else) {}
  virtual Value *Codegen();
};
```

The AST node just has pointers to the various subexpressions.


### 5.1.3 Parser Extensions for If/Then/Else

Now that we have the relevant tokens coming from the lexer and we have the AST node to build, our parsing logic is relatively straightforward. First we define a new parsing function:

```c++
/// ifexpr ::= 'if' expression 'then' expression 'else' expression
static ExprAST *ParseIfExpr() {
  getNextToken();  // eat the if.

  // condition.
  ExprAST *Cond = ParseExpression();
  if (!Cond) return 0;

  if (CurTok != tok_then)
    return Error("expected then");
  getNextToken();  // eat the then

  ExprAST *Then = ParseExpression();
  if (Then == 0) return 0;

  if (CurTok != tok_else)
    return Error("expected else");

  getNextToken();

  ExprAST *Else = ParseExpression();
  if (!Else) return 0;

  return new IfExprAST(Cond, Then, Else);
}
```

Next we hook it up as a primary expression:

```c++
static ExprAST *ParsePrimary() {
  switch (CurTok) {
  default: return Error("unknown token when expecting an expression");
  case tok_identifier: return ParseIdentifierExpr();
  case tok_number:     return ParseNumberExpr();
  case '(':            return ParseParenExpr();
  case tok_if:         return ParseIfExpr();
  }
}
```

### 5.1.4 LLVM IR for If/Then/Else

Now that we have it parsing and building the AST, the final piece is adding LLVM code generation support. This is the most interesting part of the if/then/else example, because this is where it starts to introduce new concepts. All of the code above has been thoroughly described in previous chapters.

To motivate the code we want to produce, lets take a look at a simple example. Consider:

```c++
extern foo();
extern bar();
def baz(x) if x then foo() else bar();
If you disable optimizations, the code you'll (soon) get from Kaleidoscope looks like this:

declare double @foo()

declare double @bar()

define double @baz(double %x) {
entry:
  %ifcond = fcmp one double %x, 0.000000e+00
  br i1 %ifcond, label %then, label %else

then:       ; preds = %entry
  %calltmp = call double @foo()
  br label %ifcont

else:       ; preds = %entry
  %calltmp1 = call double @bar()
  br label %ifcont

ifcont:     ; preds = %else, %then
  %iftmp = phi double [ %calltmp, %then ], [ %calltmp1, %else ]
  ret double %iftmp
}
```

To visualize the control flow graph, you can use a nifty feature of the LLVM 'opt' tool. If you put this LLVM IR into "t.ll" and run `llvm-as < t.ll | opt -analyze -view-cfg`, a window will pop up and you'll see this graph:

Example CFG

![cfg](img/cfg.png)

Example CFG

Another way to get this is to call `F->viewCFG()` or `F->viewCFGOnly()` (where F is a `Function*`) either by inserting actual calls into the code and recompiling or by calling these in the debugger. LLVM has many nice features for visualizing various graphs.

Getting back to the generated code, it is fairly simple: the entry block evaluates the conditional expression (`x` in our case here) and compares the result to 0.0 with the `fcmp one` instruction ('one' is "Ordered and Not Equal"). Based on the result of this expression, the code jumps to either the `then` or `else` blocks, which contain the expressions for the true/false cases.

Once the then/else blocks are finished executing, they both branch back to the `ifcont` block to execute the code that happens after the if/then/else. In this case the only thing left to do is to return to the caller of the function. The question then becomes: how does the code know which expression to return?

The answer to this question involves an important SSA operation: the Phi operation. If you're not familiar with SSA, the wikipedia article is a good introduction and there are various other introductions to it available on your favorite search engine. The short version is that "execution" of the Phi operation requires "remembering" which block control came from. The Phi operation takes on the value corresponding to the input control block. In this case, if control comes in from the `then` block, it gets the value of `calltmp`. If control comes from the `else` block, it gets the value of `calltmp1`.

At this point, you are probably starting to think "Oh no! This means my simple and elegant front-end will have to start generating SSA form in order to use LLVM!". Fortunately, this is not the case, and we strongly advise not implementing an SSA construction algorithm in your front-end unless there is an amazingly good reason to do so. In practice, there are two sorts of values that float around in code written for your average imperative programming language that might need Phi nodes:

Code that involves user variables:
  * x = 1; x = x + 1;
  * Values that are implicit in the structure of your AST, such as the Phi node in this case.

In Chapter 7 of this tutorial ("mutable variables"), we'll talk about #1 in depth. For now, just believe me that you don't need SSA construction to handle this case. For #2, you have the choice of using the techniques that we will describe for #1, or you can insert Phi nodes directly, if convenient. In this case, it is really really easy to generate the Phi node, so we choose to do it directly.

Okay, enough of the motivation and overview, lets generate code!


### 5.1.5 Code Generation for If/Then/Else

In order to generate code for this, we implement the Codegen method for IfExprAST:

```c++
Value *IfExprAST::Codegen() {
  Value *CondV = Cond->Codegen();
  if (CondV == 0) return 0;

  // Convert condition to a bool by comparing equal to 0.0.
  CondV = Builder.CreateFCmpONE(CondV,
                              ConstantFP::get(getGlobalContext(), APFloat(0.0)),
                                "ifcond");
This code is straightforward and similar to what we saw before. We emit the expression for the condition, then compare that value to zero to get a truth value as a 1-bit (bool) value.

Function *TheFunction = Builder.GetInsertBlock()->getParent();

// Create blocks for the then and else cases.  Insert the 'then' block at the
// end of the function.
BasicBlock *ThenBB = BasicBlock::Create(getGlobalContext(), "then", TheFunction);
BasicBlock *ElseBB = BasicBlock::Create(getGlobalContext(), "else");
BasicBlock *MergeBB = BasicBlock::Create(getGlobalContext(), "ifcont");

Builder.CreateCondBr(CondV, ThenBB, ElseBB);
```

This code creates the basic blocks that are related to the if/then/else statement, and correspond directly to the blocks in the example above. The first line gets the current Function object that is being built. It gets this by asking the builder for the current BasicBlock, and asking that block for its "parent" (the function it is currently embedded into).

Once it has that, it creates three blocks. Note that it passes "TheFunction" into the constructor for the `then` block. This causes the constructor to automatically insert the new block into the end of the specified function. The other two blocks are created, but aren't yet inserted into the function.

Once the blocks are created, we can emit the conditional branch that chooses between them. Note that creating new blocks does not implicitly affect the IRBuilder, so it is still inserting into the block that the condition went into. Also note that it is creating a branch to the `then` block and the `else` block, even though the `else` block isn't inserted into the function yet. This is all ok: it is the standard way that LLVM supports forward references.

```c++
// Emit then value.
Builder.SetInsertPoint(ThenBB);

Value *ThenV = Then->Codegen();
if (ThenV == 0) return 0;

Builder.CreateBr(MergeBB);
// Codegen of 'Then' can change the current block, update ThenBB for the PHI.
ThenBB = Builder.GetInsertBlock();
```

After the conditional branch is inserted, we move the builder to start inserting into the `then` block. Strictly speaking, this call moves the insertion point to be at the end of the specified block. However, since the `then` block is empty, it also starts out by inserting at the beginning of the block. :)

Once the insertion point is set, we recursively codegen the `then` expression from the AST. To finish off the `then` block, we create an unconditional branch to the merge block. One interesting (and very important) aspect of the LLVM IR is that it requires all basic blocks to be "terminated" with a control flow instruction such as return or branch. This means that all control flow, including fall throughs must be made explicit in the LLVM IR. If you violate this rule, the verifier will emit an error.

The final line here is quite subtle, but is very important. The basic issue is that when we create the Phi node in the merge block, we need to set up the block/value pairs that indicate how the Phi will work. Importantly, the Phi node expects to have an entry for each predecessor of the block in the CFG. Why then, are we getting the current block when we just set it to ThenBB 5 lines above? The problem is that the `then` expression may actually itself change the block that the Builder is emitting into if, for example, it contains a nested "if/then/else" expression. Because calling Codegen recursively could arbitrarily change the notion of the current block, we are required to get an up-to-date value for code that will set up the Phi node.

```c++
// Emit else block.
TheFunction->getBasicBlockList().push_back(ElseBB);
Builder.SetInsertPoint(ElseBB);

Value *ElseV = Else->Codegen();
if (ElseV == 0) return 0;

Builder.CreateBr(MergeBB);
// Codegen of 'Else' can change the current block, update ElseBB for the PHI.
ElseBB = Builder.GetInsertBlock();
Code generation for the 'else' block is basically identical to codegen for the 'then' block. The only significant difference is the first line, which adds the 'else' block to the function. Recall previously that the 'else' block was created, but not added to the function. Now that the 'then' and 'else' blocks are emitted, we can finish up with the merge code:

  // Emit merge block.
  TheFunction->getBasicBlockList().push_back(MergeBB);
  Builder.SetInsertPoint(MergeBB);
  PHINode *PN = Builder.CreatePHI(Type::getDoubleTy(getGlobalContext()), 2,
                                  "iftmp");

  PN->addIncoming(ThenV, ThenBB);
  PN->addIncoming(ElseV, ElseBB);
  return PN;
}
```

The first two lines here are now familiar: the first adds the "merge" block to the Function object (it was previously floating, like the else block above). The second block changes the insertion point so that newly created code will go into the "merge" block. Once that is done, we need to create the PHI node and set up the block/value pairs for the PHI.

Finally, the CodeGen function returns the phi node as the value computed by the if/then/else expression. In our example above, this returned value will feed into the code for the top-level function, which will create the return instruction.

Overall, we now have the ability to execute conditional code in Kaleidoscope. With this extension, Kaleidoscope is a fairly complete language that can calculate a wide variety of numeric functions. Next up we'll add another useful expression that is familiar from non-functional languages...


### 5.2 `for` Loop Expression

Now that we know how to add basic control flow constructs to the language, we have the tools to add more powerful things. Lets add something more aggressive, a `for` expression:

```c++
extern putchard(char)
def printstar(n)
  for i = 1, i < n, 1.0 in
    putchard(42);  # ascii 42 = '*'

# print 100 '*' characters
printstar(100);
```

This expression defines a new variable (`i` in this case) which iterates from a starting value, while the condition (`i < n` in this case) is true, incrementing by an optional step value (`1.0` in this case). If the step value is omitted, it defaults to `1.0`. While the loop is true, it executes its body expression. Because we don't have anything better to return, we'll just define the loop as always returning `0.0`. In the future when we have mutable variables, it will get more useful.

As before, lets talk about the changes that we need to Kaleidoscope to support this.


### 5.2.1 Lexer Extensions for the `for` Loop

The lexer extensions are the same sort of thing as for if/then/else:

```c++
... in enum Token ...
// control
tok_if = -6, tok_then = -7, tok_else = -8,
tok_for = -9, tok_in = -10

... in gettok ...
if (IdentifierStr == "def") return tok_def;
if (IdentifierStr == "extern") return tok_extern;
if (IdentifierStr == "if") return tok_if;
if (IdentifierStr == "then") return tok_then;
if (IdentifierStr == "else") return tok_else;
if (IdentifierStr == "for") return tok_for;
if (IdentifierStr == "in") return tok_in;
return tok_identifier;
```


### 5.2.2 AST Extensions for the `for` Loop

The AST node is just as simple. It basically boils down to capturing the variable name and the constituent expressions in the node.

```c++
/// ForExprAST - Expression class for for/in.
class ForExprAST : public ExprAST {
  std::string VarName;
  ExprAST *Start, *End, *Step, *Body;
public:
  ForExprAST(const std::string &varname, ExprAST *start, ExprAST *end,
             ExprAST *step, ExprAST *body)
    : VarName(varname), Start(start), End(end), Step(step), Body(body) {}
  virtual Value *Codegen();
};
```

### 5.2.3 Parser Extensions for the `for` Loop

The parser code is also fairly standard. The only interesting thing here is handling of the optional step value. The parser code handles it by checking to see if the second comma is present. If not, it sets the step value to null in the AST node:

```c++
/// forexpr ::= 'for' identifier '=' expr ',' expr (',' expr)? 'in' expression
static ExprAST *ParseForExpr() {
  getNextToken();  // eat the for.

  if (CurTok != tok_identifier)
    return Error("expected identifier after for");

  std::string IdName = IdentifierStr;
  getNextToken();  // eat identifier.

  if (CurTok != '=')
    return Error("expected '=' after for");
  getNextToken();  // eat '='.


  ExprAST *Start = ParseExpression();
  if (Start == 0) return 0;
  if (CurTok != ',')
    return Error("expected ',' after for start value");
  getNextToken();

  ExprAST *End = ParseExpression();
  if (End == 0) return 0;

  // The step value is optional.
  ExprAST *Step = 0;
  if (CurTok == ',') {
    getNextToken();
    Step = ParseExpression();
    if (Step == 0) return 0;
  }

  if (CurTok != tok_in)
    return Error("expected 'in' after for");
  getNextToken();  // eat 'in'.

  ExprAST *Body = ParseExpression();
  if (Body == 0) return 0;

  return new ForExprAST(IdName, Start, End, Step, Body);
}
```


### 5.2.4 LLVM IR for the `for` Loop

Now we get to the good part: the LLVM IR we want to generate for this thing. With the simple example above, we get this LLVM IR (note that this dump is generated with optimizations disabled for clarity):

```c++
declare double @putchard(double)

define double @printstar(double %n) {
entry:
  ; initial value = 1.0 (inlined into phi)
  br label %loop

loop:       ; preds = %loop, %entry
  %i = phi double [ 1.000000e+00, %entry ], [ %nextvar, %loop ]
  ; body
  %calltmp = call double @putchard(double 4.200000e+01)
  ; increment
  %nextvar = fadd double %i, 1.000000e+00

  ; termination test
  %cmptmp = fcmp ult double %i, %n
  %booltmp = uitofp i1 %cmptmp to double
  %loopcond = fcmp one double %booltmp, 0.000000e+00
  br i1 %loopcond, label %loop, label %afterloop

afterloop:      ; preds = %loop
  ; loop always returns 0.0
  ret double 0.000000e+00
}
```
This loop contains all the same constructs we saw before: a phi node, several expressions, and some basic blocks. Lets see how this fits together.


### 5.2.5 Code Generation for the `for` Loop

The first part of Codegen is very simple: we just output the start expression for the loop value:

```c++
Value *ForExprAST::Codegen() {
  // Emit the start code first, without 'variable' in scope.
  Value *StartVal = Start->Codegen();
  if (StartVal == 0) return 0;
```

With this out of the way, the next step is to set up the LLVM basic block for the start of the loop body. In the case above, the whole loop body is one block, but remember that the body code itself could consist of multiple blocks (e.g. if it contains an if/then/else or a for/in expression).

```c++
// Make the new basic block for the loop header, inserting after current
// block.
Function *TheFunction = Builder.GetInsertBlock()->getParent();
BasicBlock *PreheaderBB = Builder.GetInsertBlock();
BasicBlock *LoopBB = BasicBlock::Create(getGlobalContext(), "loop", TheFunction);

// Insert an explicit fall through from the current block to the LoopBB.
Builder.CreateBr(LoopBB);
```

This code is similar to what we saw for if/then/else. Because we will need it to create the Phi node, we remember the block that falls through into the loop. Once we have that, we create the actual block that starts the loop and create an unconditional branch for the fall-through between the two blocks.

```c++
// Start insertion in LoopBB.
Builder.SetInsertPoint(LoopBB);

// Start the PHI node with an entry for Start.
PHINode *Variable = Builder.CreatePHI(Type::getDoubleTy(getGlobalContext()), 2, VarName.c_str());
Variable->addIncoming(StartVal, PreheaderBB);
```

Now that the "preheader" for the loop is set up, we switch to emitting code for the loop body. To begin with, we move the insertion point and create the PHI node for the loop induction variable. Since we already know the incoming value for the starting value, we add it to the Phi node. Note that the Phi will eventually get a second value for the backedge, but we can't set it up yet (because it doesn't exist!).

```c++
// Within the loop, the variable is defined equal to the PHI node.  If it
// shadows an existing variable, we have to restore it, so save it now.
Value *OldVal = NamedValues[VarName];
NamedValues[VarName] = Variable;

// Emit the body of the loop.  This, like any other expr, can change the
// current BB.  Note that we ignore the value computed by the body, but don't
// allow an error.
if (Body->Codegen() == 0)
  return 0;
```

Now the code starts to get more interesting. Our 'for' loop introduces a new variable to the symbol table. This means that our symbol table can now contain either function arguments or loop variables. To handle this, before we codegen the body of the loop, we add the loop variable as the current value for its name. Note that it is possible that there is a variable of the same name in the outer scope. It would be easy to make this an error (emit an error and return null if there is already an entry for VarName) but we choose to allow shadowing of variables. In order to handle this correctly, we remember the Value that we are potentially shadowing in OldVal (which will be null if there is no shadowed variable).

Once the loop variable is set into the symbol table, the code recursively codegen's the body. This allows the body to use the loop variable: any references to it will naturally find it in the symbol table.

```c++
// Emit the step value.
Value *StepVal;
if (Step) {
  StepVal = Step->Codegen();
  if (StepVal == 0) return 0;
} else {
  // If not specified, use 1.0.
  StepVal = ConstantFP::get(getGlobalContext(), APFloat(1.0));
}

Value *NextVar = Builder.CreateFAdd(Variable, StepVal, "nextvar");
```

Now that the body is emitted, we compute the next value of the iteration variable by adding the step value, or 1.0 if it isn't present. `NextVar` will be the value of the loop variable on the next iteration of the loop.

```c++
// Compute the end condition.
Value *EndCond = End->Codegen();
if (EndCond == 0) return EndCond;

// Convert condition to a bool by comparing equal to 0.0.
EndCond = Builder.CreateFCmpONE(EndCond,
                            ConstantFP::get(getGlobalContext(), APFloat(0.0)),
                                "loopcond");
```

Finally, we evaluate the exit value of the loop, to determine whether the loop should exit. This mirrors the condition evaluation for the if/then/else statement.

```c++
// Create the "after loop" block and insert it.
BasicBlock *LoopEndBB = Builder.GetInsertBlock();
BasicBlock *AfterBB = BasicBlock::Create(getGlobalContext(), "afterloop", TheFunction);

// Insert the conditional branch into the end of LoopEndBB.
Builder.CreateCondBr(EndCond, LoopBB, AfterBB);

// Any new code will be inserted in AfterBB.
Builder.SetInsertPoint(AfterBB);
```

With the code for the body of the loop complete, we just need to finish up the control flow for it. This code remembers the end block (for the phi node), then creates the block for the loop exit (`afterloop`). Based on the value of the exit condition, it creates a conditional branch that chooses between executing the loop again and exiting the loop. Any future code is emitted in the `afterloop` block, so it sets the insertion position to it.

```c++
  // Add a new entry to the PHI node for the backedge.
  Variable->addIncoming(NextVar, LoopEndBB);

  // Restore the unshadowed variable.
  if (OldVal)
    NamedValues[VarName] = OldVal;
  else
    NamedValues.erase(VarName);

  // for expr always returns 0.0.
  return Constant::getNullValue(Type::getDoubleTy(getGlobalContext()));
}
```

The final code handles various cleanups: now that we have the `NextVar` value, we can add the incoming value to the loop PHI node. After that, we remove the loop variable from the symbol table, so that it isn't in scope after the for loop. Finally, code generation of the for loop always returns 0.0, so that is what we return from `ForExprAST::Codegen`.

With this, we conclude the "adding control flow to Kaleidoscope" chapter of the tutorial. In this chapter we added two control flow constructs, and used them to motivate a couple of aspects of the LLVM IR that are important for front-end implementors to know. In the next chapter of our saga, we will get a bit crazier and add user-defined operators to our poor innocent language.
