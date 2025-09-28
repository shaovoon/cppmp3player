# Simple C++ DirectShow MP3 Player Class

## Table of Content

* Introduction
* Source Code
* Conclusion
* History

![cppmp3player.png](/images/cppmp3player.png) 

## Introduction

The DirectShow MP3 player class featured in this article, is part of a bigger C++ MFC Karaoke subtitle project until I decided to scrap the source code and rewrite in C# WPF to take advantage of WPF builtin animation. The MP3 source code is available on [Codeplex](http://cppmp3player.codeplex.com/) for quite some time and the [number of downloads](http://cppmp3player.codeplex.com/releases/view/71402) (323) has exceeded my other popular [project](http://outlinetext.codeplex.com/releases/view/61340) (317). This reflects that in the Windows C++ world, there is no good simple MP3 player class. The alternatives are usually either the outdated [Media Control Interface (MCI)](http://en.wikipedia.org/wiki/Media_Control_Interface) or monolithic commercial libraries (with many features) which is an overkill if the programmer just need to simply play a MP3 file.

## Source Code

If you need to just play MP3s in your application (for example, play a short MP3 during the application splash screen), Mp3 class is a no frills C++ MP3/WMA DirectShow player class, for such simple needs. The original code is from [Flipcode](http://www.flipcode.com/)&#39;s contributor, [Alan Kemp](http://www.flipcode.com/archives/Playing_An_MP3_With_DirectShow.shtml). The original code need a bit of tweaking to include the necessary header files and import libraries to get it to compile in Visual Studio 2010. Since this class relies on DirectShow, you need to download the Windows SDK to build it. If you are using Visual Studio 2010, it actually comes with a subset of the Windows SDK which includes the DirectShow lbraries, so you can build this class without downloading anything. You have to call COM&#39;s `CoInitialize` to initialize COM&#39;s runtime before calling the `Load` on mp3 file. And you have to also call `CoUninitialize` at the end of your application, after the `Cleanup` is called. The header file, `Mp3.h` is listed below.

```Cpp
#define WIN32_LEAN_AND_MEAN             // Exclude rarely-used stuff from Windows headers
// Windows Header Files:
#include <windows.h>
#include <mmsystem.h>
#include <strmif.h>
#include <control.h>

#pragma comment(lib, "strmiids.lib")

class Mp3
{
public:
    Mp3();
    ~Mp3();

    bool Load(LPCWSTR filename);
    void Cleanup();

    bool Play();
    bool Pause();
    bool Stop();
	
    // Poll this function with msTimeout = 0, so that it return immediately.
    // If the mp3 finished playing, WaitForCompletion will return true;
    bool WaitForCompletion(long msTimeout, long* EvCode);

    // -10000 is lowest volume and 0 is highest volume, positive value > 0 will fail
    bool SetVolume(long vol);
	
    // -10000 is lowest volume and 0 is highest volume
    long GetVolume();
	
    // Returns the duration in 1/10 millionth of a second,
    // meaning 10,000,000 == 1 second
    // You have to divide the result by 10,000,000 
    // to get the duration in seconds.
    __int64 GetDuration();
	
    // Returns the current playing position
    // in 1/10 millionth of a second,
    // meaning 10,000,000 == 1 second
    // You have to divide the result by 10,000,000 
    // to get the duration in seconds.
    __int64 GetCurrentPosition();

    // Seek to position with pCurrent and pStop
    // bAbsolutePositioning specifies absolute or relative positioning.
    // If pCurrent and pStop have the same value, the player will seek to the position
    // and stop playing. Note: Even if pCurrent and pStop have the same value,
    // avoid putting the same pointer into both of them, meaning put different
    // pointers with the same dereferenced value.
    bool SetPositions(__int64* pCurrent, __int64* pStop, bool bAbsolutePositioning);

private:
    IGraphBuilder *  pigb;
    IMediaControl *  pimc;
    IMediaEventEx *  pimex;
    IBasicAudio * piba;
    IMediaSeeking * pims;
    bool    ready;
    // Duration of the MP3.
    __int64 duration;

};
```

The original class only has the play, pause and stop functionality. Note: after calling `Pause`, you have to call `Play` to resume playing. Since I have a need to loop my music, so I need to know when my MP3 ended, I added the method, `WaitForCompletion` for me to poll periodically whether the playing has ended, to replay it again. Since the original code always played at full volume, I have also added a method, `GetVolume` to get volume and another method, `SetVolume` to adjust volume. __Note__: -10000 is the minimum volume and 0 is the maximum volume. And if you set any positive volume greater than 0, you will receive an error. You can call `GetDuration` and `GetCurrentPosition` to get the duration of the MP3 and the current playing (time) position of the MP3 respectively. These 2 methods return units of 10th millionth of a second(1/10,000,000 of a second): you have to divide by 10,000,000 to get the duration in seconds. The reason I did not return the duration in seconds, is because I found that second unit is too coarse grained to do seeking. The source code implementation of `Mp3.cpp` is listed below.

```Cpp
#include "Mp3.h"
#include <uuids.h>

Mp3::Mp3()
{
    pigb = NULL;
    pimc = NULL;
    pimex = NULL;
    piba = NULL;
    pims = NULL;
    ready = false;
    duration = 0;
}

Mp3::~Mp3()
{
    Cleanup();
}

void Mp3::Cleanup()
{
    if (pimc)
        pimc->Stop();

    if(pigb)
    {
        pigb->Release();
        pigb = NULL;
    }

    if(pimc)
    {
        pimc->Release();
        pimc = NULL;
    }

    if(pimex)
    {
        pimex->Release();
        pimex = NULL;
    }

    if(piba)
    {
        piba->Release();
        piba = NULL;
    }

    if(pims)
    {
        pims->Release();
        pims = NULL;
    }
    ready = false;
}

bool Mp3::Load(LPCWSTR szFile)
{
    Cleanup();
    ready = false;
    if (SUCCEEDED(CoCreateInstance( CLSID_FilterGraph,
        NULL,
        CLSCTX_INPROC_SERVER,
        IID_IGraphBuilder,
        (void **)&this->pigb)))
    {
        pigb->QueryInterface(IID_IMediaControl, (void **)&pimc);
        pigb->QueryInterface(IID_IMediaEventEx, (void **)&pimex);
        pigb->QueryInterface(IID_IBasicAudio, (void**)&piba);
        pigb->QueryInterface(IID_IMediaSeeking, (void**)&pims);

        HRESULT hr = pigb->RenderFile(szFile, NULL);
        if (SUCCEEDED(hr))
        {
            ready = true;
            if(pims)
            {
                pims->SetTimeFormat(&TIME_FORMAT_MEDIA_TIME);
                pims->GetDuration(&duration); // returns 10,000,000 for a second.
                duration = duration;
            }
        }
    }
    return ready;
}

bool Mp3::Play()
{
    if (ready&&pimc)
    {
        HRESULT hr = pimc->Run();
        return SUCCEEDED(hr);
    }
    return false;
}

bool Mp3::Pause()
{
    if (ready&&pimc)
    {
        HRESULT hr = pimc->Pause();
        return SUCCEEDED(hr);
    }
    return false;
}

bool Mp3::Stop()
{
    if (ready&&pimc)
    {
        HRESULT hr = pimc->Stop();
        return SUCCEEDED(hr);
    }
    return false;
}

bool Mp3::WaitForCompletion(long msTimeout, long* EvCode)
{
    if (ready&&pimex)
    {
        HRESULT hr = pimex->WaitForCompletion(msTimeout, EvCode);
        return *EvCode > 0;
    }

    return false;
}

bool Mp3::SetVolume(long vol)
{
    if (ready&&piba)
    {
        HRESULT hr = piba->put_Volume(vol);
        return SUCCEEDED(hr);
    }
    return false;
}

long Mp3::GetVolume()
{
    if (ready&&piba)
    {
        long vol = -1;
        HRESULT hr = piba->get_Volume(&vol);

        if(SUCCEEDED(hr))
            return vol;
    }

    return -1;
}

__int64 Mp3::GetDuration()
{
    return duration;
}

__int64 Mp3::GetCurrentPosition()
{
    if (ready&&pims)
    {
        __int64 curpos = -1;
        HRESULT hr = pims->GetCurrentPosition(&curpos);

        if(SUCCEEDED(hr))
            return curpos;
    }

    return -1;
}

bool Mp3::SetPositions(__int64* pCurrent, __int64* pStop, bool bAbsolutePositioning)
{
    if (ready&&pims)
    {
        DWORD flags = 0;
        if(bAbsolutePositioning)
            flags = AM_SEEKING_AbsolutePositioning | AM_SEEKING_SeekToKeyFrame;
        else
            flags = AM_SEEKING_RelativePositioning | AM_SEEKING_SeekToKeyFrame;

        HRESULT hr = pims->SetPositions(pCurrent, flags, pStop, flags);

        if(SUCCEEDED(hr))
            return true;
    }

    return false;
}
```

Below is an example of how the programmer would use the `Mp3` class in a console application. __Note:__ `Play` is an non-blocking call.

```Cpp
#include "Mp3.h"

void main()
{
    // Initialize COM
    ::CoInitialize(NULL);

    std::wcout<<L"Enter the MP3 path: ";
    std::wstring path;
    getline(wcin, path);

    std::wcout<<path<<std::endl;

    Mp3 mp3;

    int status = 0;
    if(mp3.Load(path.c_str()))
    {
        status = SV_LOADED;
    }
    else // Error
    {
        // ...
    }

    if(mp3.Play())
    {
        status = SV_PLAYING;
    }
    else // Error
    {
        // ...
    }

    // ... after some time

    if(mp3.Stop())
    {
        status = SV_STOPPED;
    }
    else // Error
    {
        // ...
    }

    // Uninitialize COM
    ::CoUninitialize();
}
```

The source code includes a static library project and the DLL project and a demo project, `PlayMp3`, which plays MP3 with a helper class, `CLibMP3DLL`, to load the `LibMP3DLL.dll` at runtime. Usage of `CLibMP3DLL` is similar to `Mp3` class, with additional `LoadDLL` and `UnloadDLL` methods to load/unload dll. Below is the header file of `CLibMP3DLL`.

```Cpp
class CLibMP3DLL
{
public:
    CLibMP3DLL(void);
    ~CLibMP3DLL(void);

    bool LoadDLL(LPCWSTR dll);
    void UnloadDLL();

    bool Load(LPCWSTR filename);
    bool Cleanup();

    bool Play();
    bool Pause();
    bool Stop();
    bool WaitForCompletion(long msTimeout, long* EvCode);

    bool SetVolume(long vol);
    long GetVolume();

    __int64 GetDuration();
    __int64 GetCurrentPosition();

    bool SetPositions(__int64* pCurrent, __int64* pStop, bool bAbsolutePositioning);

private:
    HMODULE m_Mod;
};
```

__Note:__ the source code download also include a C++/CLI wrapper to enable the Mp3 class to be used in .NET application.&nbsp;<strong style="color: rgba(48, 51, 45, 1); font-family: "Segoe UI", "Microsoft Sans Serif", Arial, Geneva, sans-serif">Note__<span style="color: rgba(48, 51, 45, 1); font-family: "Segoe UI", "Microsoft Sans Serif", Arial, Geneva, sans-serif">: If you build the C++ library in x86 (32 bits) or x64 (64 bits) platform, the .NET application has to be in the same platform, not &#39;Any CPU&#39; platform.</span>&nbsp;<strong style="color: rgba(48, 51, 45, 1); font-family: "Segoe UI", "Microsoft Sans Serif", Arial, Geneva, sans-serif">Note__<span style="color: rgba(48, 51, 45, 1); font-family: "Segoe UI", "Microsoft Sans Serif", Arial, Geneva, sans-serif">: You have to install the Advanced Audio Coding (AAC) codec in order to play AAC files (*.aac).</span>&nbsp;

## Conclusion

Though I may have added a few methods to the `Mp3` class, it took me quite a bit effort to get them run correctly. I hope to pass these time-savings to other developers who simply wants to play a MP3 file, minus the hassle.

## History

* __2023-03-11__ : v0.7.3 added `#pragma once` to the `Mp3` header
* __2012-04-26__ : Initial Release

