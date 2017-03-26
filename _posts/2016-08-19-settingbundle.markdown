---
layout:     post
title:      iOS使用setting bundle
author:     慢慢
tags: 		settingbundle iOS
subtitle:  	
category:  blog
---
<!-- Start Writing Below in Markdown -->

测试App的时候，有时候需要对App进行配置，除了自己在应用中增加调试的选项外，还可以使用iOS的偏好设置禁行设置
在项目中新建文件，增加Settings.bundle文件
![这里写图片描述](https://wf96390.github.io/img/settingbundle/1.png)
打开Settings.bundle文件
![这里写图片描述](https://wf96390.github.io/img/settingbundle/2.png)
编辑Root.plist文件
![这里写图片描述](https://wf96390.github.io/img/settingbundle/3.png)
每个Item中可以增加几种值
![这里写图片描述](https://wf96390.github.io/img/settingbundle/4.png)

Group -- 编组。键为PSGroupSpecifier，首选项逻辑编组的标题。
Text Field -- 文本框。键为PSTextFieldSpecifier，可编辑的文本字符串。
Title -- 标题。键为PSTitleValueSpecifier，只读文本字符串。
Toggle Switch -- 开关。键为PSToggleSwitchSpecifier，开关按钮。
Slide -- 滑块。键为PSSliderSpecifier，取值位于特定范围内的滑块。
Multivalue -- 多值。键为PSMultiValueSpecifier，下拉式列表。（上图用的就是这种，其他大同小异）
Child Pane -- 子窗格。键为PSChildPaneSpecifier，子首选项页。

获取值
```
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
NSString *host =  [defaults objectForKey:@“test_host"];
```

修改值
```
[defaults setValue:@"abc" forKey:@"test_host"];
```

相对自己在应用中添加调试模块来说简单很多