---
title: Android Studio中常用设置与快捷键（私人珍藏，Eclipse转AS必看）
date: 2014-08-29 22:28:22
categories: Android Studio
tags: [Android, Android Studio]
---

## 常用设置

1. **Tab不用4个空格**
Code Style->Java->Tabs and Indents->Use tab character
Code Style->General->Use tab character
（例如：版本控制Diff界面按下Tab）

2. **可视化Tab和空格**
Settings->IDE Settings->Editor->Appearance->Show whitespaces

3. **显示代码行数**
Settings->IDE Settings->Editor->Appearance->Show line numbers

4. **修改代码字体大小**
Settings->IDE Settings->Editor->Colors & Fonts ->Font->Save As->改个名字后才能改字体大小

5. **鼠标悬浮显示doc**
Settings->IDE Settings->Editor->Show quick doc on mouse move

6. **空行的Tab和空格被自动干掉**
Settings->IDE Settings->Editor->Other->Strip trailing spaces on Save->None

7. **智能提示首字母匹配**
如String和string，默认情况输入首字母s是不会匹配String的。
Settings->IDE Settings->Editor->Code Completion->Case sensitive completion->设置为None


## 常用快捷键

1. **首先改为Eclipse快捷键**（然后大部分快捷键都会跟Eclipse一致了）
Settings->IDE Settings->Keymap->Keymaps选择Eclipse

2. **像Eclipse那样快速跳出括号**
Keymap->Editor Actions->Complete Current Statement：默认是Ctrl+Shift+Enter；Shift+Enter则不管现在光标在哪个位置,直接新开一行

3. **代码提示列表**（Eclipse中的Content Assist，Alt+/）
Keymap->Main Menu->Code->Completion->Basic：默认是Ctrl+Space

4. **错误修正提示列表**（Eclipse中的Quick Fix，Ctrl+1）
Keymap->Other->Show Intention Action：默认是Alt+Enter

5. **快速Overried方法**
Keymap->Main menu->Code->Override Methods：需要自己设定

6. **Eclipse中的outline**
Keymap->Main Menu->Navigate->File Structure：默认是Ctrl+F3

7. **版本控制中Diff的Next和Prev**
Keymap->Other->Move to the next difference：默认是Ctrl+f7
Keymap->Other->Move to the previous difference：默认是Shift+f7