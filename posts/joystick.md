# Windows下的游戏手柄信息读取 | Gamepad | Joystick | C/C++
Windows下可用的方法主要有MultiMedia API和DirectInput两种，另外也可以使用开源库[SDL](https://www.libsdl.org/)。 

## MultiMedia API
可以参考[这篇博客](https://blog.csdn.net/liyuanbhu/article/details/51714045)，特点是简单方便，但是存在一些小问题：  
* `szName`获取到的是`Microsoft PC-joystick driver`而不是原本的手柄名称。
* `szOEMVxD`为空。
* `szRegKey`是`DINPUT.DLL`。
* `joyGetNumDevs`始终返回16个设备。

不知道是Win10的问题还是什么问题，获取不到ID会导致多手柄一起用出问题。  
看起来这个库是把DirectInput封装了一层。这个API使用起来非常简单，不需要复杂配置，直接用`joyGetPosEx`获取状态就可以，读手柄信息则用`joyGetDevCaps`。  
另外一个特点是，这里获取到的轴的数据和手柄给的参数是匹配的，比如我的手柄说是16384级的传感器，读到的数据就是0-16383。  
示例代码如下：  
```C++
#include <iostream>
#include <Windows.h>
#include <mmsystem.h>
using namespace std;

int main()
{
	LPJOYINFOEX info = new JOYINFOEX;
	JOYCAPSA jc;
	joyGetDevCaps(0, &jc, sizeof jc);
	cout << jc.szPname << endl << jc.szOEMVxD << endl << jc.szRegKey << endl;
	while(true)
	{
		joyGetPosEx(0, info);
		cout << info->dwXpos << ", " << info->dwYpos << ", " << info->dwZpos << ", "
			<< info->dwRpos << ", " << info->dwUpos << ", " << info->dwVpos << ", "
			<< info->dwButtons << ", " << info->dwButtonNumber << ", " << info->dwPOV << endl;
		Sleep(500);
	}
}
```

## DirectInput
可以参考[这篇博客](https://www.cnblogs.com/f91og/p/7223281.html)。这个接口就比较复杂了，需要很多枚举设备、打开设备、参数设置的过程，但是可控性也要更强一些。  
使用过程主要分为以下五步：  
1. 创建DirectInput接口和设备
2. 设置数据格式和协作级别
3. 获取设备控制权
4. 获取按键情况并做响应
5. 释放控制权和接口对象

其中1和2是初始化阶段，3-5是需要循环执行的，最好放在子线程中。  
DirectInput有一个问题是，所有轴的值全部映射成了0-65535，而不是设备原本的精度。  
以下是我的一段代码：  
```C++
#include <iostream>
#include <windows.h>
#include <winerror.h>
#include <dinput.h>
#include <dinputd.h>
#pragma comment(lib,"dinput8.lib")
#pragma comment(lib,"dxguid.lib")
using namespace std;

#define SAFE_DELETE(p)  { if(p) { delete (p);     (p)=nullptr; } }
#define SAFE_RELEASE(p) { if(p) { (p)->Release(); (p)=nullptr; } }

BOOL CALLBACK EnumJoysticksCallback(const DIDEVICEINSTANCE* pdidInstance,
	VOID* pContext);

LPDIRECTINPUT8 g_pDI = nullptr;
GUID    guidInstance;

int main()
{
	HRESULT hr;
	if (FAILED(hr = DirectInput8Create(GetModuleHandle(nullptr), DIRECTINPUT_HEADER_VERSION,
		IID_IDirectInput8, (VOID**)&g_pDI, nullptr)))
	{
		cout << "DirectInput8Create failed: " << hex << hr << endl;
		return 0;
	}
    // 只枚举GAMECTRL
    if (FAILED(hr = g_pDI->EnumDevices(DI8DEVCLASS_GAMECTRL,
		EnumJoysticksCallback,
		nullptr, DIEDFL_ATTACHEDONLY)))
	{
		cout << "EnumDevices failed: " << hex << hr << endl;
		return 0;
	}

	LPDIRECTINPUTDEVICE8 device = nullptr;
	if (FAILED(hr = g_pDI->CreateDevice(guidInstance, &device, nullptr)))
	{
		cout << "CreateDevice failed: " << hex << hr << endl;
		return 0;
	}
	if (FAILED(hr = device->SetDataFormat(&c_dfDIJoystick)))
	{
		cout << "SetDataFormat failed: " << hex << hr << endl;
		return 0;
	}
	if (FAILED(hr = device->SetCooperativeLevel(GetConsoleHwnd(), DISCL_BACKGROUND | DISCL_NONEXCLUSIVE)))
	{
		cout << "SetCooperativeLevel failed: " << hex << hr << endl;
		return 0;
	}
	if (FAILED(hr = device->Acquire()))
	{
		cout << "Acquire failed: " << hex << hr << endl;
		return 0;
	}
	DIJOYSTATE state;
	while (true)
	{
		device->Poll();    //轮询设备
		device->Acquire();
		if (SUCCEEDED(hr = device->GetDeviceState(sizeof state, &state)))
		{
			cout << state.lX << ", " << state.lY << ", " << state.rglSlider[0] << ", ";
			cout << int(state.rgdwPOV[0]) << ", ";
			for(auto i = 0;i < 32;++i)
			{
				if (state.rgbButtons[i])
					cout << i << ", ";
			}
			cout << endl;
		}
		Sleep(200);
	}
}

BOOL CALLBACK EnumJoysticksCallback(const DIDEVICEINSTANCE* pdidInstance,
	VOID* pContext)
{
	LPDIRECTINPUTDEVICE8 device = nullptr;
	DIDEVICEINSTANCE info;
	info.dwSize = sizeof(info);
	HRESULT hr;
	//DI8DEVTYPE_JOYSTICK
	cout << "Get device: " << pdidInstance->tszInstanceName << " " << pdidInstance->tszProductName
	<< " Type=" << (pdidInstance->dwDevType & 0xff) << "," << ((pdidInstance->dwDevType >> 8) & 0xff) << endl;
	if((pdidInstance->dwDevType & 0xff) == DI8DEVTYPE_JOYSTICK)
	{
		guidInstance = pdidInstance->guidInstance;
		return DIENUM_STOP;
	}
	return DIENUM_CONTINUE;
}
```

这里还需要获取HWND，MFC程序比较方便，控制台程序可以通过下面函数获取：  
```C++
HWND GetConsoleHwnd(void)
{
#define MY_BUFSIZE 1024 // Buffer size for console window titles.
	HWND hwndFound;         // This is what is returned to the caller.
	char pszNewWindowTitle[MY_BUFSIZE]; // Contains fabricated
										// WindowTitle.
	char pszOldWindowTitle[MY_BUFSIZE]; // Contains original
										// WindowTitle.

	// Fetch current window title.

	GetConsoleTitle(pszOldWindowTitle, MY_BUFSIZE);

	// Format a "unique" NewWindowTitle.

	wsprintf(pszNewWindowTitle, "%d/%d",
		GetTickCount(),
		GetCurrentProcessId());

	// Change current window title.

	SetConsoleTitle(pszNewWindowTitle);

	// Ensure window title has been updated.

	Sleep(40);

	// Look for NewWindowTitle.

	hwndFound = FindWindow(NULL, pszNewWindowTitle);

	// Restore original window title.

	SetConsoleTitle(pszOldWindowTitle);

	return(hwndFound);
}
```

另外还可以通过以下代码设置joystick参数，但是不加这部分也可以，并且我现在也不知道能设置什么。MSDN上的手册[在这](https://docs.microsoft.com/en-us/windows/win32/api/dinputd/ns-dinputd-dijoyconfig)。
```C++
DIJOYCONFIG PreferredJoyCfg = { 0 };
	IDirectInputJoyConfig8* pJoyConfig = nullptr;
	if (FAILED(hr = g_pDI->QueryInterface(IID_IDirectInputJoyConfig8, (void**)&pJoyConfig)))
	{
		cout << "QueryInterface failed: " << hex << hr << endl;
		return 0;
	}
	PreferredJoyCfg.dwSize = sizeof(PreferredJoyCfg);
	if (FAILED(hr = pJoyConfig->GetConfig(0, &PreferredJoyCfg, DIJC_GUIDINSTANCE))) // This function is expected to fail if no joystick is attached
	{
		cout << "GetConfig failed: " << hex << hr << endl;
		return 0; 
	}
	SAFE_RELEASE(pJoyConfig);
```