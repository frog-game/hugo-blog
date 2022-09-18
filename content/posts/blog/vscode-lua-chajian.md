---
title: "微服务lua调试器"
date: 2022-09-14T01:30:29+08:00
lastmod: 2022-09-154T01:30:29+08:00
author: ["frog"]
keywords:
-
categories:
-
tags:
-
description: "针对云风skynet微服器框架写的vscode-lua调试器"
weight:
draft: false # 是否为草稿
comments: true
reward: true # 打赏
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    image: "posts/blog/vscode-lua-chajian/image-20220913233913512.png" #图片路径例如：posts/tech/123/123.png
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

## luadebug 实现多虚拟机原理

先从luahook原理说起

在lua.h 当中我们有 lua_sethook函数来给们设置钩子

```c
LUA_API void (lua_sethook) (lua_State *L, lua_Hook func, int mask, int count);
```

> - lua_State *L :虚拟机的地址
> - lua_Hook func:我们设定的钩子回调函数
> - mask: 状态掩码可以组合操作
>
>   ```
>   LUA_MASKCALL : 调用函数时回调
>                    
>   LUA_MASKRET :函数返回时回调
>                    
>   LUA_MASKLINE :执行一行代码时候回调
>                    
>   LUA_MASKCOUNT :每执行count条指令时候回调
>   ```
> - count：只有掩码包含LUA_MASKCOUNT 这个状态时候才有效果，代表执行count次才会回调一次钩子函数

`<font color='orange'>`**LUA_MASKCALL 会在调用函数时回调**  我们在追踪lua源码，可以发现在每次调用函数之前都回**ldo.c **去调用** luaD_precall **函数并检测是否设置了掩码标识，如果设置了** LUA_MASKCALL **掩码状态，就会调用 **luaD_hook** 这个回调函数

![202201211113975](202201211113975.png)

**`<font color='orange'>`LUA_MASKRET :会在函数返回时回调 **  我们在追踪lua源码，可以发现在每次函数返回时候都会去**ldo.c ** 里面调用**luaD_poscall ** 里面的**rethook **函数 如果设置了就会调用** LUA_MASKRET **掩码状态 ， 就会调用 **luaD_hook** 这个回调函数

![202201211154326](202201211154326.png)

![202201211156389](202201211156389.png)

**`<font color='orange'>`LUA_MASKLINE :执行一行代码时候回调 **  我们在追踪lua源码，可以发现在每次执行一行指令都会去**ldebug.c ** 去调用 **luaG_traceexec** 函数 如果设置了**LUA_MASKLINE ** 掩码状态 那么久会调用**luaD_hook **函数

![202201211515663](202201211515663.png)

**`<font color='orange'>`LUA_MASKCOUNT :执行count条指令时候回调 **  我们在追踪lua源码，可以发现每次执行一行指令都会去**ldebug.c ** 去调用 **luaG_traceexec** 函数 这个函数需要和**count **参数配合才能发挥效果，可以看到如果** L->hookcount **在一次次递减之后等于** 0 **了就会调用** luaD_hook **函数

![202201211515663](202201211515663-16530988286073.png)

综合上述我们看到最终都会调用到**luaD_hook **函数，仔细看源码观察可以看到在经过一系列判断以后会回调我们设置好的** L->hook **函数

![202201211215165](202201211215165.png)

回到我们第**2 **个参数

```c
/* Functions to be called by the debugger in specific events */
typedef void(*lua_Hook) (lua_State *L, lua_Debug *ar);
```

可以看到返回了一个**lua_Debug **结构体 我们进入这个结构体

![202201211242014](202201211242014-16530988621344.png)

这里为了兼容每个不同的lua版本，弄了个**union **联合体 写在了** lua_api_loder.h **里面

我们进入 **lua_Debug_54 **结构体里面

![202201211342005](202201211342005.png)

可以发现这里有许多的信息

