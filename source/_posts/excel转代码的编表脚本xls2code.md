---
title: excel转代码的编表脚本xls2code
date: 2021-04-15 23:48:21
tags: [Python,游戏开发]	
categories: 工具
---

游戏行业中，策划的主力输出工具是excel，他们需要在一张张excel中不断填写和修改数据，因为游戏行业的多变性，因此策划需要经常反复地对一个系列数据进行修改，然后再到游戏里观察效果，再不断地作出调整。而这些excel表的修改反映到游戏里就是一个个功能点的调整，在游戏程序员看来，他们修改的是游戏程序里的参数和配置。因此，为了适应策划善变的想法，游戏开发中需要一个自动化编表工具，自动将一张张excel表转化为代码，做到自动化更新代码配置，而无需程序员主动参与。这就释放了游戏程序开发者的双手，让策划承担了一部分游戏开发者的工作，提升了整体游戏开发的效率。

要做这么一个编表工具，基本都是从以下几点出发来设计：
1. 与策划沟通excel的数据格式
2. 设计自己的代码模板template
2. 将excel表的数据读入内存
3. 按照指定的格式解析数据
4. 将解析后的数据按照代码模板生成出目标代码文件

这里以excel文件转python代码的实践作为例子，解析编表过程。总体来讲，使用Python + jinja2库，就能实现一套通用的编表脚本。


# excel转python文件
## 1. 填excel表，保存为后缀名为.xls文件
首先需要跟策划商量需求，他们打算需要什么数据，而这些数据到最后又该以什么形式呈现在代码里。格式如下，一行为一条数据，首列是该行数据的唯一ID。

![image](1.png)


## 2. 在.xls里建立CONFIG表，配置相关编表信息
在excel文件里新建一个CONFIG表，作为这个excel文件的数据解析schema。我们在这个SHEET里，定义好输入的Excel文件路径、输出文件路径、代码模板路径以及excel中每一列的数据定义。

![image](2.png)

## 3. 填写template代码模板
填写template代码模板，需要借助jinja2来完成代码的生成，该模板需要放在template文件夹内。一般而言，如果你的编程语言是Python，那我们就把excel数据转化为字典来存储，此时需要保证key要唯一，这里的key对用也就是excel表中的第一列，一般都定义为唯一ID。
```

################## 以下是自动生成的代码 ##################
## 比赛系统  lijunshi2015@163.com

TimeConf = { {% for v in content.list %}
{{v.id}} : {
	"season_name"	: "{{v.season_name}}",
	"release_start"	: "{{v.release_start}}",
	"release_end"	: "{{v.release_end}}",
	"exchange_end"	: "{{v.exchange_end}}",
	"turnplate_id"	: {{v.turnplate_id}},
},{% endfor %}
}

def get_conf():
	return TimeConf;



################## 以上是自动生成的代码 ##################

################## 以下是手工编写部分 ##################
```

## 4. 运行自动编表脚本，参数传入的是要处理的xls文件
```
python3 xls2code.py ./xls/game_season.xls
```
## 5. 最后会自动生成配置代码，放在auto_codes文件夹内
```


################## 以下是自动生成的代码 ##################
## 比赛系统  lijunshi2015@163.com

TimeConf = { 
2101 : {
	"season_name"	: "2021年第一赛季",
	"release_start"	: "2021-03-09 08:00:00",
	"release_end"	: "2021-05-09 24:00:00",
	"exchange_end"	: "2021-05-11 08:00:00",
	"turnplate_id"	: 21013001,
},
2102 : {
	"season_name"	: "2021年第二赛季",
	"release_start"	: "2021-05-11 08:00:00",
	"release_end"	: "2021-07-11 24:00:00",
	"exchange_end"	: "2021-07-13 08:00:00",
	"turnplate_id"	: 21023001,
},
}

def get_conf():
	return TimeConf;



################## 以上是自动生成的代码 ##################

################## 以下是手工编写部分 ##################

```

## 6. 自己的业务代码调用配置代码，例子是demo_main.py
```
import auto_codes.game_time_conf as time_conf

conf_time = time_conf.get_conf()
print(conf_time)
```


# excel转cpp文件的步骤
编表工具理论上可以通过xls生成任何语言的代码，关键在于配置编写指定语言的template，比如我们这次要通过excel表生成C++的头文件，方便被引用，可以参考xls/game_season_cpp.xls和模板template/game_timne_cpp.template.使用xls2code.py脚本编出代码文件game_time_conf.h，最后再被业务文件include调用。请参考demo_main.cpp。

使用的cpp模板如下，思路也是使用std::map将Excel数据转过来，再通过业务代码引用来使用。
```
#include <iostream>
#include <map>
#include <string>

// ################## 以下是自动生成的代码 ##################
// ## 比赛系统  lijunshi2015@163.com

static std::map<int, std::map<std::string, std::string> > time_conf = 
{ {% for v in content.list %}
	{ {{v.id}} , { 
				{"season_name"		, "{{v.season_name}}" } ,
				{"release_start"	, "{{v.release_start}}" } ,
				{"release_end"		, "{{v.release_end}}" } ,
				{"exchange_end"		, "{{v.exchange_end}}" } ,
				{"turnplate_id"		, "{{v.turnplate_id}}" } ,
			}
	},{% endfor %}
};

std::map<int, std::map<std::string, std::string> > get_conf()
{
	return time_conf;
}



// ################## 以上是自动生成的代码 ##################

// ################## 以下是手工编写部分 ##################


```

业务代码调用数据配置文件的实例。

```
#include <iostream>
#include <map>
#include <stdio.h>
#include "auto_codes/game_time_conf.h"

using namespace std;

int main()
{

	std::map<int, std::map<string, string> > my_conf = get_conf();
	printf("%s\n", my_conf[2101]["season_name"].c_str());
	printf("%s\n", my_conf[2102]["release_start"].c_str());
	return 0;
}
```

项目路径：https://github.com/AstarLight/xls2code