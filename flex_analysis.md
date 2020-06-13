<!-- Title: flex analysis -->
<!-- author: HAPPY -->

本文大致从分析生成的lexer.yy.c、flex源码入手简单分析。

## 宏&函数定义分析
首先用一个空的规则文件，生成一个`lex.yy.c`
```
%%
```
输出文件见[https://pastebin.com/ZAeGWtUU](https://pastebin.com/ZAeGWtUU)
> 因为空白框架就有1.7k+行源代码，故略去一些宏定义的说明。

首先定义了Flex版本号
```
#define FLEX_SCANNER
#define YY_FLEX_MAJOR_VERSION 2
#define YY_FLEX_MINOR_VERSION 6
#define YY_FLEX_SUBMINOR_VERSION 0
#if YY_FLEX_SUBMINOR_VERSION > 0
#define FLEX_BETA
#endif
```

定义了在不同C标准下的各种整数类型以供后续使用。这其中为了防止类型被用户代码冲突，此处typedef了一些类型
```
#include <inttypes.h>
typedef int8_t flex_int8_t;
typedef uint8_t flex_uint8_t;
...
```
而后定义了buffer的结构
### buffer
```
struct yy_buffer_state
	{
	FILE *yy_input_file;

	char *yy_ch_buf;		/* input buffer */
	char *yy_buf_pos;		/* current position in input buffer */

	//	input buffer的大小 (不包括End Of Block的字符空间)
	yy_size_t yy_buf_size;

	// 已经读入yy_ch_buf的字符数量，即已经使用的buffer大小 (不包括End Of Block字符)
	int yy_n_chars;

	// 改buffer是否归属自己, 以便自行通过realloc free等调整堆缓存
	int yy_is_our_buffer;

	// 如果是interactive,则从stdio得到输入,使用getc函数;反之使用fread()
	int yy_is_interactive;

	// 是否在行首 (begin of line), 用于 '^' 规则的匹配
	int yy_at_bol;

    int yy_bs_lineno; /**< The line count. */
    int yy_bs_column; /**< The column count. */
    
	/* Whether to try to fill the input buffer when we reach the
	 * end of it.
	 */
	int yy_fill_buffer;

	int yy_buffer_status;

	// 三种不同的状态，用以表示在读入EOF后是否还需要从input流中读入
#define YY_BUFFER_NEW 0
#define YY_BUFFER_NORMAL 1
#define YY_BUFFER_EOF_PENDING 2
	};
```
在action匹配前做的工作
```
#define YY_DO_BEFORE_ACTION \
	(yytext_ptr) = yy_bp; \
	yyleng = (size_t) (yy_cp - yy_bp); \
	(yy_hold_char) = *yy_cp; \
	*yy_cp = '\0'; \
	(yy_c_buf_p) = yy_cp;
```

### INPUT IO
虽然DFA在状态转移的过程中一次前进一个字符，但是为了提高IO效率，实际从文件读取的时候一般是批量往缓冲区读入的。如果有需要微调这个读入策略的需求，可以通过定义`YY_INPUT`宏来实现。默认的实现中，考虑了前文注释中的interactive模式对输入源的判断
```
#ifndef YY_INPUT
#define YY_INPUT(buf,result,max_size) \
	if ( YY_CURRENT_BUFFER_LVALUE->yy_is_interactive ) \
		{ \
		int c = '*'; \
		size_t n; \
		for ( n = 0; n < max_size && \
			     (c = getc( yyin )) != EOF && c != '\n'; ++n ) \
			buf[n] = (char) c; \
		if ( c == '\n' ) \
			buf[n++] = (char) c; \
		if ( c == EOF && ferror( yyin ) ) \
			YY_FATAL_ERROR( "input in flex scanner failed" ); \
		result = n; \
		} \
	else \
		{ \
		errno=0; \
		while ( (result = fread(buf, 1, max_size, yyin))==0 && ferror(yyin)) \
			{ \
			if( errno != EINTR) \
				{ \
				YY_FATAL_ERROR( "input in flex scanner failed" ); \
				break; \
				} \
			errno=0; \
			clearerr(yyin); \
			} \
		}\
\
#endif
```
一般从文件中读取就等价于
```
#define YY_INPUT(buf,result,max_size) \
    if ( (result = fread( (char*)buf, sizeof(char), max_size, fin)) < 0) \
        YY_FATAL_ERROR( "read() in flex scanner failed");
```

## DFSA分析
### DFA状态转移表结构分析
加下来会看到一系列数组常量的定义。
如果使用默认选项编译(例如`flex happy.l`),那么会看到如下的状态表
```
yyconst flex_int16_t yy_accept[6]
yyconst YY_CHAR yy_ec[256]
yyconst YY_CHAR yy_meta[2]
yyconst flex_uint16_t yy_base[7]
yyconst flex_int16_t yy_def[7]
yyconst flex_uint16_t yy_nxt[5]
yyconst flex_int16_t yy_chk[5]
```
 > 这些是被压缩后的表信息，对我们的后续分析来说，采用原始表更便利。默认选项中，flex会输出压缩后的状态转移表，而完整版本的矩阵是Nx128大小（其中N是自动机的状态数，128则是字符集大小）
  
未压缩的tables结构如下
```
static yyconst flex_int16_t yy_nxt[][128] = {...}
static yyconst flex_int16_t yy_accept[..] = {...}
```
- yy_accept是accept的状态
- yy_nxt是状态跳转表

使用下面的选项来不压缩tables
```
flex -Cf happy.l
```
使用`--verbose`/`-v`能显示DFA的分析数据
```
flex --verbose happy.l 

flex version 2.6.0 usage statistics:
  scanner options: -vI8 -Cem
  6/2000 NFA states
  4/1000 DFA states (5 words)
  0 rules
  Compressed tables always back-up
  1/40 start conditions
  5 epsilon states, 2 double epsilon states
  1/100 character classes needed 0/500 words of storage, 0 reused
  2 state/nextstate pairs created
  2/0 unique/duplicate transitions
  6/1000 base-def entries created
  4/2000 (peak 1) nxt-chk entries created
  2/2500 (peak 2) template nxt-chk entries created
  0 empty table entries
  1 protos created
  2 templates created, 2 uses
  1/256 equivalence classes created
  1/256 meta-equivalence classes created
  0 (0 saved) hash collisions, 1 DFAs equal
  0 sets of reallocations needed
  277 total table entries needed
```
这里使用的`happy.l`是空定义文件，这里可以看到，在初始状态下有6个NFA状态和4个DFA状态。

例如一个规则表达式`a(b|c)d*e+`
![](https://upload-images.jianshu.io/upload_images/13348817-770a614f0461a6d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

会产生如下选择表
![](https://upload-images.jianshu.io/upload_images/13348817-271e13b75a112967.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- a表示可接受的状态，accept
- m(state)表示跳转到state状态
- 空白处表示error

写一个示例
```flex
%%
a(b|c)d*e+ ;
%%
```
用`flex -vC happy.l`编译后发现有8个DFA状态，推测是基础的4个状态加上`a(b|c)d*e+`规则中的4个状态。(此处我们仅关心DFA状态，因为flex会把规则转换为NFA，再转换为DFA，中间会有状态化简)
```
 18/2000 NFA states
  8/1000 DFA states (19 words)
```
此时`yyconst flex_uint16_t yy_nxt[620]`中next状态共有8组数据，分别用一个空行隔开(其中第一个0表示BEGIN状态)。每组状态都是10*10的数据表
若采用`-Cf`编译，能得到如下的next表,具体内容可见[https://pastebin.com/VYqCwCh5](https://pastebin.com/VYqCwCh5)

```
static yyconst flex_int16_t yy_nxt[][128];
```
表中，大于0的部分表示要跳转到的下一个状态，小于0的部分表示终止状态(其绝对值表示对应的状态号) ~~这样设计(估计)为了方便由在EOB时获得上一个状态~~

### DFA工作流程分析
经过一连串的宏定义和结构表定义，我们来到了scanner的主入口`YY_DECL`
删去了不必要的宏定义后，大致简化代码如下

(此处小声吐槽一下flex生成的代码，在switch的地方代码缩进错乱，某些地方还有莫名其妙多的空格)
```

/** The main scanner function which does all the work.
 */
YY_DECL
{
	yy_state_type yy_current_state;
	char *yy_cp, *yy_bp;
	int yy_act;

	if ( !(yy_init) )
		{
		(yy_init) = 1;

		if ( ! (yy_start) )
			(yy_start) = 1;	// 初始状态定义

		/* 设置文件输入、输出指针 */
		if ( ! yyin )
			yyin = stdin;

		if ( ! yyout )
			yyout = stdout;

		/* 对非owner的buffer使用stack方式管理*/
		if ( ! YY_CURRENT_BUFFER ) {
			yyensure_buffer_stack ();
			YY_CURRENT_BUFFER_LVALUE =
				yy_create_buffer(yyin,YY_BUF_SIZE );
		}

		yy_load_buffer_state( );
		}

	{
	while ( /*CONSTCOND*/1 )		/* loops until end-of-file is reached */
		{
		// 设置各种buffer处的指针，以便在匹配成功时，通过yytext获取对应字符串

		yy_cp = (yy_c_buf_p);

		/* Support of yytext. */
		*yy_cp = (yy_hold_char);

		/* yy_bp points to the position in yy_ch_buf of the start of
		 * the current run.
		 */
		yy_bp = yy_cp;

		// 默认从start状态开始
		yy_current_state = (yy_start);
yy_match:
		// 开始进行状态转移, 通过yy_nxt表，结合yy_current_state和当前读入的字符来索引跳转状态, 直到无法转移

		
		// YY_SC_TO_UI将字符转换为对应的ascii  ->  #define YY_SC_TO_UI(c) ((unsigned int) (unsigned char) c)
		// 此处的转换挺严谨的
		while ( (yy_current_state = yy_nxt[yy_current_state][ YY_SC_TO_UI(*yy_cp) ]) > 0 )
			++yy_cp;

		yy_current_state = -yy_current_state;

yy_find_action:
		// 状态转移完了，查看转移后的状态是否出于可接受状态
		yy_act = yy_accept[yy_current_state];

		YY_DO_BEFORE_ACTION;

do_action:	/* This label is used only to access EOF actions. */

switch ( yy_act )	{ /* beginning of action switch */
	case 1:
		// yy_accept中值为1为接受状态，其他状态不合法 
		{return true ;}
		YY_BREAK

	case YY_STATE_EOF(INITIAL):
		yyterminate();

	case YY_END_OF_BUFFER:
		{
		/* Amount of text matched not including the EOB char. */
		int yy_amount_of_matched_text = (int) (yy_cp - (yytext_ptr)) - 1;

		// 还原YY_DO_BEFORE_ACTION做出的变动
		*yy_cp = (yy_hold_char);
		YY_RESTORE_YY_MORE_OFFSET

		// 调整buffer，其中有对EOB的不同情况的处理等等
		}
	default:
		YY_FATAL_ERROR("fatal flex scanner internal error--no action found" );
		} /* end of action switch */
	} /* end of scanning one token */
	} /* end of user's declarations */
} /* end of yylex */
```
其DFA的伪代码可以抽象成
```c
state= 0; 
get next input character
while (not end of input) {
	depending on current state and input character
		match: /* input expected */
			calculate new state; get next input character
		accept: /* current pattern completely matched */
			state= 0; perform action corresponding to pattern
		error: /* input unexpected */
			state= 0; echo 1st character input after last accept or error;
			reset input to 2nd character input after last accept or error;
}
```
简单来说，就是从同样的start状态，通过每个读入字符结合转移表来跳转状体，然后在无法继续转移，或者字符已经读完的时候，判断一下是否停在了接受状态，然后进行对应的用户定义的规则代码。

### 压缩后的状态转移矩阵
前面提到，默认情况下flex会对矩阵进行压缩。会得到类似的一些列表
```
yyconst flex_int16_t yy_accept[10]
yyconst YY_CHAR yy_ec[256]
yyconst YY_CHAR yy_meta[7]
yyconst flex_uint16_t yy_base[12]
yyconst flex_int16_t yy_def[12]
yyconst flex_uint16_t yy_nxt[19]
yyconst flex_int16_t yy_chk[19]
```
此时，在match中的代码变换为
```
yy_current_state = (yy_start);
yy_match:
	do
		{
		YY_CHAR yy_c = yy_ec[YY_SC_TO_UI(*yy_cp)] ;
		if ( yy_accept[yy_current_state] )
			{
			(yy_last_accepting_state) = yy_current_state;
			(yy_last_accepting_cpos) = yy_cp;
			}
		while ( yy_chk[yy_base[yy_current_state] + yy_c] != yy_current_state )
			{
			yy_current_state = (int) yy_def[yy_current_state];
			// 10 为Magic Number，随accept数量变化
			if ( yy_current_state >= 10 )
				yy_c = yy_meta[(unsigned int) yy_c];
			}
		yy_current_state = yy_nxt[yy_base[yy_current_state] + (unsigned int) yy_c];
		++yy_cp;
		}
		// 12也是Magic Number
	while ( yy_base[yy_current_state] != 12 );
```
其中用到了yy_meta，这方面代码没有注释阐述。遂在[flex源码](https://github.com/westes/flex/blob/98018e3f58d79e082216d406866942841d4bdf8a/src/gen.c)中找相应的压缩处理逻辑
[src/gen.c#L742](https://github.com/westes/flex/blob/98018e3f58d79e082216d406866942841d4bdf8a/src/gen.c#L742)
```
/* lastdfa + 2 is the beginning of the templates */
out_dec ("if ( yy_current_state >= %d )\n", lastdfa + 2);
```
可知代码中10这个魔数是由`lastdfa + 2`确定的,`lastdfa+2`在注释中为templates的开始序号。(templates估计是接在状态表之后的表，用来节省空间。)
其对应的[src/gen.c#L335](https://github.com/westes/flex/blob/98018e3f58d79e082216d406866942841d4bdf8a/src/gen.c#L335)压缩表生成中有如下注释
```
	/* We want the transition to be represented as the offset to the
	 * next state, not the actual state number, which is what it currently
	 * is.  The offset is base[nxt[i]] - (base of current state)].  That's
	 * just the difference between the starting points of the two involved
	 * states (to - from).
	 *
	 * First, though, we need to find some way to put in our end-of-buffer
	 * flags and states.  We do this by making a state with absolutely no
	 * transitions.  We put it at the end of the table.
	 */

	/* We need to have room in nxt/chk for two more slots: One for the
	 * action and one for the end-of-buffer transition.  We now *assume*
	 * that we're guaranteed the only character we'll try to index this
	 * nxt/chk pair with is EOB, i.e., 0, so we don't have to make sure
	 * there's room for jam entries for other characters.
	 */
```
由于压缩逻辑太长，从此可以粗略看到在压缩后的状态转移表中，转移到下一个状态所用的transition表中包含的并不是下一个状态的索引号，而是下一个状态索引和当前状态的offset。在使用时即为`base[nxt[i]]`。

这样设计之后，会把action 和end-of-buffer transition的状态放在表的最后。

这样一来，`lastdfa+1`的地方用于表示action匹配状态，`lastdfa+2`表示EOB的结束状态，也就能阐释之前魔数`lastdfa`的由来了。

## 总结
基本上flex生成的代码还是通过匹配循环来实现`规则->行为`的模式，通过压缩表的方式实现代码速度和体积上的优化。对于从用户的模式定义到表的生成，少不了正则表达式引擎和编译器的智慧结晶。

## reference
- [http://www.cs.man.ac.uk/~pjj/cs211/ho/node6.html](http://www.cs.man.ac.uk/~pjj/cs211/ho/node6.html)
- [https://blog.finaltheory.me/research/Flex-Tricks.html](https://blog.finaltheory.me/research/Flex-Tricks.html)