| 结构变量                                                             | 解释                                                                                                                                      |
| :------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| event                                                                | Event codes  事件类型标识如下几种 [LUA_HOOKCALL,LUA_HOOKRET,LUA_HOOKLINE,LUA_HOOKCOUNT,LUA_HOOKTAILCALL] |
| name                                                                 | 函数名字                                                                                                                                  |
| namewhat                                                             | 作用域的含义，比如是global，local，method，field 或者""   ""代表没有找到这个函数                                                          |
| what                                                                 | `<span style="display:inline-block;width: 600px">`函数的类型 一般为"lua"                                                                |
| source                                                               | 函数定义的位置，如果是loadstring载入的，source是string 如果是在一个文件中source标识带有前缀的@文件名字                                    |
| srclen                                                               | source的长度                                                                                                                              |
| currentline                                                          | 当前函数所在的行                                                                                                                          |
| linedefined                                                          | 函数定义的首行地址                                                                                                                        |
| `<span style="display:inline-block;width: 120px">` lastlinedefined | 函数定义的最后一行的行号                                                                                                                  |
| nups                                                                 | 上值的个数                                                                                                                                |
| nparams                                                              | 参数数量                                                                                                                                  |
| isvararg                                                             | 是不是可变参数                                                                                                                            |
| istailcall                                                           | 是不是最后一个函数是一个函数调用 形如**function f(x) return g(x) end **                                  |
| ftransfer                                                            | 与第一个转移值的偏移量 主要用call/return方式                                                                                              |
| ntransfer                                                            | 传输的值 主要用call/return方式                                                                                                            |
| short_src                                                            | source 的简短表示                                                                                                                         |
| i_ci                                                                 | 记录一个函数调用涉及到的栈引用，lua在调用函数的时候会把每个callinfo用双向链表串起来                                                       |

综合上述原理我们可以看到每一个lua_sethook 被调用时候通过hook返回的信息有这么多，而且每一个lua_state 都是沙盒隔离，所以我们可以利用沙盒原理，通过在进程中创建一个debuggerManager的管理器把所有生成的lua_state的指针保存在这个管理器里面，这样在每次lua调用pcall执行脚本的时候都会去触发自己相对应的lua_sethook设置的hook函数，在里面获取当时触发的时候的上表返回的信息，然后给客服端显示

![202201211549501](202201211549501.png)

2. luadebug 实现修改变量值

首先需要创建一个protobuf的cmd命令

**MessageCMD::SetVariableReq **   //修改变量

客服端设置变量逻辑

![202201211620038](202201211620038.png)

服务器接收到请求逻辑核心逻辑如下

第一步: 将此lua定义check 用luaL_dostring 进行load执行

```c
const char* loadstr = "function dlua_setvarvalue (name, frame, val, level)\n"
			"    local found\n"
			"\n"
			"    -- try local variables\n"
			"    local i = 1\n"
			"    while true do\n"
			"        local n, v = debug.getlocal(frame + level, i)\n"
			"        if not n then\n"
			"            break\n"
			"        end\n"
			"        if n == name then\n"
			"            debug.setlocal(frame + level, i, val)\n"
			"            found = true\n"
			"        end\n"
			"        i = i + 1\n"
			"    end\n"
			"    if found then\n"
			"        return true\n"
			"    end\n"
			"\n"
			"    -- try upvalues\n"
			"    local func = debug.getinfo(frame + level).func\n"
			"    i = 1\n"
			"    while true do\n"
			"        local n, v = debug.getupvalue(func, i)\n"
			"        if not n then\n"
			"            break\n"
			"        end\n"
			"        if n == name then\n"
			"            debug.setupvalue(func, i, val)\n"
			"            return true\n"
			"        end\n"
			"        i = i + 1\n"
			"    end\n"
			"\n"
			"    return false\n"
			"end"
			"";

		luaL_dostring(L, loadstr);
```

如果是set a=1类型

