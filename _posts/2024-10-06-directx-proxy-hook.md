
## Capturing FPS from DirectX Games using Proxy DLL

![A screenshot from [Saints Row 2](https://store.steampowered.com/app/9480/Saints_Row_2/), showing FPS on the top left corner.](https://cdn-images-1.medium.com/max/2470/1*YrS6Qu75BdN0pKeT3Imb7Q.png)

It’s 2024 and I still love gaming. Don’t know, but I can’t let it go no matter how old I get. Since childhood, I have  like played a million games and I only wish to continue doing the same.

In my childhood, I never had a fancy PC, so I had do way with my dusty old Dell laptop. One of things which was the most important bit while playing games was to know how well it ran. This “measure” of performance was the **Frames Per Second** or as we know it **FPS**. The higher the FPS, the better the game ran. Back then, there were numerous softwares which did this, including the OG one, FRAPS. 

I was always wondered how softwares were able to do this. So today, I would be showing you how can we actually capture the FPS in a game using win32 programming ! Don’t worry it sounds very complicated (it was at first), but if you really like learning by doing things on your own, this article is for you.

We are gonna focus on capturing FPS from Games which are based on Microsoft’s DirectX. If you play on a Windows machine, then your game is most likely built using DirectX. (There’s also OpenGL similar to DirectX, but that is out of our scope). If you’re completely new to game programming, I would recommend reading about DirectX and how we can create games using it.

### Pre requisites

 1. Visual Studio: [https://visualstudio.microsoft.com/vs/community/](https://visualstudio.microsoft.com/vs/community/)

 2. Windows OS
 
 3. Basic knowledge how DirectX works 

### DirectX Version

There are numerous versions of DirectX and Games which are built using them. So, how do we capture the frames per second ? The answer to this depends what version of DirectX a games uses. I had some old games lying around in my steam library, so I thought of starting with them.

One of them is the classic sandbox game, Saints Row 2. A quick google search yielded that the game is based on **DirectX9 **(9th version of DirectX)

### Proxy Method

Now that we know that our game uses DX9, we can perhaps somehow exploit the way the game works. Actually, the way most of the games work is, that they require a DirectX DLL (in our case d3d9.dll) which is usually installed in our windows os. DLL stands for Dynamic Link Library which is a type of a shared library that can be used by numerous windows program. Our game, after all, is a windows program at the end of the day and it uses the d3d9.dll to run.

This is our entry point. But, we won’t be modifying this DLL at all. You see, there is a very convenient way we can trick the game into loading our own DLL ! This is called the proxy DLL method, in which a program loads a DLL thinking it is the original one, but in fact it is a proxy one. This is possible because when a program runs, it first looks for the DLL in it’s parent directory and then the system directories. Our original DLL is in systems directory, but we can place a DLL with the same name (say d3d9.dll) in the game’s directory and the game will load this one instead of the original one. Pretty sweet !

### Creating a proxy DLL

Well simply placing a DLL with the same name isn’t enough. When the game loads it, it gonna search for the functions it needs in that DLL and the it’s gonna crash because it isn’t there ! 

![](https://cdn-images-1.medium.com/max/2000/1*q5n1cPd76ekdUK_CgsUBtw.png)

Onto to making our DLL, let’s fire up Visual Studio and choose a Project of type “Windows Dynamic Link Library”. Those are already familiar with windows programming might find it easier to do this, but I will try my best to explain all the steps.
Make sure if you have the following settings and a windows sdk selected (you need this to include directx header files)

![](https://gist.github.com/user-attachments/assets/0023dd7f-50de-47e5-b104-3350f83e8c2a)


When you create the project, you will see a dllmain.cpp file, which is the entry point for us. 

Right now it does nothing, so you will only see the main function

```cpp
BOOL APIENTRY DllMain(HMODULE hModule,
 DWORD  ul_reason_for_call,
 LPVOID lpReserved
)
{
 switch (ul_reason_for_call)
 {
 case DLL_PROCESS_ATTACH:
 case DLL_THREAD_ATTACH:
 case DLL_THREAD_DETACH:
 case DLL_PROCESS_DETACH:
  break;
 }
 return TRUE;
}
```

So in order to start our hook, we will rely on the common functions called from a game when it starts loading the DirectX DLL. This differs per version, and since we are targeting DX9, the first function it would call is the [Direct3DCreate9](https://learn.microsoft.com/en-us/windows/win32/api/d3d9/nf-d3d9-idirect3d9-createdevice) method. This means that our proxy DLL needs to have this function defined as an export. When the games loads up, it's gonna load our proxy DLL and look for this function.

We will need use the directx header files here to create a copy of the `Direct3DCreate9()` function

```cpp
#include "IDirect3D9.h" // for interface and method declarations
```

This file is incredibly useful, as we can use it to make an identical function with identical signature later on.
Onto to our function !

If we take a look at the function signature for `Direct3DCreate9()`, we see that it's type is a pointer to `IDirect3D9` interface and takes only one parameter `SDKVersion` of type `UINT`

```cpp
IDirect3D9 * Direct3DCreate9(
  UINT SDKVersion
);
```
When our proxy DLL loads, we are going to load the original DLL within our DLL and then use it to pass all calls from game to it. To do that, we need some way to read the pointer location of the this function from the original directx9 DLL. Luckily enough, windows provides a very convenient function [GetProcAddress](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getprocaddress) to do just that.

Create a type definition for our load function

```cpp
typedef IDirect3D9* (WINAPI* fn_D3D9Direct3DCreate9)(UINT SDKVersion);
```
Notice, the signature is same as of our original function.

Next, define the load func `fn_D3D9Direct3DCreate9`

```cpp
HMODULE d3d_dll;

fn_D3D9Direct3DCreate9 LoadD3D9AndGetOriginalFuncPointer()
{
	char path[MAX_PATH];
	if (!GetSystemDirectoryA(path, MAX_PATH)) return nullptr;

	strcat_s(path, MAX_PATH * sizeof(char), "\\d3d9.dll");   // first we find and load the dll
	d3d_dll = LoadLibraryA(path);

	if (!d3d_dll)
	{
		MessageBox(NULL, TEXT("Could Not Locate Original D3D9 DLL"), TEXT("Darn"), 0);
		return nullptr;
	}

	return (fn_D3D9Direct3DCreate9)GetProcAddress(d3d_dll, LPCSTR("Direct3DCreate9")); // find original func and return pointer
}
```

We use the literal func name here to retrieve it's address.

### Creating proxy interface and methods

This is the cumbersome part. For the game to correctly work with our proxy DLL, we need to define the same exact methods the original DLL has. This means we need to copy all the methods defined in the `d3d9.h` file where our `IDirect3D9` interface is defined.

Let's create a `IDirect3D9.h` file with our class which will inherit from `IDirect3D9`. We will call our class `HookDirect3D9`

```cpp
#pragma once

#include <d3d9.h>

class HookDirect3D9 : public IDirect3D9 {
public:
	HookDirect3D9(IDirect3D9* originalPointer);
	virtual ~HookDirect3D9(void);

	HRESULT __stdcall QueryInterface(REFIID riid, void** ppvObj);
	ULONG __stdcall AddRef();
	ULONG __stdcall Release();

	/*** IDirect3D9 methods ***/
	HRESULT __stdcall RegisterSoftwareDevice(void* pInitializeFunction);
	UINT __stdcall GetAdapterCount();
	HRESULT __stdcall GetAdapterIdentifier(UINT Adapter, DWORD Flags, D3DADAPTER_IDENTIFIER9* pIdentifier);
	UINT __stdcall GetAdapterModeCount(UINT Adapter, D3DFORMAT Format);
	HRESULT __stdcall EnumAdapterModes(UINT Adapter, D3DFORMAT Format, UINT Mode, D3DDISPLAYMODE* pMode);
	HRESULT __stdcall GetAdapterDisplayMode(UINT Adapter, D3DDISPLAYMODE* pMode);
	HRESULT __stdcall CheckDeviceType(UINT Adapter, D3DDEVTYPE DevType, D3DFORMAT AdapterFormat, D3DFORMAT BackBufferFormat, BOOL bWindowed);
	HRESULT __stdcall CheckDeviceFormat(UINT Adapter, D3DDEVTYPE DeviceType, D3DFORMAT AdapterFormat, DWORD Usage, D3DRESOURCETYPE RType, D3DFORMAT CheckFormat);
	HRESULT __stdcall CheckDeviceMultiSampleType(UINT Adapter, D3DDEVTYPE DeviceType, D3DFORMAT SurfaceFormat, BOOL Windowed, D3DMULTISAMPLE_TYPE MultiSampleType, DWORD* pQualityLevels);
	HRESULT __stdcall CheckDepthStencilMatch(UINT Adapter, D3DDEVTYPE DeviceType, D3DFORMAT AdapterFormat, D3DFORMAT RenderTargetFormat, D3DFORMAT DepthStencilFormat);
	HRESULT __stdcall CheckDeviceFormatConversion(UINT Adapter, D3DDEVTYPE DeviceType, D3DFORMAT SourceFormat, D3DFORMAT TargetFormat);
	HRESULT __stdcall GetDeviceCaps(UINT Adapter, D3DDEVTYPE DeviceType, D3DCAPS9* pCaps);
	HMONITOR __stdcall GetAdapterMonitor(UINT Adapter);
	HRESULT __stdcall CreateDevice(UINT Adapter, D3DDEVTYPE DeviceType, HWND hFocusWindow, DWORD BehaviorFlags, D3DPRESENT_PARAMETERS* pPresentationParameters, IDirect3DDevice9** ppReturnedDeviceInterface);
private:
	IDirect3D9* hookPointer;
};
```

Next, we need to create a definition file, `IDirect3D9.cpp` and we need to add all the function definitions (without the actual code) and point these definitions to the original function. To make it possible, we will use the pointer we will get after we load the original DLL in our load function defined above. 

```cpp
#include <d3d9.h>
#include "IDirect3D9.h"
#include "IDirect3D9Device9.h"


HookDirect3D9::HookDirect3D9(IDirect3D9* originalPointer) {
	hookPointer = originalPointer; // we store a reference to the pointer we get from our load func
}

HookDirect3D9::~HookDirect3D9(void) {}

HRESULT __stdcall HookDirect3D9::QueryInterface(REFIID riid, void** ppvObj)
{
    // any code you want goes here
	return hookPointer->QueryInterface(riid, ppvObj); // we use the hook pointer to call the original func once we are done with our own code
}
```

Notice how we can write any code just before we pass the control to the original func. This is the essence of hooking !

You will need repeat the same for all the funcs defined in the header file, otherwise your game will simply crash since it won't be able to execute the missing funcs.

To actually measure the FPS, there's another func which we will need to hook. Recall the initial func which we intend to use is `Direct3DCreate9`. This func will create the actual device object of type [IDirect3DDevice9](https://learn.microsoft.com/en-us/windows/win32/api/d3d9helper/nn-d3d9helper-idirect3ddevice9). This is the object which is responsible for drawing stuff on the screen and it contains the methods which we require in order to track and then draw the fps on screen. If you want to understand how rendering works, I highly recommend reading this [directx tutorial](http://www.directxtutorial.com/LessonList.aspx?listid=9).

So we create another file `IDirect3DDevice9.h` with all method declarations 

```cpp
#pragma once

#include <d3d9.h>

class HookDirect3D9Device9 : public IDirect3DDevice9 {
public:
	HookDirect3D9Device9(IDirect3DDevice9* originalPointer);
	virtual ~HookDirect3D9Device9(void);

	HRESULT __stdcall QueryInterface(THIS_ REFIID riid, void** ppvObj);
    ... 
    ...
    private:
	IDirect3DDevice9* hookPointer;
};
```

and then `IDirect3DDevice9.cpp` with definitions pointing to the original func we get from `IDirect3D9.cpp` (we will get to this part next)

```cpp
HookDirect3D9Device9::HookDirect3D9Device9(IDirect3DDevice9* pOriginal)
{
	MessageBox(NULL, TEXT("Hooked Direct3DCreate9"), TEXT("Success"), 0); // just for debug
    hookPointer = pOriginal; // store the pointer to original object
}

HookDirect3D9Device9::~HookDirect3D9Device9(void)
{
}

HRESULT HookDirect3D9Device9::QueryInterface(REFIID riid, void** ppvObj)
{
    return hookPointer->QueryInterface(riid, ppvObj);
}
```

Now with our hook files defined, we can come back to our `dllmain.cpp` file. We will now define an identical func `Direct3DCreate9` which we will later export as well

```cpp
IDirect3D9* WINAPI Direct3DCreate9(UINT SDKVersion)
{
	fn_D3D9Direct3DCreate9 D3D9Direct3DCreate9_Orig = LoadD3D9AndGetOriginalFuncPointer(); // load original dx9 dll
	
	IDirect3D9* originalIDirect3D9 = D3D9Direct3DCreate9_Orig(SDKVersion); // create the original direct3d9 object

	HookDirect3D9* myIDirect3D9 = new HookDirect3D9(originalIDirect3D9); // pass the object to our hook

	return myIDirect3D9; // return pointer to our hooked object
}
```
About the `WINAPI` keyword, it's just an alias for `__stdcall` which is a way to indicate that this is a Win32 func.

As you would recall, once the game calls this `Direct3DCreate9` func to create an object, it's gonna call the `CreateDevice` func next.
Return back to `IDirect3D9.cpp` and modify the `CreateDevice` func

```cpp
HRESULT __stdcall HookDirect3D9::CreateDevice(UINT Adapter, D3DDEVTYPE DeviceType, HWND hFocusWindow, DWORD BehaviorFlags, D3DPRESENT_PARAMETERS* pPresentationParameters, IDirect3DDevice9** ppReturnedDeviceInterface)
{	
	HRESULT hres = hookPointer->CreateDevice(Adapter, DeviceType, hFocusWindow, BehaviorFlags, pPresentationParameters, ppReturnedDeviceInterface); // using the original dll pointer to create a device

	HookDirect3D9Device9* hookIDirect3DDevice9 = new HookDirect3D9Device9(*ppReturnedDeviceInterface); // pass the device to our Direct3DDevice9 hook

	*ppReturnedDeviceInterface = hookIDirect3DDevice9; // set the device to the hooked device object and return

	return hres;
}
```

At this point, all our hooks are set. This means if we will be able to intercept all dx9 game calls if we wish to. We have hooked the creation of direct3d9 object as well as the creation of th direct3d9 device. That's all we will need to start inserting our custom code to count the no of frames our game is displaying.

### Exporting functions

In order for windows to recognize our proxy DLL function `Direct3DCreate9`, we will need to export it using the `exports.def` file. Add a a new file named `exports.def`. This tells windows that our DLL has one primary exported function. This is how the game will also know which function to call when it loads our proxy DLL. If you want to know more about this method, you can read it [here](https://learn.microsoft.com/en-us/cpp/build/exporting-from-a-dll-using-def-files?view=msvc-170).

```cpp
LIBRARY	"d3d9"

EXPORTS
        Direct3DCreate9 @1

```

We are ready to intercept ! Hit the compile button on your visual studio code and copy the dll created in your game's directory. Now start the game and voila, you should see a message box "Hooked Direct3DCreate9" which we placed in our code in `IDirect3DDevice9.cpp` file. If you press okay, the game will proceed to run normally. 

*Important* If your game crashes, that means you might have missed copying methods from the original dx9 header files.
I will provide a link to my code repo so that you can reference that.

### Hooking into EndScene

There are 3 major functions in dx9 which a game uses to render images on screen. These are 
- BeginScene()
- Present()
- EndScene()  // our hook 

While the details about the inner working of these functions are out of scope for this article, I would recommend going through the dx9 tutorial I shared before to understand the methodology.

#### Why EndScene() ?
That is because it is called on every time the game draws a frame. It means if your Game is running at 60 FPS, then this function is being called 60 times per second. If we were to add our counter here, then wew get exactly the amount of fps our game has and consequently, display the same.

For displaying an overlay on the game's screen, we will use the draw func dx9 (actually D3dx9core) has. D3dx9core is a set of tools available separately along with dx9 to help create and draw fonts. This package is not included in the windows sdk, so we need to install it separately using the package manager menu. Right click on you project -> packages and install the following package

![](https://gist.github.com/user-attachments/assets/9a526a9f-4eed-4080-8b40-9fc42b788c45)

### Counting FPS

We are close now to our end goal (phew). We will initialize our fonts first to use them later to display the text over the game

Go to `IDirect3D9Device9` and initialize the fonts in the constructor

```cpp
LPD3DXFONT	m_font;
D3DCOLOR fontColor;
RECT rct;

HookDirect3D9Device9::HookDirect3D9Device9(IDirect3DDevice9* pOriginal)
{
    hookPointer = pOriginal; // store the pointer to original object

    // Define fonts and color
    m_font = NULL;
    D3DXCreateFont(hookPointer, 17, 0, FW_BOLD, 0, FALSE, DEFAULT_CHARSET, OUT_DEFAULT_PRECIS, ANTIALIASED_QUALITY, DEFAULT_PITCH | FF_DONTCARE, TEXT("Arial"), &m_font);
    fontColor = D3DCOLOR_ARGB(255, 255, 255, 0);

    rct.left = 40;
    rct.right = 1680;
    rct.top = 40;
    rct.bottom = rct.top + 200;
}
```

To count the no of times the endscene func is called, we can use the chrono package cpp provides. Add a few more global variables in the above file.

```cpp
INT frames = 0;
INT lastFrame = 0;
auto ts = std::chrono::system_clock::now() + std::chrono::seconds(1);
auto twoSeconds = std::chrono::system_clock::now() + std::chrono::seconds(2);
```

Next, we place our code in the `EndScene()` func to count how many times this func has been called. Furthermore, we will use this count to display our counter using the fonts we defined in the previous steps. 

Note: We only update the fonts once per 2 seconds, since updating it every sec will make the counter change too fast.

```cpp

HRESULT HookDirect3D9Device9::EndScene(void)
{
    // if we reached 1 sec, that means we reset our counter and display the sum of the counted frames
    if (ts <= std::chrono::system_clock::now()) {
        lastFrame = frames;
        frames = 0;
        ts += std::chrono::seconds(1);

        if (twoSeconds <= std::chrono::system_clock::now()) {
            m_font->DrawTextA(NULL, std::to_string(lastFrame).c_str(), -1, &rct, 0, fontColor);
            twoSeconds += std::chrono::seconds(2);
        }
    }
    else {
        frames++;
        m_font->DrawTextA(NULL, std::to_string(lastFrame).c_str(), -1, &rct, 0, fontColor);
    }

    return(hookPointer->EndScene());
}
```

Compile the project again and this time, it should produce 3 DLLs, namely our proxy DLL and the helper DLLs `d3dx9d_43.dll` `D3DCompiler_43.dll`

Copy and paste all three of them in the game directory and run the game. Now the bright yellow fps counter should pop up in the top left corner ! 

Thanks for reading. 
Here's the link to the entire code: [https://github.com/SpartanX1/directx-proxy](https://github.com/SpartanX1/directx-proxy)
