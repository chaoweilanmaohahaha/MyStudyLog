# antlr4

总结一下antlr4这个工具呗，它是一款比较有趣的语法和词法分析工具，可以通过自己编写的语法，然后对输入的语句进行语法分析，同时能够生成对应的抽象语法树。

## 用法

需要提前安装好java的环境，1.6版本朝上；然后去它的官网去下载jar包(比如antlr-4.7.1-complete.jar)，接下来就是配置一下使用的环境，需要注意的是需要把jar包存在的位置加入到CLASSPATH环境变量中，当我们完成以上的这几步之后，可以测试jar包是否能使用：

```
$ java -jar /usr/local/lib/antlr-4.7.1-complete.jar
```

假设我们定义使用这个工具的命令简称为antlr(完整的就是使用上面这个语句)，那么完整的使用过程如下：

```
$ cd /tmp
$ antlr Hello.g4 // Hello.g4文件是自行编写的语法文件，下面再介绍
$ javac Hello*.java // 上面的语句会生成许多的.java文件，我们使用javac对它们进行编译
```

这样我们就能初步的进行测试了，使用它内部带有的一些方法。官网上的一个例子：

```
$ grun Hello r -tree
(Now enter something like the string below)
hello parrt
(now,do:)
^D
(The output:)
(r hello parrt)
(That ^D means EOF on unix; it's ^Z in Windows.) The -tree option prints the parse tree in LISP notation.
It's nicer to look at parse trees visually.
$ grun Hello r -gui
hello parrt
^D
```

#### 在idea中的使用

用maven创建一个普通的项目，随后导入运行时所需要的库文件：

```
<dependency>
	<groupId>org.antlr</groupId>
    <artifactId>antlr4-runtime</artifactId>
    <version>4.7.2</version>
</dependency>
```

这个库文件中包含了对这个语法分析器使用的一些方法，比如创建一棵语法分析树，然后进行一些操作等等。而我们还可以在依赖文件中导入antlr专门将g4语法转换成java类的插件：

```
<build>
    <plugins>
        <plugin>
            <groupId>org.antlr</groupId>
            <artifactId>antlr4-maven-plugin</artifactId>
            <version>4.7.2</version>
        </plugin>
    </plugins>
</build>
```

当然有个最简单的方法，就是在idea的setting中的plugins中直接安装antlr v4 grammar的插件，这样我们右击g4语法文件，就直接有generate antlr recognizer的选项，当然最好第一步是进行configure antlr(右击g4文件)，指定一下生成类文件的输出路径。

## 语法

这个语法分析器的一个主要的内容就是要编写自己的语法规则，从而对输入的语句进行语法分析，生成语法分析树。首先看一下这个语法文件中都有哪些部分的内容。语法文件中可能出现一下几种量：

* 标识符：也就是变量的意思，在下面会遇到专门叫做**标记**的成员，该成员经常以大写字母开头；那么一般和规则有关的标识符都是以小写字母开头的。

* 常量：常量字符串都是用单引号括起来的，比如‘、’

* 动作：这个相当于是内嵌的一些函数，需要用目标语言编写，用{}括住。

* 关键字：这些关键字是不能用来用作变量的：

  ```
  import, fragment, lexer, parser, grammar, returns,
  locals, throws, catch, finally, mode, options, tokens
  ```

总的来说，一个语法文件的结构可以写成下面这样：

```
grammar Name;
options {...}
import ... ;
 	
tokens {...}
channels {...} // lexer only
@actionName {...}
 	 
rule1 // parser and lexer rules, possibly intermingled
...
ruleN
```

首先grammar关键字定义了这个语法文件的一个名称，后面跟着的Name是用户自己起的语法名，这个语法名一定是和g4文件的文件名相同的。如果说你在grammar关键字前不加任何修饰，则默认这个语法是一个语法和词法混合的分析文件，当然你也可以指定这个文件是专门的语法或者词法的：

```
lexer grammar Name;
parser grammar Name;
```

