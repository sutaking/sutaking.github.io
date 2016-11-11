---
layout: post
title: 在国内怎么玩AngularJS的E2E测试protractor
date: 2015-11-02 15:32:24.000000000 +09:00
---

天朝码农，网络环境不好，自己公司防火墙网络限制更多，导致做Web开发问题更多。 这个protractor的测试画面很美，但是想吃上一口，那是非常的麻烦。下面先放上guide，如果你能全数通过，按照里面的命令走完，你直接可以闪了，下面的内容对你毫无意义，如果你跟我一样，从第一步npm install就开始报err，那么恭喜你，找对组织了。

E2E测试简单点说就是用jasmine写一个测试程序，用来自动让浏览器运行一下页面的操作。内部机制是，通过访问web的server以及有支持运行浏览器的selenium的webdriver。

这里推荐angular-phonecat项目作为练手，十分推荐研究下package.json、bower.json等配置文件，掌握下angular项目的基本构架。

 `git clone https://github.com/angular/angular-phonecat.git`

protractor的setup命令
`
 npm install & bower install
 npm install -g protractor
 npm run update-webdriver
 npm start //运行server
 npm run protractor //另起一个终端run
`
我很负责的说，上面每一个需要download包的命令，我都遇到err走不下去了，真心悲剧！

* 问题一：npm install -g protractor `python err`
  
 Windows首次运行npm install的时候会提示需要python依赖，装个2.7的版本就行，不要忘记配置环境变量path。<br>
![image](http://7xng4l.com1.z0.glb.clouddn.com/njfeng_151012_npm_install_about_python_err.jpg)<br>

* 问题二：npm install err

 这应该是最普遍的问题了，原因是gwf造成的，所以你懂的，解决方案很简单，要么让你的终端22端口也可以fan墙，要么用修改掉npm的package搜索节点。亲测可用。详细内容可以看[这篇文章](https://cnodejs.org/topic/4f9904f9407edba21468f31e)

 `npm --registry https://registry.npm.taobao.org info underscore `

*  问题三：bower install ECMDERR

 如果你的内网防火墙不做什么限制的话，应该不会遇到这个问题，但是悲剧的是我司就是这么变态，有的git端口就是不给你用，那就修改下git的访问方式。
![image](http://7xng4l.com1.z0.glb.clouddn.com/njfeng_151012_ecmderr.jpg)<br>
 `
 //The solution without changing the firewall:
 git config --global url."https://".insteadOf git://
`

 然后就搞定了(bower install 多执行几次，因为网络环境不是那么的好)
![image](http://7xng4l.com1.z0.glb.clouddn.com/njfeng_151012_result.jpg)<br>

*  问题四：npm run update-webdriver

 这个命令跟`webdriver-manager update`是一个意思，由于文件存放在Google上，在国内你肯定没办法执行成功的。
![image](http://7xng4l.com1.z0.glb.clouddn.com/njfeng_151012_webdriver.jpg)<br>
<br>
我就直接开了ss，下载了，放在protractor的目录里面,路径如下：
我把文件做了[百度网盘分享](http://pan.baidu.com/s/1bnv4zPP)
 `
 ～\protractor\selenium\
 chromedriver.exe
 selenium-server-standalone-2.47.1.jar`

现在可以运行`npm run protractor`了，记得一定要先运行`npm start`起server

*  问题四：Mac OS 小伙伴的protractor会运行不了
![image](http://7xng4l.com1.z0.glb.clouddn.com/npm%20install%20darwin%20err.png)<br>
这个问题跟protractor的配置有关系，找到`protractor-conf.js`去掉`chromeOnly: true`这句，再补上`directConnect: true`即可。

* 小结

没有gfw根本不要这么复杂的过程，不过想到fan墙是天朝码农的基本技能，也就释然了吧。
 
* END