```c
	std::string loadstr =
		"if not dlua_setvarvalue(\"" + val + "\"," + std::to_string(currentFrameId) + "," + input + ", 3" +
		") then\n";
	loadstr += val + "=" + input + "\n";
	loadstr += "end\n";
	int status = luaL_dostring(L, loadstr.c_str());
	if (status != 0)
	{
		std::string ret = lua_tostring(L, -1);
		lua_settop(L, oldn);
		EmmyFacade::Get().SendLog(LogType::Error, ret.c_str());
		return -1;
	}
```

如果是set [t1] t1.a=1 类型

```c
	std::string loadstr = "function dlua_set_val(";
		for (auto it = inputval.begin(); it != inputval.end();)
		{
			loadstr = loadstr + it->first;
			it++;
			if (it != inputval.end())
			{
				loadstr = loadstr + ",";
			}
		}
		loadstr = loadstr + ")\n" + val + "=" + input + "\n";
		loadstr = loadstr + "return ";
		for (auto it = inputval.begin(); it != inputval.end();)
		{
			loadstr = loadstr + it->first;
			it++;
			if (it != inputval.end())
			{
				loadstr = loadstr + ",";
			}
		}
		loadstr = loadstr + "\n end\n";


		int status = luaL_dostring(L, loadstr.c_str());
		if (status != 0)
		{
			std::string ret = lua_tostring(L, -1);
			lua_settop(L, oldn);

			EmmyFacade::Get().SendLog(LogType::Error, ret.c_str());
			return -1;
		}

		lua_settop(L, oldn);

		lua_getglobal(L, "dlua_set_val");
		if (!lua_isfunction(L, -1))
		{
			lua_settop(L, oldn);
			EmmyFacade::Get().SendLog(LogType::Error, "get dlua_set_val fail");
			return -1;
		}

		for (auto it = inputval.begin(); it != inputval.end(); it++)
		{
			if (!FindAndPushVal(L, it->first, currentFrameId))
			{
				lua_settop(L, oldn);
				EmmyFacade::Get().SendLog(LogType::Error, (std::string("can not find val ") + it->first).c_str());
				return -1;
			}
		}


		int ret = lua_pcall(L, inputval.size(), inputval.size(), 0);
		if (ret != 0)
		{
			std::string ret = lua_tostring(L, -1);
			lua_settop(L, oldn);
			EmmyFacade::Get().SendLog(LogType::Error, ret.c_str());
			return -1;
		}

		int index = -inputval.size();
		for (auto it = inputval.begin(); it != inputval.end(); it++)
		{
			std::string name = it->first;
			int curoldn = lua_gettop(L);

			lua_getglobal(L, "dlua_setvarvalue");
			if (!lua_isfunction(L, -1))
			{
				lua_settop(L, oldn);
				EmmyFacade::Get().SendLog(LogType::Error, "get dlua_setvarvalue fail");
				return -1;
			}

			lua_pushstring(L, name.c_str());
			lua_pushinteger(L,currentFrameId);
			lua_pushnil(L);
			lua_pushinteger(L, 2);
			lua_copy(L, index - 5, -2);


			ret = lua_pcall(L, 4, 1, 0);
			if (ret != 0)
			{
				std::string ret = lua_tostring(L, -1);
				lua_settop(L, oldn);

				EmmyFacade::Get().SendLog(LogType::Error, ret.c_str());
				return -1;
			}

			bool suc = lua_toboolean(L, -1);
			if (!suc)
			{
				lua_settop(L, oldn);
				EmmyFacade::Get().SendLog(LogType::Error, (std::string("dlua_setvarvalue set ") + name + " fail").c_str());
				return -1;
			}

			lua_settop(L, curoldn);
			index++;
		}
```

### 图片效果

![imgeimage-20220124163841008](imgeimage-20220124163841008.png)

## 真机调试

### adb forward 原理

要想知道怎么真机调试我们首先应该知道adb调试的原理

比如我现在调试安卓时候的使用的命令: adb forward tcp:8888 tcp:9966

![imgeimage-20220124142626506](imgeimage-20220124142626506.png)

通过此端口转发我们就可以做到吧电脑tcp端口的消息转发到真机里面tcp9966端口上