### import

import关键字用来引入外部的语法文件，就像是c语言的include一样。那么引入语法之后，引入文件中定义的语法规则将被导入该文件中，当然其中会有一些问题：

* 如果有重复的语法规则出现，则这里默认选择最先出现的版本。
* 词法文件只能引入词法文件，语法文件只能引入语法文件。
* 如果引入的语法功能上出现重叠，会保留各自双方的功能。

### Token

记号是由字符汇聚起来的一个东西，那么分析这样一个过程称为词法分析，所以可以看到很多词法规则中定义了许多的Token，并且这些Token也会用到规则的书写中来。

### parser rules

语法文件中所谓的规则就是包含了一个规则名加上规则描述：

```
retstat : 'return' expr ';' ;
```

如果规则存在多种：

```
retstat : 'return' expr ';' 
		| 'balabala' // 用|来划分不同的规则
		;
```

我们有的时候会看到一些规则的后面加上了带有#的描述：

```
e   : e '*' e # Mult
    | e '+' e # Add
    | INT # Int
    ;
```

这是为了在java类中能够生成更加具体的监听语法树的一些方法，比如：

```
void enterMult(AParser.MultContext ctx);
void exitMult(AParser.MultContext ctx);
void enterAdd(AParser.AddContext ctx);
void exitAdd(AParser.AddContext ctx);
void enterInt(AParser.IntContext ctx);
void exitInt(AParser.IntContext ctx);
```

为什么会有这些方法？在解析成java类文件的时候，antlr将一条规则的上下文用一个类来表示，例如下面这个例子：

```
inc : e '++' ;

public static class IncContext extends ParserRuleContext {
 	public EContext e() { ... } // e在这里也应该是一个规则，所以在这个inc规则中要被使用到
 	...
} // 如果规则里有多个e的出现，则该类中会给每个e一个对象
```

我们可以在规则中使用=来给一个变量赋值，例如下面这个例子：

```
stat: 'return' value=e ';' # Return
 	| 'break' ';' # Break
 	;
 	
public static class ReturnContext extends StatContext {
 	public EContext value;
 	...
}
```

### Actions and Attributes

动作其实就是在比如在某个规则的后面加入一个用花括号包起来的目标语言的程序代码，举个例子：

```
decl: type ID ';' {System.out.println("found a decl");} ;
type: 'int' | 'float' ;
```

不一定非得要是在完整的规则后面，穿插在规则中也是允许的，一方面确实这个可以用作是日志的功能。

如果我们使用Token，每一个Token都有自己的属性，只不过这些属性都是只读的。使用的时候需要使用$token.attr来用，举个例子：

```
r : INT {int x = $INT.line;}
    ( ID {if ($INT.line == $ID.line) ...;} )?
    a=FLOAT b=FLOAT {if ($a.line == $b.line) ...;}
  ;
```

| Attribute | Type   | Description                                                  |
| --------- | ------ | ------------------------------------------------------------ |
| text      | String | The text matched for the token; translates to a call to getText. Example: $ID.text. |
| type      | int    | The token type (nonzero positive integer) of the token such as INT; translates to a call to getType. Example: $ID.type. |
| line      | int    | The line number on which the token occurs, counting from 1; translates to a call to getLine. Example: $ID.line. |
| pos       | int    | The character position within the line at which the token’s first character occurs counting from zero; translates to a call togetCharPositionInLine. Example: $ID.pos. |
| index     | int    | The overall index of this token in the token stream, counting from zero; translates to a call to getTokenIndex. Example: $ID.index. |
| channel   | int    | The token’s channel number. The parser tunes to only one channel, effectively ignoring off-channel tokens. The default channel is 0 (Token.DEFAULT_CHANNEL), and the default hidden channel is Token.HIDDEN_CHANNEL. Translates to a call to getChannel. Example: $ID.channel. |
| int       | int    | The integer value of the text held by this token; it assumes that the text is a valid numeric string. Handy for building calculators and so on. Translates to Integer.valueOf(text-of-token). Example: $INT.int. |

