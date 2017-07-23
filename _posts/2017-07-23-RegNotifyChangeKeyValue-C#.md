---
layout: post
title:  C#监视注册表-------RegNotifyChangeKeyValue
date:   2017-07-23 12:17:27 +0800
categories: Application
tag: C#
---

* content
{:toc}

场景：在C#窗体应用程序中监控某软件安装更新卸载时注册表的变化， 包括键值数据的创建修改删除重命名等。

关键点：使用RegNotifyChangeKeyValue函数提供通知机制

RegNotifyChangeKeyValue
-------------------------------------
该API在指定注册表项属性或内容发生更改时通知调用者。

     LONG WINAPI RegNotifyChangeKeyValue(
      _In_     HKEY   hKey,
     _In_     BOOL   bWatchSubtree,
      _In_     DWORD  dwNotifyFilter,
      _In_opt_ HANDLE hEvent,
      _In_     BOOL   fAsynchronous
      );

 返回值

       Long，零（ERROR_SUCCESS）表示成功。其他任何值都代表一个错误代码
 参数 类型及说明

       hKey，要监视的一个项的句柄，或者指定一个标准项名
       bWatchSubtree，TRUE（非零）表示监视子项以及指定的项
       dwNotifyFilter：
            REG_NOTIFY_CHANGE_NAME 侦测名称的变化，以及侦测注册表的创建和删除事件
            REG_NOTIFY_CHANGE_ATTRIBUTES 侦测属性的变化
            REG_NOTIFY_CHANGE_LAST_SET 侦测上一次修改时间的变化
            REG_NOTIFY_CHANGE_SECURITY 侦测对安全特性的改动
        hEvent Long，一个事件的句柄。如fAsynchronus为False，则这里的设置会被忽略
        fAsynchronus Long，如果为零，那么除非侦测到一个变化，否则函数不会返回。否则这个函数会立即返回，而且在发生变化时触发由hEvent参数指定的一个事件

