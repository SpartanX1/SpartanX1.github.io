
## Capturing FPS from DirectX Games using Proxy DLL

![A screenshot from [Saints Row 2](https://store.steampowered.com/app/9480/Saints_Row_2/), showing FPS on the top left corner.](https://cdn-images-1.medium.com/max/2470/1*YrS6Qu75BdN0pKeT3Imb7Q.png)

It’s 2024 and I still love gaming. Don’t know, but I can’t let it go no matter how old I get. Since childhood, I have  like played a million games and I only wish to continue doing the same.

In my childhood, I never had a fancy PC, so I had do way with my dusty old Dell laptop. One of things which was the most important bit while playing games was to know how well it ran. This “measure” of performance was the **Frames Per Second** or as we know it **FPS**. The higher the FPS, the better the game ran. Back then, there were numerous softwares which did this, including the OG one, FRAPS. 

I was always wondered how softwares were able to do this. So today, I would be showing you how can we actually capture the FPS in a game using win32 programming ! Don’t worry it sounds very complicated (it was at first), but if you really like learning by doing things on your own, this article is for you.

We are gonna focus on capturing FPS from Games which are based on Microsoft’s DirectX. If you play on a Windows machine, then your game is most likely built using DirectX. (There’s also OpenGL similar to DirectX, but that is out of our scope). If you’re completely new to game programming, I would recommend reading about DirectX and how we can create games using it.

### Pre requisites

 1. Visual Studio: [https://visualstudio.microsoft.com/vs/community/](https://visualstudio.microsoft.com/vs/community/)

 2. Windows OS 

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

This is the cumbersome part. For the game to correctly work with our proxy DLL, we need to define the same exact methods the original DLL has. This means we need to copy all the methods defined in 
