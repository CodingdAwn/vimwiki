## windows下使用windows terminal + wsl
主要为了实现terminal透明

#### 遇到的问题
透明度现在只支持 acrylic透明 即毛玻璃特效  
但是我不想用啊 我有wallpaper engine啊  
使用autohotkey 加载快捷键

- 以下为透明方式
```
SendMode Input  ; Recommended for new scripts due to its superior speed and reliability.
SetWorkingDir %A_ScriptDir%  ; Ensures a consistent starting directory.

TLevel = 180

#^Esc::
	WinGet, CurrentTLevel, Transparent, A
	If (CurrentTLevel = OFF) {
		WinSet, Transparent, %TLevel%, A
	} Else {
		WinSet, Transparent, OFF, A
	}
return

SetTransparency:
	WinGet, CurrentTLevel, Transparent, A
	WinSet, Transparent, %TLevel%, A
return


#^=::
	TLevel += 5
	If TLevel >= 255
	{
		TLevel = 255
	}

	Gosub, SetTransparency
return

#^-::
	TLevel -= 10
	If TLevel <= 0
	{
		TLevel = 0
	}

	Gosub, SetTransparency
return
```

- 以下设置快捷键 google
```
; 首先，在最最开始的地方，放这样的定义（定义 group）
groupadd , CHROME              , ahk_exe chrome.exe
return

; 然后，就可以用 Win+1来激活Firefox（甚至是其不同的session／窗口）。
#1::
IfWinExist ahk_exe chrome.exe
    groupactivate, CHROME, r
else
	run "C:\Program Files\Google\Chrome\Application\chrome.exe"
return
```

- 以下设置快捷键 terminal
```
groupadd , TERMINAL              , ahk_exe WindowsTerminal.exe
return

; 然后，就可以用 Win+2来激活Firefox（甚至是其不同的session／窗口）。
#2::
IfWinExist ahk_exe WindowsTerminal.exe
    groupactivate, TERMINAL, r
else
	;run "C:\Program Files\Google\Chrome\Application\chrome.exe"
return
```