源码
----------------
```
using Microsoft.Win32;
    public class MonitorWindowsReg
    {
        [DllImport("advapi32.dll", EntryPoint = "RegNotifyChangeKeyValue")]
        private static extern int RegNotifyChangeKeyValue(IntPtr hKey, bool bWatchSubtree, int dwNotifyFilter, int hEvent, bool fAsynchronus);
        [DllImport("advapi32.dll", EntryPoint = "RegOpenKey")]
        private static extern int RegOpenKey(uint hKey, string lpSubKey, ref IntPtr phkResult);
        [DllImport("advapi32.dll", EntryPoint = "RegCloseKey")]
        private static extern int RegCloseKey(IntPtr hKey);
        private static uint HKEY_CLASSES_ROOT = 0x80000000;
        private static uint HKEY_CURRENT_USER = 0x80000001;
        private static uint HKEY_LOCAL_MACHINE = 0x80000002;
        private static uint HKEY_USERS = 0x80000003;
        private static uint HKEY_PERFORMANCE_DATA = 0x80000004;
        private static uint HKEY_CURRENT_CONFIG = 0x80000005;
        private static uint HKEY_DYN_DATA = 0x80000006;
        private static int REG_NOTIFY_CHANGE_NAME = 0x1;
        private static int REG_NOTIFY_CHANGE_ATTRIBUTES = 0x2;
        private static int REG_NOTIFY_CHANGE_LAST_SET = 0x4;
        private static int REG_NOTIFY_CHANGE_SECURITY = 0x8;
        /// <summary>
        /// 打开的注册表句饼
        /// </summary>
        private IntPtr _OpenIntPtr = IntPtr.Zero;
        private RegistryKey _OpenReg;
        private Hashtable _Date = new Hashtable();
        private Hashtable _DateKey = new Hashtable();
        public string _Text;
        /// <summary>
        /// 监视注册表 
        /// </summary>
        /// <param name="MonitorKey">Microsfot.Win32.RegistryKey</param>
        public MonitorWindowsReg(RegistryKey MonitorKey)
        {
            if (MonitorKey == null) throw new Exception("注册表参数不能为NULL");
            _OpenReg = MonitorKey;
            string[] _SubKey = MonitorKey.Name.Split('\\');
            uint _MonitorIntPrt = HKEY_CURRENT_USER;
            switch (_SubKey[0])
            {
                case "HKEY_CLASSES_ROOT":
                    _MonitorIntPrt = HKEY_CLASSES_ROOT;
                    break;
                case "HKEY_CURRENT_USER":
                    _MonitorIntPrt = HKEY_CURRENT_USER;
                    break;
                case "HKEY_LOCAL_MACHINE":
                    _MonitorIntPrt = HKEY_LOCAL_MACHINE;
                    break;
                case "HKEY_USERS":
                    _MonitorIntPrt = HKEY_USERS;
                    break;
                case "HKEY_PERFORMANCE_DATA":
                    _MonitorIntPrt = HKEY_PERFORMANCE_DATA;
                    break;
                case "HKEY_CURRENT_CONFIG":
                    _MonitorIntPrt = HKEY_CURRENT_CONFIG;
                    break;
                case "HKEY_DYN_DATA":
                    _MonitorIntPrt = HKEY_DYN_DATA;
                    break;
                default:
                    break;
            }
            //_Text = MonitorKey.Name;
            _Text = MonitorKey.Name.Remove(0, MonitorKey.Name.IndexOf('\\') + 1);
            RegOpenKey(_MonitorIntPrt, _Text, ref _OpenIntPtr);
        }
        public string GetRegFullName()
        {
            return _OpenReg.Name;
        }
        /// <summary>
        /// 开始监控
        /// </summary>
        public void Star()
        {
            if (_OpenIntPtr == IntPtr.Zero)
            {
                throw new Exception("不能打开的注册项！");
            }
            GetOldRegData();

            System.Threading.Thread _Thread = new System.Threading.Thread(new System.Threading.ThreadStart(Monitor));
            StarMonitor = true;
            _Thread.Start();
        }
        /// <summary>
        /// 更新老的数据表
        /// </summary>
        private void GetOldRegData()
        {
            _Date.Clear();
            _DateKey.Clear();
            string[] OldName = new String[1]; ;
            try { OldName = _OpenReg.GetValueNames(); } catch
            {
                if (OldName == null || OldName[0] == null) return;
            }
            for (int i = 0; i != OldName.Length; i++)
            {
                _Date.Add(OldName[i], _OpenReg.GetValue(OldName[i]));
            }
            string[] OldKey = _OpenReg.GetSubKeyNames();
            for (int i = 0; i != OldKey.Length; i++)
            {
                _DateKey.Add(OldKey[i], "");
            }
        }
        /// <summary>
        /// 停止监控
        /// </summary>
        public void Stop()
        {
            StarMonitor = false;
            RegCloseKey(_OpenIntPtr);
        }
        /// <summary>
        /// 停止标记
        /// </summary>
        private bool StarMonitor = false;
        /// <summary>
        /// 开始监听
        /// </summary>
        public void Monitor()
        {
            while (StarMonitor)
            {
                //System.Threading.Thread.Sleep(1000);
                RegNotifyChangeKeyValue(_OpenIntPtr, false, REG_NOTIFY_CHANGE_NAME + REG_NOTIFY_CHANGE_ATTRIBUTES + REG_NOTIFY_CHANGE_LAST_SET + REG_NOTIFY_CHANGE_SECURITY, 0, false);
                System.Threading.Thread.Sleep(0);
                GetUpdate();
            }
        }
        /// <summary>
        /// 检查数据
        /// </summary>
        private void GetUpdate()
        {
            string[] NewName = new String[1];
            try
            {
                NewName = _OpenReg.GetValueNames();  //获取当前的名称
            }
            catch { }//Rename cause a IO Exception
            foreach (string Key in _Date.Keys)
            {
                for (int i = 0; i != NewName.Length; i++)     //循环比较
                {
                    if (Key == NewName[i])
                    {
                        if (_Date[NewName[i]].ToString() != _OpenReg.GetValue(NewName[i]).ToString())
                        {
                            //修改
                            if (UpReg != null) UpReg(_OpenReg.Name + '\\' + NewName[i], _Date[NewName[i]], _OpenReg.Name + '\\' + NewName[i], _OpenReg.GetValue(NewName[i]));
                        }
                        NewName[i] = "nullbug"; //标记该value已被比较, value name can be ""
                        break;
                    }
                    else if (i == NewName.Length - 1)
                    {
                        if (UpReg != null) UpReg(_OpenReg.Name + '\\' + Key, _Date[Key], "", "");      //删除 
                    }
                }
                if (0 == NewName.Length)
                {
                    if (UpReg != null) UpReg(_OpenReg.Name + '\\' + Key, _Date[Key], "", "");      //删除 
                }
            }
             for (int i = 0; i != NewName.Length; i++)     //循环比较
            {
                if (NewName[i] != "nullbug") //NewName[i] != null && 
                {
                    if (UpReg != null) UpReg("", "", _OpenReg.Name + '\\' + NewName[i], _OpenReg.GetValue(NewName[i]));   //添加
                }
            }

            //subkeys
            string[] NewKey = new String[1];
            try
            {
                NewKey = _OpenReg.GetSubKeyNames();  //获取当前的名称
            }
            catch {
                if (NewKey[0] == null)
                {
                    GetOldRegData();
                    //TODO HOW TO avoid reach here
                    //MessageBox.Show(_OpenReg.Name + " get null reg key!");
                    return;
                }
                else
                {
                    MessageBox.Show(_OpenReg.Name + " :" + NewKey);
                    return;
                }

            }//Rename cause a IO Exception
            foreach (string Key in _DateKey.Keys)
            {
                for (int i = 0; i != NewKey.Length; i++)     //循环比较
                {
                    if (Key == NewKey[i])
                    {
                        NewKey[i] = ""; //标记该value已被比较
                        break;
                    }
                    else if (i == NewKey.Length - 1)
                    {
                        if (UpReg != null) UpReg(_OpenReg.Name + '\\' + Key, "_key_", "", "");      //删除 
                    }
                }
                if (0 == NewKey.Length)
                {
                    if (UpReg != null) UpReg(_OpenReg.Name + '\\' + Key, "_key_", "", "");      //删除 
                }
            }
            for (int i = 0; i != NewKey.Length; i++)     //循环比较
            {
                if (NewKey[i] != "")
                {
                    if (UpReg != null) UpReg("", "", _OpenReg.Name + '\\' + NewKey[i], "_key_");   //添加
                }
            }
            GetOldRegData();
            return;
        }
```
Form1类构造函数中初始化：
```
InitialMonitor("HKEY_CURRENT_USER\\Software\\Microsoft\\Windows\\CurrentVersion\\Test");
```
各Form1类中API定义如下
```
// 通过注册表项路径字符串获得RegistryKey
        RegistryKey GetRegistryKey(string text)
        {
            string[] _SubKey = text.Split('\\');
            Microsoft.Win32.RegistryKey _Key = Microsoft.Win32.Registry.CurrentUser;
            switch (_SubKey[0])
            {
                case "HKEY_CLASSES_ROOT":
                    _Key = Microsoft.Win32.Registry.ClassesRoot;
                    break;
                case "HKEY_CURRENT_USER":
                    _Key = Microsoft.Win32.Registry.CurrentUser;
                    break;
                case "HKEY_LOCAL_MACHINE":
                    _Key = Microsoft.Win32.Registry.LocalMachine;
                    break;
                case "HKEY_USERS":
                    _Key = Microsoft.Win32.Registry.Users;
                    break;
                case "HKEY_CURRENT_CONFIG":
                    _Key = Microsoft.Win32.Registry.CurrentConfig;
                    break;
                default:
                    break;
            }
            _Key = _Key.OpenSubKey(String.Join("\\", _SubKey, 1, _SubKey.Length - 1));
            return _Key;
        }
// 初始化监控某注册表项
        public void InitialMonitor(string text)
        {
            string[] tmpText = text.Split('\\');
            if (tmpText.Length <= 1 || tmpText[1] == null || tmpText[1] == "") return; //no path input
            // Monitor the upper level.
            string upperText = String.Join("\\", tmpText, 0, tmpText.Length - 1);
            if (TS.Count > 100)
            {
                MessageBox.Show("Monitor more than 100 keys!");
                return;
            }
            if (TS[upperText] != null)
            {
                return;
            }
            Microsoft.Win32.RegistryKey _Key = GetRegistryKey(upperText);
            if (_Key == null)
            {
                MessageBox.Show(text + " is not vaild path!");
                return;
            }

            MonitorWindowsReg tmpT = new MonitorWindowsReg(_Key);
// 实例化委托， 调用UpdateReg实际将调用T__UpdateReg
            tmpT.UpReg += new MonitorWindowsReg.UpdataReg(T__UpdateReg);
            tmpT.Star();
            TS.Add(upperText, tmpT);
            // Monitor the current level recur.
            CreateMonitor(text);
        }
// 新增监控注册表项
        public void CreateMonitor(string text)
        {
            string[] tmpText = text.Split('\\');
            //Trace.Assert(tmpText[1] != null);
            if (tmpText.Length <=1|| tmpText[1] == null || tmpText[1] == "") return; //no path input

            if (TS.Count > 100)
            {
                MessageBox.Show("Monitor more than 100 keys!");
                return;
            }
            if (TS[text] != null)
            {
                return;
            }
            Microsoft.Win32.RegistryKey _Key = GetRegistryKey(text);
            if(_Key == null)
            {
                //MessageBox.Show(text + " is not vaild path!");
                return;
            }

            MonitorWindowsReg tmpT = new MonitorWindowsReg(_Key);
            tmpT.UpReg += new MonitorWindowsReg.UpdataReg(T__UpdateReg);
            tmpT.Star();
            TS.Add(text, tmpT);
 
            foreach (string key in _Key.GetSubKeyNames())
            {
                CreateMonitor(text + '\\' + key);
            }
        }
// 删除监控的注册表项
        public void RemoveMonitor(string text)
        {
            if (TS[text] == null)
                return;
            Microsoft.Win32.RegistryKey _Key = GetRegistryKey(text);
            if(_Key!=null)
            foreach (string key in _Key.GetSubKeyNames())
            {
                RemoveMonitor(text + '\\' + key);
            }
            try { ((MonitorWindowsReg)TS[text]).Stop(); } catch { }
           
            TS.Remove(text);
        }
// 其他非UI线程数据传入UI线程处理
        delegate void RegAppendTextCallback(string text);
        private void RegAppendText(string text)
        {
            // InvokeRequired required compares the thread ID of the
            // calling thread to the thread ID of the creating thread.
            // If these threads are different, it returns true.
            if (this.richTextBox_Reg.InvokeRequired)
            {
                RegAppendTextCallback d = new RegAppendTextCallback(RegAppendText);
                this.Invoke(d, new object[] { text });
            }
            else
            {
                this.richTextBox_Reg.AppendText(DateTime.Now.ToLocalTime().ToString()+':'+text);
            }
        }
// 递归遍历并输出该项下面key和value的变化
        void recurAppendNewKeyValue(string NewKey)
        {
            Microsoft.Win32.RegistryKey _Key = GetRegistryKey(NewKey);
            if (_Key == null)
            {
                //MessageBox.Show(text + " is not vaild path!");
                return;
            }
            RegAppendText("Create new key: " + NewKey + '\n');
            foreach (string value in _Key.GetValueNames())
            {
                RegAppendText("Create new value: " + value + " with data " + _Key.GetValue(value).ToString() + '\n');
            }
            foreach (string key in _Key.GetSubKeyNames())
            {
                recurAppendNewKeyValue(NewKey + '\\' + key);
 
            }
        }
//与MonitorWindowsReg.UpdataReg绑定
        void T__UpdateReg(string OldText, object OldValue, string NewText, object NewValue)
        {
            object Old = OldValue;
            object New = NewValue;
            if (Old == null || Old.ToString() == "") Old = "null";
            if (New == null || New.ToString() == "") New = "null";
            string[] nameParts = NewText.Split('\\');
            if(nameParts.Length>2 && nameParts[nameParts.Length-1] == "")
            NewText = String.Join("\\", nameParts, 0, nameParts.Length-1);
            //create value
            if (OldText == "" && NewText != "")
            {
                if (NewValue == null)
                {
                    if(TS[NewText] != null)
                    {
                        RegAppendText("Delete old key: " + NewText + '\n');
                        RemoveMonitor(NewText);
                    }
                    return;
                }
                    
                if(NewValue.ToString() == "_key_")
                {
                    recurAppendNewKeyValue(NewText);
                    CreateMonitor(NewText);
                }
                else
                {
                    RegAppendText("Create new value: " + NewText + " with data " + New.ToString() + '\n');
                }

            }
            else if (OldText != "" && NewText == "")
            {
                if (OldValue == null)
                    return;
                if (OldValue.ToString() == "_key_")
                {
                    if(TS[OldText] != null)
                    {
                        RegAppendText("Delete old key: " + OldText + '\n');
                        RemoveMonitor(OldText);
                    }
                }
                else
                {
                    RegAppendText("Delete old value: " + OldText + '\n');
                }
            }
            else if (OldText != "" && NewText == OldText)
            {
                RegAppendText("Update data of "+NewText+ " from "+ Old.ToString() + " to "+ New.ToString() + '\n');
            }else
            {
                RegAppendText("NewText: " + NewText + "OldText: " + OldText + " OldValue: " + Old.ToString() + " NewValue: " + New.ToString() + '\n');
            }
        }
//停止所有监控
        private void button_StopMon(object sender, EventArgs e)
        {
            if (TS.Count == 0)
            {
                MessageBox.Show("Register text not been monitoring!");
                return;
            }
            foreach (string Key in new List<object>(TS.Keys.Cast<object>()))
                //foreach (string Key in TS.Keys)//error: Collection was modified; enumeration operation may not exec
            {
                RemoveMonitor(Key);
            }

            Trace.Assert(TS.Count == 0);
                for (int i = 0; i < TextBoxes.Count; i++)
                //foreach (TextBox textBox in TextBoxes)
            {
                    ComboBoxes[i].Enabled = true;
                    TextBoxes[i].Enabled = true;
                }
        }
// 开启监控，并记录在hashtable TS中
        private void button_StartMon(object sender, EventArgs e)
        {
            if (TS.Count > 0)
            {
                MessageBox.Show("Register text has already been monitoring!");
                return;
            }
            for (int i = 0; i < TextBoxes.Count; i++)
            //foreach (TextBox textBox in TextBoxes)
            {
                InitialMonitor(ComboBoxes[i].SelectedItem.ToString() + TextBoxes[i].Text);
            }
            if (TS.Count > 0)
            {
                for (int i = 0; i < TextBoxes.Count; i++)
                {
                    ComboBoxes[i].Enabled = false;
                    TextBoxes[i].Enabled = false;
                }
            }else
            {
                MessageBox.Show("No vaild path has been monitoring!");
            }
        }
```
其他
----------------
本文中RegNotifyChangeKeyValue中bWatchSubtree设置为False， 自行递归监控每个子项， 每项建立一个监控。可以把bWatchSubtree设置为True，直接递归监控注册表项下所有子项。

参考
----------
[C#委托(Delegate)](http://www.runoob.com/csharp/csharp-delegate.html)

[RegNotifyChangeKeyValue function](https://msdn.microsoft.com/en-us/library/windows/desktop/ms724892(v=vs.85).aspx)

[How to: Make Thread-Safe Calls to Windows Forms Controls](https://docs.microsoft.com/en-us/dotnet/framework/winforms/controls/how-to-make-thread-safe-calls-to-windows-forms-controls)