## 条件断点

条件断点分为

1：表达式 ![imgeimage-20220124162929882](imgeimage-20220124162929882.png)

2：命中次数

![imgeimage-20220124162952669](imgeimage-20220124162952669.png)

3：日志断点

![imgeimage-20220124163041652](imgeimage-20220124163041652.png)

首先需要在此结构中定义3个变量用于处理3中类型，然后通过vscode设置条件端点类型

部分核心代码

```c++
bool Debugger::ProcessBreakPoint(std::shared_ptr<BreakPoint> bp)
{
	if (!bp->condition.empty())
	{
		auto ctx = std::make_shared<EvalContext>();
		ctx->expr = bp->condition;
		ctx->depth = 1;
		bool suc = DoEval(ctx);
		return suc && ctx->result->valueType == LUA_TBOOLEAN && ctx->result->value == "true";
	}
	if (!bp->logMessage.empty())
	{
		DoLogMessage(bp);
		return false;
	}
	if (!bp->hitCondition.empty())
	{
		bp->hitCount++;

		return DoHitCondition(bp);
	}
	return true;
}
```

```c++
bool Debugger::DoEval(std::shared_ptr<EvalContext> evalContext)
{
	if (!currentL || !evalContext)
	{
		return false;
	}

	auto L = currentL;

	//auto* const L = L;
	// From "cacheId"
	if (evalContext->cacheId > 0)
	{
		lua_getfield(L, LUA_REGISTRYINDEX, CACHE_TABLE_NAME); // 1: cacheTable|nil
		if (lua_type(L, -1) == LUA_TTABLE)
		{
			lua_getfield(L, -1, std::to_string(evalContext->cacheId).c_str()); // 1: cacheTable, 2: value
			GetVariable(evalContext->result, -1, evalContext->depth);
			lua_pop(L, 2);
			return true;
		}
		lua_pop(L, 1);
	}
	// LOAD AS "return expr"
	std::string statement = "return ";
	statement.append(evalContext->expr);
	int r = luaL_loadstring(L, statement.c_str());
	if (r == LUA_ERRSYNTAX)
	{
		evalContext->error = "syntax err: ";
		evalContext->error.append(evalContext->expr);
		return false;
	}
	// call
	const int fIdx = lua_gettop(L);
	// create env
	if (!CreateEnv(evalContext->stackLevel))
		return false;
	// setup env
#ifndef EMMY_USE_LUA_SOURCE
	lua_setfenv(L, fIdx);
#elif defined(EMMY_LUA_51) || defined(EMMY_LUA_JIT)
    lua_setfenv(L, fIdx);
#else //52 & 53
    lua_setupvalue(L, fIdx, 1);
#endif
	assert(lua_gettop(L) == fIdx);
	// call function() return expr end
	r = lua_pcall(L, 0, 1, 0);
	if (r == LUA_OK)
	{
		evalContext->result->name = evalContext->expr;
		GetVariable(evalContext->result, -1, evalContext->depth);
		lua_pop(L, 1);
		return true;
	}
	if (r == LUA_ERRRUN)
	{
		evalContext->error = lua_tostring(L, -1);
	}

	return false;
}

```

