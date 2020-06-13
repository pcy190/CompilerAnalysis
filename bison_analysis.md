## table分析
和flex类似，bison会有一系列用于parse的表，而且经过压缩处理。在理顺这些表的作用和索引方式后，对yyparse的理解就方便很多了。

### yytranslate
这个表映射了token字符和symbol token之间的关系。如果是预定义的`%token`,则会映射到别的token序号去；如果只是简单的终结符，则会把对应字符的ascii码映射为对应的token。
例如`S: '\n' ;`规则在生成后如下
```
static const yytype_uint8 yytranslate[] =
{
       0,     2,     2,     2,     2,     2,     2,     2,     2,     2,
       3,     2,     2,     2,     2,     2,     2,     2,     2,     2,
...
       2,     2,     2,     2,     2,     2,     2,     2,     2,     2,
       2,     2,     2,     2,     2,     2,     1,     2
};
```
`\n` ascii为10，此处yytranslate[10]=3
大部分映射的token为2，而2是`YYEMPTY`未定义的。当yyparse()调用yylex()然后获取对应字符的symbol token内部码。

### yydefact
`yydefact`存储了每个状态下，产生式规则的序号(`rule number`)。
```
  /* YYDEFACT[STATE-NUM] -- Default reduction number in state STATE-NUM.
     Performed when YYTABLE does not specify something else to do.  Zero
     means the default is an error.  */
static const yytype_uint8 yydefact[] =
{
       0,     2,     0,     1
};
```
在STATE-NUM状态下，通过YYDEFACT[STATE-NUM]获取其对应的产生式序号。

其中，`0`号表示error。由于默认`($accept → L $end)`rule(其rule number为1)的存在，我们定义的所有rule索引都会增加1。即我们定义的第一个rule `S: '\n' ;`, 其rule number在此处为2。

### yydefgoto
yydefgoto是GOTO跳转表的压缩形式。它的元素数量为语法中非终结符的数量。元素的值表示每个非终结符要跳转的state
```
  /* YYDEFGOTO[NTERM-NUM].  */
static const yytype_int8 yydefgoto[] =
{
      -1,     2
};
```
其规则为
```
yydefgoto[nth non-terminal] = most common GOTO state for the nth non-terminal.
```
索引方式为rule number减去非终结符的数目
```
  /* yyn is the number of a rule to reduce with.  */

yystate = yydefgoto[yyn - YYNTOKENS];
```
parser在用rule归约栈上内容的时候会查询yydefgoto，但在某些状态下会使用yytable表。

第一个非终结符`$accept`的rule为-1(即yydefgoto[0]),`$accept`不会被归约。

### yyr1 & yyr2
yyr1是每个rule左部的symbol number。0号不作为rule，其共有NRULES+1个元素
```
  /* YYR1[YYN] -- Symbol number of symbol that rule YYN derives.  */
static const yytype_uint8 yyr1[] =
{
       0,     4,     5
};
```
通过yyr1，在归约发生的时候，我们可以知道这个归约式左部的symbol并转移到对应状态。

yyr2表示每个rule右部的symbol数量
```

  /* YYR2[YYN] -- Number of symbols on the right hand side of rule YYN.  */
static const yytype_uint8 yyr2[] =
{
       0,     2,     1
};
```
例如rule2的`S: '\n'`右侧有1个symbol，因此yyr2[2]=1。在归约的时候，通过这个表中查找在对应的rule下，我们要从栈中pop出多少state来作为归约。

### yytable
yytable和yycheck, yypact, yypgoto 配合来表示当前的状态是应该放入parse stack中还是符合rule来进行归约。
```
#define YYTABLE_NINF -1

  /* YYTABLE[YYPACT[STATE-NUM]] -- What to do in state STATE-NUM.  If
     positive, shift that token.  If negative, reduce the rule whose
     number is the opposite.  If YYTABLE_NINF, syntax error.  */
static const yytype_uint8 yytable[] =
{
       1,     3
};
```
正数表示移入。负数表示归约，(其绝对值表示归约产生式的rule值，即如果为-3，则表示使用rule3来归约栈上的符号)。

