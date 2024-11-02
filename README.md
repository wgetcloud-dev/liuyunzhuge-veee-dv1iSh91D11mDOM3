
在前面随笔《[基于wxpython的跨平台桌面应用系统开发](https://github.com/wuhuacong/p/18512657)》介绍了一些关于wxpython开发跨平台桌面应用的总体效果，开发桌面应用，会有很多界面细节需要逐一处理，本篇随笔继续深入该主题，对开发跨平台桌面应用的一些实现细节继续深入研究并总结，介绍程序托盘图标和界面最小化及恢复处理。


### 1、程序托盘图标处理效果


我们知道，一般桌面的应用，如Windows上的Winform应用，MacOS上桌面应用，都会提供一个托盘图标来对程序进行标识和处理，有时候在托盘图标上提供一些常用的菜单操作，如下是本程序在Windows下的实现托盘图标的界面效果。


![](https://img2024.cnblogs.com/blog/8867/202411/8867-20241101101451608-98919290.png)


而同样的程序，在MacOS上也会实现类似的效果，如下试试MacOS上实现的效果。只不过，MacOs自带了一些特定的菜单，因此有点重复的感觉。


![](https://img2024.cnblogs.com/blog/8867/202411/8867-20241101120541806-1752872032.png)


其中托盘图标的一些菜单可以用来显示程序的关于信息，以及一些常见操作。


在Windows系统里面，可以很容易的通过双击托盘图标显示主窗体，或者隐藏主窗体（缩小托盘中）的操作。  


 


### 2、程序托盘功能实现


那么上面托盘图标的处理以及相关菜单的处理，具体在wxpython开发代码中如何实现的呢？


首先我们继承 wx.adv.TaskBarIcon 来自定义托盘图标类，如下所示。




```
import wx
import wx.adv
from wx.adv import TaskBarIcon as TaskBarIcon
import core.core_images as images
from core.event_pub import EventPub


class SystemTaskBarIcon(wx.adv.TaskBarIcon):
    """自定义系统托盘图标处理类"""

    TBMENU_ABOUT = wx.NewIdRef()
    TBMENU_RESTORE = wx.NewIdRef()
    TBMENU_CLOSE = wx.NewIdRef()

    def __init__(self, frame: wx.Frame):

        try:
            TaskBarIcon.__init__(self, wx.adv.TBI_DOCK)  # wx.adv.TBI_CUSTOM_STATUSITEM
            self.frame = frame
            icon = images.appIcon.Icon
            self.SetIcon(icon, "wxPython")
            self.imgidx = 1

            # 绑定事件和菜单
            self.Bind(wx.adv.EVT_TASKBAR_LEFT_DCLICK, self.OnTaskBarActivate)
            self.Bind(wx.EVT_MENU, self.OnTaskBarActivate, id=self.TBMENU_RESTORE)
            self.Bind(wx.EVT_MENU, self.OnTaskBarClose, id=self.TBMENU_CLOSE)
            self.Bind(wx.EVT_MENU, self.OnAbout, id=self.TBMENU_ABOUT)

        except Exception as e:
            print("托盘图标初始化 Error:", e)

    def CreatePopupMenu(self):
        """
        创建托盘图标右键菜单
        """
        aboutIcon = wx.ArtProvider.GetBitmap(wx.ART_QUESTION, wx.ART_OTHER, (16, 16))
        showIcon = wx.ArtProvider.GetBitmap(wx.ART_REPORT_VIEW, wx.ART_OTHER, (16, 16))
        quitIcon = wx.ArtProvider.GetBitmap(wx.ART_QUIT, wx.ART_OTHER, (16, 16))

        menu = wx.Menu()
        aboutitem: wx.MenuItem = menu.Append(self.TBMENU_ABOUT, "关于本程序")
        aboutitem.SetBitmap(aboutIcon)
        showitem: wx.MenuItem = menu.Append(self.TBMENU_RESTORE, "显示/隐藏窗体")
        showitem.SetBitmap(showIcon)
        closeitem: wx.MenuItem = menu.Append(self.TBMENU_CLOSE, "退出", "退出程序")
        closeitem.SetBitmap(quitIcon)

        return menudef OnTaskBarActivate(self, evt):
        if self.frame.IsIconized():
            self.frame.Iconize(False)
        if not self.frame.IsShown():
            self.frame.Show(True)
        else:
            self.frame.Show(False)
            self.frame.Iconize(True)
        self.frame.Raise()

    def OnTaskBarClose(self, evt):
        wx.CallAfter(self.frame.Close)

    def OnTaskBarRemove(self, evt):
        self.RemoveIcon()

    def OnAbout(self, evt):
        wx.MessageBox("This is a wxPython demo program.", "关于", wx.OK | wx.ICON_INFORMATION)
```


 有了上面的自定义子类，我们在主窗体中简单调用初始化一下即可构建托盘图标及菜单了。




```
        # 创建系统托盘图标
        self.tbicon = SystemTaskBarIcon(self)
```


最后在主窗体关闭事件中处理下销毁即可。




```
    def on_close(self, event: wx.CloseEvent):
        """关闭时清理资源"""
         ............

        # 销毁托盘图标
        if self.tbicon is not None:
            self.tbicon.Destroy()

        # 销毁窗口
        self.Destroy()
        event.Skip()
```


如果需要图标进行闪烁的处理，也可以参考下面示例代码。




```
import wx
ID_ICON_TIMER = wx.NewId()

class TaskBarFrame(wx.Frame):
    def __init__(self, parent):
        wx.Frame.__init__(self, parent, style=wx.FRAME_NO_TASKBAR | wx.NO_FULL_REPAINT_ON_RESIZE)
        self.icon_state = False
        self.blink_state = False

        self.tbicon = wx.TaskBarIcon()
        icon = wx.Icon('yellow.ico', wx.BITMAP_TYPE_ICO)
        self.tbicon.SetIcon(icon, '')
        wx.EVT_TASKBAR_LEFT_DCLICK(self.tbicon, self.OnTaskBarLeftDClick)
        wx.EVT_TASKBAR_RIGHT_UP(self.tbicon, self.OnTaskBarRightClick)
        self.Show(True)

    def OnTaskBarLeftDClick(self, evt):
        try:
            self.icontimer.Stop()
        except:
            pass
        if self.icon_state:
            icon = wx.Icon('yellow.ico', wx.BITMAP_TYPE_ICO)
            self.tbicon.SetIcon(icon, 'Yellow')
            self.icon_state = False
        else:
            self.SetIconTimer()
            self.icon_state = True

    def OnTaskBarRightClick(self, evt):
        self.Close(True)
        wx.GetApp().ProcessIdle()

    def SetIconTimer(self):
        self.icontimer = wx.Timer(self, ID_ICON_TIMER)
        wx.EVT_TIMER(self, ID_ICON_TIMER, self.BlinkIcon)
        self.icontimer.Start(1000)

    def BlinkIcon(self, evt):
        if not self.blink_state:
            icon = wx.Icon('red.ico', wx.BITMAP_TYPE_ICO)
            self.tbicon.SetIcon(icon, 'Red')
            self.blink_state = True
        else:
            icon = wx.Icon('black.ico', wx.BITMAP_TYPE_ICO)
            self.tbicon.SetIcon(icon, 'Black')
            self.blink_state = False

app = wx.App(False)
frame = TaskBarFrame(None)
frame.Show(False)
app.MainLoop()
```


 


 本博客参考[MeoMiao 萌喵加速](https://biqumo.org)。转载请注明出处！