```c++
void Debugger::DoLogMessage(std::shared_ptr<BreakPoint> bp)
{
	std::string& logMessage = bp->logMessage;
	// 为什么不用regex?
	// 因为gcc 4.8 regex还是空实现
	// 而且后续版本的gcc中正则表达式行为似乎也不太正常
	enum class ParseState
	{
		Normal,
		LeftBrace,
		RightBrace
	} state = ParseState::Normal;

	std::vector<LogMessageReplaceExpress> replaceExpresses;


	std::size_t leftBraceBegin = 0;

	std::size_t rightBraceBegin = 0;

	// 如果在表达式中出现左大括号
	std::size_t exprLeftCount = 0;


	for (std::size_t index = 0; index != logMessage.size(); index++)
	{
		char ch = logMessage[index];

		switch (state)
		{
		case ParseState::Normal:
			{
				if (ch == '{')
				{
					state = ParseState::LeftBrace;
					leftBraceBegin = index;
					exprLeftCount = 0;
				}
				else if (ch == '}')
				{
					state = ParseState::RightBrace;
					rightBraceBegin = index;
				}
				break;
			}
		case ParseState::LeftBrace:
			{
				if (ch == '{')
				{
					// 认为是左双大括号转义为可见的'{'
					if (index == leftBraceBegin + 1)
					{
						replaceExpresses.emplace_back("{", leftBraceBegin, index, false);
						state = ParseState::Normal;
					}
					else
					{
						exprLeftCount++;
					}
				}
				else if (ch == '}')
				{
					// 认为是表达式内的大括号
					if (exprLeftCount > 0)
					{
						exprLeftCount--;
						continue;
					}

					replaceExpresses.emplace_back(logMessage.substr(leftBraceBegin + 1, index - leftBraceBegin - 1),
					                              leftBraceBegin, index, true);


					state = ParseState::Normal;
				}
				break;
			}
		case ParseState::RightBrace:
			{
				if (ch == '}' && (index == rightBraceBegin + 1))
				{
					replaceExpresses.emplace_back("}", rightBraceBegin, index, false);
				}
				else
				{
					//认为左右大括号失配，之前的不做处理，退格一位回去重新判断
					index--;
				}
				state = ParseState::Normal;
				break;
			}
		}
	}

	std::stringstream message;

	if (replaceExpresses.empty())
	{
		message << logMessage;
	}
	else
	{
		// 拼接字符串
		// 怎么replace 函数都没有啊

		std::size_t start = 0;
		for (std::size_t index = 0; index != replaceExpresses.size(); index++)
		{
			auto& replaceExpress = replaceExpresses[index];
			if (start < replaceExpress.StartIndex)
			{
				auto fragment = logMessage.substr(start, replaceExpress.StartIndex - start);
				message << fragment;
				start = replaceExpress.StartIndex;
			}

			if (replaceExpress.NeedEval)
			{
				auto ctx = std::make_shared<EvalContext>();
				ctx->expr = std::move(replaceExpress.Expr);
				ctx->depth = 1;
				bool succeed = DoEval(ctx);
				if (succeed)
				{
					message << ctx->result->value;
				}
				else
				{
					message << ctx->error;
				}
			}
			else
			{
				message << replaceExpress.Expr;
			}

			start = replaceExpress.EndIndex + 1;
		}

		if (start < logMessage.size())
		{
			auto fragment = logMessage.substr(start, logMessage.size() - start);
			message << fragment;
		}
	}

	std::string baseName = BaseName(bp->file);

	EmmyFacade::Get().SendLog(LogType::Info, "[%s:%d] %s", baseName.c_str(), bp->line, message.str().c_str());
}
```