YYTABLE_NINF用来表示我们在产生式中指定error情况下的状态。
>    If the value in YYTABLE is positive, we shift the token and go to that state.
   If the value is negative, it is minus a rule number to reduce by.
   If the value is YYTABLE_NINF, it's a syntax error.

### yypgoto
因为同一个非终结符，根据之前状态的不同，其跳转的状态也是不同的。公用的GOTO表定义在前文提到的`YYDEFGOTO`中。那么对于存在不同情况的GOTO情况，则由yypgoto记录。
```
  /* YYPGOTO[NTERM-NUM].  */
static const yytype_int8 yypgoto[] =
{
      -4,    -4
};
```
这个表的索引对应着所有的非终结符，此处的两个元素分别是`$accept`,`S`的入口

比如现在在rule2 (`S -> '\n'`)归约后，栈顶的状态为4，parser会将yypgoto[S]的值(此处yypgoto[1]=-4)加上原先的状态值作为现在的状态索引，去yytable找到新的状态值，即此时状态索引为0 (4-4)， 那么yytable[0]为1，那么现在的状态就是1，被压入状态栈顶。

那么parser怎么知道是选择yypgoto还是yydefgoto来设置接下来的状态呢?答案在接下来会将的`yycheck`中，即通过`yycheck`来选择对应的模式(yypgoto或yydefgoto)

### yypact
yypact定义了在S状态(初始状态)下的部分yytable。它由token标号来索引。
```
#define YYPACT_NINF -4
  /* YYPACT[STATE-NUM] -- Index in YYTABLE of the portion describing
     STATE-NUM.  */
static const yytype_int8 yypact[] =
{
      -3,    -4,     1,    -4
};
```
在parse循环的开始时，如果yypact[cur-state] = YYPACT_NINF， 意味着使用yydefact来归约。也即在这种状态下，只有归约没有移入的操作。
例如此处yypact中为YYPACT_NINF (`-4`)的状态有1和3.即在1号状态下，只进行对应的归约此操作。

如果现在在状态0，展望符为`\n`(即3号symbol,由yytranslate可得)，那么现在的yypact[0]为-3，所以在yytable中的索引应该为`-3+3`=0, yytable[0]为1，表示移入并转移到状态1。

### yycheck
yycheck主要有两个作用，一个是判断归约还是移入；一个是判断选择哪个跳转表。(设计yydefgoto 和yypgoto，将公共的goto提取出来到yydefgoto ，可以节省空间，而后再通过check的方式选择具体哪个表)

