# car
rasperryPi using javaScript
![image](https://github.com/zexiplus/car/blob/master/WP_20160624_003.jpg?raw=true)
## **前言** ##


----------
前些天入手了**树莓派3**代，为了能在**raspberry Pi 上用上node.js**,克服了各种坑，完成了一些配置，希望新手共勉。

## **项目概要** ##


----------
本项目可以实现远程控制小车实现前进后退及转弯，后续项目会实现许多传感器数据回送，我们先来看一下怎么实现远程控制。

## **图片** ##


----------


## **配置树莓派** ##



----------
### **更新系统** ##


----------
为了能直接下载node，建议更新系统
```
sudo apt-get update
sudo apt-get upgrade
```

###**安装node**  ##
用最新的官方镜像raspbian自带node，省去下面步骤

----------

```
sudo apt-get install node
```


###**安装npm包管理工具** ##


----------


```
sudo apt-get install npm
```
### **安装gpio-admin** ##


----------

因为js本身并不能直接操控硬件及io口，所以要安装系统层面的库，这里使用gpio-admin，这里贴上官方github：https://github.com/rakeshpai/pi-gpio

这个步骤**非常关键**，其中有几处坑，按照网上的教程（都是几年前的）会出现错误，我已进行适当更改

```
                      
mkdir gpio            //自行创建文件夹

cd gpio               //打开该文件夹

git clone git://github.com/quick2wire/quick2wire-gpio-admin.git           
                    
                      //用git工具下载gpio源码

cd quick2wire-gpio-admin

make                  //编译

```
这个步骤还没有完，因为这个库最后一次更新是四年以前了，那时还没有树莓派三代，所以直接在 make后就用sudo make install 命令，之后的程序运行时，gpio-admin会报错：错误提示为：error : Error when trying to open pin 16 gpio-admin : could not flush data to /sys/class/gpio/export : Device or resource busy 
在这里要修改原C文件中的内容再安装。步骤如下：

```
cd quick2wire-gpio-admin      //打开刚刚下载的文件夹
ls                            //列出文件夹内容
cd src                        //打开src目录
vim gpio-admin.c              //修改原文件

```
找到这一行：

```
int size = snprintf(path, PATH_MAX, "/sys/devices/virtual/gpio/gpio%u/%s", pin, filename);
```
用这一行代替：

```
int size = snprintf(path, PATH_MAX, GPIO_CLASS_PATH "gpio%u/%s", pin, filename);
```
退出并保存，然后安装。

```
cd ..              //退回quick2wire-gpio-admin目录
sudo make install  //安装
```
大功告成，来看代码实现。

## **代码实现** ##


----------
### **一些思路** ##


----------
本次实验是基于websocket实时通讯并配合着前端的键盘事件完成的，所以要安装socket.io，jquery。但是在我的项目中还引入了其他的一些前端框架比如angularjs 和bootstrap这些是为了以后的项目做的所以大家如果只是为了实现本次实验可以不引入这么多库。本实验也可以通过ajax轮询方式实现，但是websocket更为方便。
先看一下依赖

```
{
  "name": "car",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "start": "node ./bin/www"
  },
  "dependencies": {
    "body-parser": "~1.13.2",
    "cookie-parser": "~1.3.5",
    "debug": "~2.2.0",
    "express": "~4.13.1",
    "jade": "~1.11.0",
    "morgan": "~1.6.1",
    "serve-favicon": "~2.3.0",
    "mongoose":"*",
    "socket.io":"*",
    "pi-gpio":"*"
  }
}
```
**其中pi-gpio 和 socket.io是必要项**。

### **服务器端逻辑** ##


----------


```
var express = require('express');
var path = require('path');
var favicon = require('serve-favicon');
var logger = require('morgan');
var cookieParser = require('cookie-parser');
var bodyParser = require('body-parser');
var gpio= require('pi-gpio');

var routes = require('./routes/index');
var users = require('./routes/users');







var errhan = function(err){if(err){console.log(err);}}
gpio.open(12,'output',errhan);
gpio.open(11,'output',errhan);
gpio.open(13,'output',errhan);
gpio.open(15,'output',errhan);

var app = express();
var apt = require('http').Server(app);
var io  = require('socket.io').listen(apt);
io.on('connection',function(socket){
  socket.on('up',function(){
      gpio.write(12,1,errhan);
      gpio.write(11,0,errhan);
      gpio.write(13,1,errhan);
      gpio.write(15,0,errhan);
  });
  socket.on('loseup',function(){
      gpio.write(12,0,errhan);
      gpio.write(11,0,errhan);
      gpio.write(13,0,errhan);
      gpio.write(15,0,errhan);
  });
  socket.on('down',function(){
      gpio.write(12,0,errhan);
      gpio.write(11,1,errhan);
      gpio.write(13,0,errhan);
      gpio.write(15,1,errhan);
  });
  socket.on('losedown',function(){
      gpio.write(12,0,errhan);
      gpio.write(11,0,errhan);
      gpio.write(13,0,errhan);
      gpio.write(15,0,errhan);
  });
  socket.on('left',function(){
      gpio.write(12,0,errhan);
      gpio.write(11,0,errhan);
      gpio.write(13,1,errhan);
      gpio.write(15,0,errhan);
  });
  socket.on('loseleft',function(){
      gpio.write(12,0,errhan);
      gpio.write(11,0,errhan);
      gpio.write(13,0,errhan);
      gpio.write(15,0,errhan);
  });
  socket.on('right',function(){
      gpio.write(12,1,errhan);
      gpio.write(11,0,errhan);
      gpio.write(13,0,errhan);
      gpio.write(15,0,errhan);
  });
  socket.on('loseright',function(){
      gpio.write(12,0,errhan);
      gpio.write(11,0,errhan);
      gpio.write(13,0,errhan);
      gpio.write(15,0,errhan);
  });

  socket.on('disconnect',function(){
      console.log('the connection is closed')
  });
})
```

这段代码只是核心代码，是app.js中的一部分，分析一下代码，作用是引入pi-gpio库和websocket库，并创建socketio实时连接监听服务器，分别开启11，12，13，15gpio引脚，设置为输出模式，然后监听前端事件并输出高低电平控制小车。

### **前端逻辑** ##


----------

```
(function($){
	$(function(){

	var up 		=	$('.up'),
		down 	=	$('.down'),
		left 	= 	$('.left'),
		right 	=	$('.right'),
		socket 	= 	io.connect(); 
		socket.on('connection',function(){console.log('the connection is on')});

	$('.up').on('mousedown',function(e){
		socket.emit('up');
	});
	$('.down').on('mousedown',function(e){
		socket.emit('down');
	});
	$('.left').on('mousedown',function(e){
		socket.emit('left');
	});
	$('.right').on('mousedown',function(e){
		socket.emit('right');
	});

	$('.up').on('mouseup',function(e){
		socket.emit('loseup');
	});
	$('.down').on('mouseup',function(e){
		socket.emit('losedown');
	});
	$('.left').on('mouseup',function(e){
		socket.emit('loseleft');
	});
	$('.right').on('mouseup',function(e){
		socket.emit('loseright');
	});


	$(document).on('keydown',function(e){
		console.log(e.keyCode);
		if(e.keyCode == 38){
			socket.emit('up');
		}
		if(e.keyCode == 40){
			socket.emit('down');
		}
		if(e.keyCode == 37){
			socket.emit('left');
		}
		if(e.keyCode == 39){
			socket.emit('right');
		}

	});
	$(document).on('keyup',function(e){
		if(e.keyCode == 38){
			socket.emit('loseup');
		}
		if(e.keyCode == 40){
			socket.emit('losedown');
		}
		if(e.keyCode == 37){
			socket.emit('loseleft');
		}
		if(e.keyCode == 39){
			socket.emit('loseright');
		}
	});

	});
})(jQuery);
```
前端监听键盘和点击事件，手动出发自定义事件。

前端的UI就不给大家看了，所有代码会放在我的github中。

## **跑起来吧** ##


----------
在跑代码之前不妨用面包板和led来做个测试以防烧坏树莓派，因为我之前开发单片机已经烧坏了一块，树莓派这东西有些小贵所以要谨慎。

在树莓派上安装tightvncserver，以便控制树莓派

```
sudo apt-get install tightvncserver
```
在自己的pc上安装vnc和ftp工具，自行百度
安装完后把项目文件夹一股脑传到树莓派，把node_modules目录下的文件夹都删了，重新npm install 下载。

```
cd car       //打开项目目录
npm install  //下载包
```

最后要给项目文件夹**赋予可执行权限**：

```
sudo chmod -R +x car // 这里car为项目文件夹
```

把树莓派接到小车上
运程控制下

```
sudo node app
```

打开电脑浏览器输入 树莓派ip地址：3000端口，我的是：192.168.1.107：3000

就可以用键盘或鼠标控制小车车了。

## **跳出局域网限制** ##

如果仅仅是这样就失去了可把玩性，所以把这个跑在树莓派上的nodejs服务器投射到公网就可以让远方的朋友千里之外控制树莓派小车。
所以要在树莓派上下载 ngrok 神器，这个神器可以把局域网投射到公网。下载方法自行百度。

```
ngrok http 3000          
```
登陆ngrok给出的公网地址就可以在互联网上的每一个角落控制车车。

## **至此** ##


----------
简单的遥控功能就开发完毕了，后面的文章会介绍用node把小车传感器上的数据读取到前端界面，今天就到这里。

 **——作者 小西西**
 **转载请注明**