```c++
bool Debugger::DoHitCondition(std::shared_ptr<BreakPoint> bp)
{
	auto& hitCondition = bp->hitCondition;

	enum class ParseState
	{
		ExpectedOperator,
		// 大于
		Gt,
		// 小于
		Le,
		// 单等号 
		Eq,

		ExpectedHitTimes,

		ParseDigit,

		ParseFinish
	} state = ParseState::ExpectedOperator;

	enum class Operator
	{
		// 大于
		Gt,
		// 小于
		Le,
		// 小于等于
		LeEq,
		// 大于等于
		GtEq,
		// 双等号
		EqEq,
	} evalOperator = Operator::EqEq;

	unsigned long long hitTimes = 0;

	for (std::size_t index = 0; index != hitCondition.size(); index++)
	{
		char ch = hitCondition[index];

		switch (state)
		{
		case ParseState::ExpectedOperator:
			{
				if (ch == ' ')
				{
					continue;
				}

				if (ch == '=')
				{
					state = ParseState::Eq;
				}
				else if (ch == '<')
				{
					state = ParseState::Le;
				}
				else if (ch == '>')
				{
					state = ParseState::Gt;
				}
				else
				{
					return false;
				}

				break;
			}
		case ParseState::Eq:
			{
				if (ch == '=')
				{
					evalOperator = Operator::EqEq;
					state = ParseState::ExpectedHitTimes;
				}
				else
				{
					return false;
				}
				break;
			}
		case ParseState::Gt:
			{
				if (ch == '=')
				{
					evalOperator = Operator::GtEq;
					state = ParseState::ExpectedHitTimes;
				}
				else if (isdigit(ch))
				{
					evalOperator = Operator::Gt;
					hitTimes = ch - '0';
					state = ParseState::ParseDigit;
				}
				else if (ch == ' ')
				{
					evalOperator = Operator::Gt;
					state = ParseState::ExpectedHitTimes;
				}
				else
				{
					return false;
				}
				break;
			}
		case ParseState::Le:
			{
				if (ch == '=')
				{
					evalOperator = Operator::LeEq;
					state = ParseState::ExpectedHitTimes;
				}
				else if (isdigit(ch))
				{
					evalOperator = Operator::Le;
					hitTimes = ch - '0';
					state = ParseState::ParseDigit;
				}
				else if (ch == ' ')
				{
					evalOperator = Operator::Gt;
					state = ParseState::ExpectedHitTimes;
				}
				else
				{
					return false;
				}
				break;
			}
		case ParseState::ExpectedHitTimes:
			{
				if (ch == ' ')
				{
					continue;
				}
				else if (isdigit(ch))
				{
					hitTimes = ch - '0';
					state = ParseState::ParseDigit;
				}
				else
				{
					return false;
				}
				break;
			}
		case ParseState::ParseDigit:
			{
				if (isdigit(ch))
				{
					hitTimes = hitTimes * 10 + (ch - '0');
				}
				else if (ch == ' ')
				{
					state = ParseState::ParseFinish;
				}
				else
				{
					return false;
				}

				break;
			}
		case ParseState::ParseFinish:
			{
				if (ch == ' ')
				{
					break;
				}
				else
				{
					return false;
				}
				break;
			}
		}
	}

	switch (evalOperator)
	{
	case Operator::EqEq:
		{
			return bp->hitCount == hitTimes;
		}
	case Operator::Gt:
		{
			return bp->hitCount > hitTimes;
		}
	case Operator::GtEq:
		{
			return bp->hitCount >= hitTimes;
		}
	case Operator::Le:
		{
			return bp->hitCount < hitTimes;
		}
	case Operator::LeEq:
		{
			return bp->hitCount <= hitTimes;
		}
	}


	return false;
}
```

###  视频效果展示

####  多虚拟机测试

<iframe src="https://player.bilibili.com/player.html?aid=638985869&bvid=BV1iY4y1r7GE&cid=716979750&page=1" allowfullscreen="allowfullscreen" width="100%" height="500" scrolling="no" frameborder="0" sandbox="allow-top-navigation allow-same-origin allow-forms allow-scripts"></iframe>

#### linux测试

<iframe src="https://player.bilibili.com/player.html?aid=683901725&bvid=BV1sU4y1S7LS&cid=716979645&page=1" allowfullscreen="allowfullscreen" width="100%" height="500" scrolling="no" frameborder="0" sandbox="allow-top-navigation allow-same-origin allow-forms allow-scripts"></iframe>

####  真机测试

<iframe src="https://player.bilibili.com/player.html?aid=553968251&bvid=BV1Bv4y1P7cE&cid=716979648&page=1" allowfullscreen="allowfullscreen" width="100%" height="500" scrolling="no" frameborder="0" sandbox="allow-top-navigation allow-same-origin allow-forms allow-scripts"></iframe>

#### 插件下载地址

![image-20220913233913512](image-20220913233913512.png)

针对skynet这种微服器框架和自己从 `0开发的frog微服务框架`编写的lua调试器，感兴趣的可以去vscode商店进行

下载使用，有任何问题可以加QQ私聊