```
static const yytype_uint8 yycheck[] =
{
       3,     0
};
```
#### yytable or yydefact
源码注释[src/tables.h#L58](https://github.com/akimd/bison/blob/master/src/tables.h#L58)中可以看到，YYCHECK通过索引的界限来判断是否采用yytable
> YYCHECK = a vector indexed in parallel with YYTABLE.  It indicates,  in a roundabout way, the bounds of the portion you are trying to  examine.
   Suppose that the portion of YYTABLE starts at index P and the index
   to be examined within the portion is I.  Then if YYCHECK[P+I] != I,
   I is outside the bounds of what is actually allocated, and the
   default (from YYDEFACT or YYDEFGOTO) should be used.  Otherwise,
   YYTABLE[P+I] should be used.



如果现在在状态0，展望符为`\n`(即3号symbol)，那么现在的yypact[0]为-3，此时我们要通过yycheck来选择是使用yytable还是yydefact。我们查看yycheck[0]的值为3.

parse此时会判断yycheck[0]是否为所有有效token的symbol number, 如果不是，则进行yydefault的归约。因为要保证在yytable里面索引的值是有效的symbol序号才有意义。
```
/* If the proper action on seeing token YYTOKEN is to reduce or to
     detect an error, take that action.  */
  yyn += yytoken;
  if (yyn < 0 || YYLAST < yyn || yycheck[yyn] != yytoken)
    goto yydefault;
```
此处yycheck[0]为3，而`\n`正好是3号symbol，所以我们可以从`yytable`中去索引接下来的状态。

#### yydefgoto or yypgoto
除此以外，yycheck还帮助判断选择yydefgoto亦或是yypgoto.
```
  /* Now 'shift' the result of the reduction.  Determine what state
     that goes to, based on the state we popped back to and the rule
     number reduced by.  */

  yyn = yyr1[yyn];

  yystate = yypgoto[yyn - YYNTOKENS] + *yyssp;
  if (0 <= yystate && yystate <= YYLAST && yycheck[yystate] == *yyssp)
    yystate = yytable[yystate];
  else
    yystate = yydefgoto[yyn - YYNTOKENS];
```
比如现在在rule2 (`S -> '\n'`)归约后，栈顶的状态为4，parser会将yypgoto[S]的值(此处yypgoto[1]=-4)加上原先的状态值作为现在的状态索引4-4=0，查询yycheck，如果为4，则表示状态4为特殊状态，选择yytable; 否则就用yydefgoto表来决定跳转。



### 其他表

> yyrhs - A -1 separated list of RHS symbol numbers of all rules. yyrhs[n] = first symbol on the RHS of rule #n.

> yyprhs[n] - Index in yyrhs of the first RHS symbol of rule n.

> yyrline[n] - Line # in the .y grammar source file where rule n is defined.

> yytname[n] - A string specifying the symbol for symbol number n. yytoknum[n] - Token number of token n.

### 小结
- **yytranslate**: maps token numbers returned by yylex() to Bison's internal symbol 
- **yydefact**: lists default reduction rules for each state. 0 represents an error.
- **yydefgoto**: lists default GOTOs for each non-terminal symbol. It is only used after checking with yypgoto.
- **yyr1**: symbol number of lhs of each rule. Used at the time of a reduction to find the next state.
- **yyr2**: length of rhs of each rule. Used at the time of reduction to pop the stack.
- **yytable**: a highly compressed representation of the actions in each state. Negative entries represent reductions. There is a negative infinity to detect errors.
- **yypgoto**: accounts for non-default GOTOs for all non-terminal symbols.
- **yypact**: directory into yytable indexed by state number. The displacements in yytable are indexed by symbol number.
- **yycheck**: guard table used to check legal bounds within portions of yytable.


## bison parse

> 当bison读入一个终结符`token`，它会将该终结符及其语意值一起压入分析器栈`parser stack`。把一个token压入栈就是移入`shifting`操作。由于采用LALR，故parser会分析展望符，当一个终结符被读进来后，并不会马上移入`parser stack`,根据对应的展望符来确定接下来的移入归约/状态

yyparse主体框架(详见注释)
```
/* look-ahead symbol */
int yychar;

/* The semantic value of the look-ahead symbol.  */
YYSTYPE yylval;

int 
yyparse()
{
	int yystate; 	/* current state */
	int yyn;	/* This is an all purpose variable! One time it may represent a state, the next time, it might be a rule! */
	
	int yyresult;	/* Result of parse to be returned to the called */
	
	int yytoken=0;	/* current token */
	
	/* The state stack: This parser does not shift symbols on to the stack. Only a stack of states is maintained. */	
	
	 int yyssa[YYINITDEPTH];	/*YYINITDEPTH is 200 */
	 int *yyss = yyssa	/* Bottom of state stack */
	 int *yyssp;	/* Top of state stack */
	 
	 /* The semantic value stack: This stack grows parallel to the state stack. At each reduction, semantic values are popped 
	  * off this stack and the semantic action is executed
	  */
	  YYSTYPE yyvsa[YYINITDEPTH];
	  YYSTYPE *yyvs = yyvsa;	/* Bottom of semantic stack */
	  YYSTYPE *yyvsp;	/* Top of semantic stack */
	
	  /* POP the state and semantic stacks by N symbols - useful for reduce actions */
	  #define YYPOPSTACK(N)   (yyvsp -= (N), yyssp -= (N))
	  
	  
	  int yylen = 0;	/* This variable is used in reduce actions to store the length of RHS of a rule; */
	  
	  /* Ok done declaring variables. Set the ball rolling */
	  
	  yystate = 0;	/* Initial state */
	  yychar = YYEMPTY /* YYEMPTY is -2 */
	  
	  yyssp = yyss; 	/* Top = bottom for state stack */
	  yyvsp = yyvs;	/* Same for semantic stack */
	  
	  goto yysetstate; 	/* Well, gotos are used for extracting maximum performance. */
	  
	  
	  /* Each label can be thought of as a function */
	  
	  yynewstate:  /* Push a new state on the stack */
	  
	  		yyssp ++;	/*Just increment the stack top; actual 'pushing' will happen in yysetstate */
	  		
	  		
	  yysetstate:
	  		
	  		*yyssp = yystate;	/* Ok pushed state on state stack top */
	  		
	  		goto yybackup;	/* This is where you will find some action */
	  		
	  		
	  yybackup:	/* The main parsing code starts here */
	  	
	  		yyn = yypact[yystate];	/* Refer to what yypact is saying about the current state */
	  		
	  		if ( yyn == YYPACT_NINF) 	/* If negative infinity its time for a default reduction */
	  		
	  			goto yydefault;	/* This label implements default reductions; see below */
	  			
	  		
	  		/* Check if we have a look-ahead token ready. This is LALR(1) parsing */
	  		
	  		if (yychar == YYEMPTY)
	  			
	  			yychar = YYLEX; /* Macro YYLEX is defined as yylex() */
	  				
	  		if (yychar <= YYEOF) 	/* YYEOF is 0 - the token returned by lexer at end of input */
	  			
	  			yychar = yytoken = YYEOF; /* set all to EOF */
	  				
	  		else
	  		
	  			yytoken = yytranslate[yychar];	/* Translate the lexer token into internal symbol number */
	  				
	  		
	  		/* Now we have a look-ahead token. Let the party begin ! */
	  		
	  		
	  		yyn = yyn + yytoken;	/* This is yypact[yystate] + yytoken */
	  		
	  		
	  		/* Observe this check carefully. We are checking that yyn is within the bounds of yytable
	  		 * and also if yycheck contains the current token number.
	  		 */
	  		if ( yyn < 0 || YYLAST < yyn  || yycheck[yyn] != yytoken )	/* YYLAST is the highest index in yytable */
	  			  			 
	  			goto yydefault; 	/* Its time for a default reduction */
	  				
	  		/* Ok, yyn is within bounds of yytable */
	  		
	  		yyn = yytable[yyn];	/* This is yytable[ yypact[yystate] + yytoken ] */
	  		
	  		if (yyn <= 0)	/* If yytable happens to contain a -ve value, its not a shift - its a reduce */
	  		{
	  			if (yyn == 0 || yyn == YYTABLE_NINF)	/* But check for out of bounds condition*/
	  				goto yyerrlab;	/* Label to handle errors */
	  		
	  			/* Other wise reduce with rule # -yyn */
	  			
	  			yyn = -yyn;
	  			goto yyreduce; 	/* Label to implement reductions */
	  		}
	  		
	  		/* Last check: See if we reached final state! */
	  		if (yyn == YYFINAL)	/* YYFINAL is 8 in our case */
	  			YYACCEPT;	/* macro deined as 'goto acceptlab - a label to finish up */
	  			
	  		/* That completes all checks; If we reached here, there is no other option but to shift */
	  		
	  		yystate = yyn;	/* Now, yyn (= yytable[ yypact[yystate] + yytoken ]) is a state that has to be pushed */
	  		
	  		*++yyvsp = yylval; /* Push the semantic value of the symbol on the semantic stack */
	  		
	  		goto yynewstate;	/* This will increment state stack top and the following yysetstate that will do the pushing */
	  		
	  
	  yydefault:	/* A label to implement default reductions */
	  	
	  		yyn = yydefact[yystate];	/* Get the default reduction rule for this state */
	  		
	  		if ( yyn == 0 )	/* This state has no default reduction. Something is wrong */
	  		
	  			goto yyerrlab;
	  			
	  		goto yyreduce;	/* Ok, got the default reduction rule # in yyn; go ahead and reduce the stack */
	  		
	  	
	  
	  yyreduce:	/* A lablel that implements reductions on stack. */
	  	
	  		/* By the time we are here, yyn contains the rule# to use for reducing the stack. */
	  		
	  		/* Steps for reduction:
	  		 * 1. Find the length of RHS of rule #yyn
	  		 * 2. Execute any semantic actions by taking the values from the semantic stack
	  		 * 3. POP 'length' symbols from the state stack and 'length' values from semantic stack
	  		 * 4. Find the LHS of rule #yyn
	  		 * 5. Find the GOTO of state currently on top of stack on LHS symbol
	  		 * 6. Push that state on top of stack
	  		 * 
	  		 */
	  		 
	  		 yylen = yyr2[yyn];	/* Get length of RHS */
	  		 
	  		 /* Default semantic action - $$=$1 */
	  		 yyval = yyvsp[1-yylen];
	  		 
	  		 /* Execute semantic actions */
	  		 switch ( yyn )	/* Each rule has its own semantic action */
	  		 {
	  		 
	  		 	default:	break;	/* We didn't have any semantic actions in the grammar.*/
	  		 	
	  		 }
	  		 
	  		 YYPOPSTACK (yylen);	/* This will pop both state and semantic stacks. See definition of this macro above */
	  		 
	  		 yylen = 0;	/* re-initialize yylen */
	  		 
	  		 *++yyvsp  = yyval;	/* Push the result of semantic evaluation on top of semantic stack */
	  		 
	  		 
	  		 /* Now shift the result of reduction (steps 4 - 6) */
	  		 
	  		 yyn = yyr1[yyn];	/* Reuse yyn at every opportunity.  For now, yyn is the LHS symbol (number) of the rule */
	  		 
			 /* First check for anomalous GOTOs, otherwise use Default GOTO (YYDEFGOTO)
			  * 
			  * Observe that if we subtract no. of terminals (YYNTOKENS) from symbol number of a nonterminal, we get
			  * an index into yypgoto or yydefgoto for that non-terminal.
			  */
			  
	  		 yystate = yypgoto[yyn - YYNTOKENS] + *yyssp;
	  		 
	  		 /* A couple of checks are needed before we know this is not a default GOTO
	  		  * 1. yystate must be within bounds of yytable. ( 0 to YYLAST )
	  		  * 2. yycheck must contain the state currently on top of the stack
	  		  */	  		 
	  		 if ( 0 <= yystate && yystate <= YYLAST && yycheck[yystate] = *yyssp)
	  		 
	  		 	yystate = yytable[yystate];	/* Take the GOTO from yytable */
	  		 	
	  		 else
	  		 	
	  		 	yystate = yydefgoto[yyn - YYNTOKENS];	/* Otherwise use the default GOTO */
	  		 	
	  		 goto yynewstate;	/* Simply push the newly found state on top of stack and continue */
	  		 
}	/* End of yyparse() */
	  		 
```

## 总结
bison将上下文无关文法转换为LALR(1)。
可以看到，bison在压缩了table之后的查找索引方面有很多机制，在理顺了这些表之后，整个自动机的雏形也就差不多了。

murmur：感觉此处的表压缩效率还是不太高，有很多表例如yytranslate等等还是有很多重复的字符。这个地方或许可以用类似哈夫曼树的方式来继续压缩，然后在parse前做一段解压的过程，~~算是时间换空间~~，这样不会影响parse时的效率(~~既然追求空间，那就做到极致咯~~)bison生成的parser代码缩进还是比较舒服的。

另： 
- 使用`--report=all`选项编译可以输出一个LALR的状态信息。
- 设置`yydebug=1`/`--debug`可以看到parse时移入归约操作的输出，比较直观。

## reference
- [https://www.cs.uic.edu/~spopuri/cparser.html#lr-parser](https://www.cs.uic.edu/~spopuri/cparser.html#lr-parser)
- [https://blog.csdn.net/qq_35886593/article/details/90694664](https://blog.csdn.net/qq_35886593/article/details/90694664)