同样对于语法规则而言也有自己的属性：

| Attribute | Type              | Description                                                  |
| --------- | ----------------- | ------------------------------------------------------------ |
| text      | String            | The text matched for a rule or the text matched from the start of the rule up until the point of the `$text` expression evaluation. Note that this includes the text for all tokens including those on hidden channels, which is what you want because usually that has all the whitespace and comments. When referring to the current rule, this attribute is available in any action including any exception actions. |
| start     | Token             | The first token to be potentially matched by the rule that is on the main token channel; in other words, this attribute is never a hidden token. For rules that end up matching no tokens, this attribute points at the first token that could have been matched by this rule. When referring to the current rule, this attribute is available to any action within the rule. |
| stop      | Token             | The last nonhidden channel token to be matched by the rule. When referring to the current rule, this attribute is available only to the after and finally actions. |
| ctx       | ParserRuleContext | The rule context object associated with a rule invocation. All of the other attributes are available through this attribute. For example, `$ctx.start` accesses the start field within the current rules context object. It’s the same as `$start`. |

还有一种叫做动态扩展的属性，这个后期研究一下啦。

### lexer rules

词法规则书写起来和语法规则类似，它返回匹配规则的token：

```
TokenName: choice1 | choice2 | choice3...
fragment Token: choice1 | choice2 ...
```

fragment关键字用来用来给Token做补充的功能，比如如下的使用：

```
INT : DIGIT+ ; // references the DIGIT helper rule
fragment DIGIT : [0-9] ; // not a token by itself
```

词法分析中还能够指定多个mode，如果我们不指定，我们默认是在default mode下，如果我们写成如下：

```
rules in default mode
...
mode MODE1;
rules in MODE1
...
```

这个功能相当于把词法规则分成了多个组，一般规则都会进入default部分进行匹配，除非指定了某个特定的模式。

## 构建应用程序中的使用

在程序中我们希望根据收集到的语句来构建一个语法分析树，那么我们要知道分析一个语句所要经过的流程。首先我们需要使用词法分析器来处理字符序列，将生成的词法符号交给语法分析器；随后语法分析器根据这些信息检查语法从而构造出一棵语法分析树。要想经历这个过程，首先要使用charStream读入语句，随后交给Lexer类来实例化，接着交付给Parser类解析，而其中链接它们的桥梁是TokenStream类，最后ParseTree类获取生成的语法分析树。

每一条规则都持有一个所谓根节点，而这个根节点包含了该规则的识别过程的全部信息，这些根节点被称为**上下文信息Context**。

### Tree Listener

antlr分析器一个重要的功能就是生成一颗语法树。分析树的叶子结点通常时输入的token，中间的结点可能是语法规则的名字，然后一层一层展开。一般而言规则分析器会在接收输入之后自动生成对应的语法树。

在antlr中存在一个叫做ParseTreeWalker，它知道应该如何遍历这颗语法树，并且触发监听器中实现的事件，在你对g4文件转换java的过程中，会自动给你生成一个监听器接口。类似于如下：

```
public interface JavaListener extends ParseTreeListener<Token> {
  void enterClassDeclaration(JavaParser.ClassDeclarationContext ctx);
  void exitClassDeclaration(JavaParser.ClassDeclarationContext ctx);
  void enterMethodDeclaration(JavaParser.MethodDeclarationContext ctx);
 ...
}
```

里面有对于每一个进入规则和退出规则的接口函数。我们可以自行实现监听器每个方法中的具体内容，而监听器的好处在于，使用ParseTreeWalker类(深度优先遍历)进行遍历时，这些listener接口函数会被自动触发。

### Tree Visitor

当然在有些时候我们需要显示地触发访问子节点的函数，listener不能满足这个需求因为它是全自动的，所有就要借助Visitor。当我们在编译时只需要在命令行中加入-visitor选项就可以生成visitor的接口。

