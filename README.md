# dwm-overlay

## DWM Overlay Demonstration for Windows 25H2
This repository demonstrates how to hook into the dwm.exe process to draw GUI elements directly onto the desktop screen buffer, and implements a screenshotting application inside of DWM. Included in this repository is the module for the hook DLL (client), a modified version of my DLL manual mapper (https://github.com/chaosium43/manual-mapper) to inject the hook (injector), and a dumper program to allow for fast updates to DWM offsets.
![Browser Screenshot With Export Window Open](./screenshots/capture3.png)

## Running The Application
If you simply want to try the overlay out for yourself, there are pre-built binaries included in this repository in ./binaries which you can run provided you are on Windows version 26200.7429.<br>
If you want to build the overlay from scratch, you will need the following prerequisites:
- Being on Windows for x86
- CMake (Version 3.21 or newer)
- Visual Studio 2022 Build Tools (Either download Visual Studio or download the toolchain separately)
If all prerequisites are satisfied, you may build this repository by running the following commands:

```
mkdir build
cd build
cmake -G "Visual Studio 17 2022" .. --fresh
cmake --build . --config=Release"
```

After building the repository, run injector.exe as administrator and the hook should inject itself into DWM. Note that building for release is recommended as it will reduce the dependencies and the binary size for the hook DLL. Also note that building will fail should you try to use another C++ compiler.

## Updating Offsets
As more Windows updates are released, the DWM binaries will change and the offsets currently in this repository may become obselete. To get the updated offsets, run dumper.exe and an updated version of offsets.cpp should be generated. Copy the contents of the generated offsets.cpp to the offsets.cpp in the client directory and rebuild.<br>

⚠️⚠️ PLEASE NOTE THAT THE DUMPER IS STILL WORK IN PROGRESS AND DOES NOT DUMP ALL OFFSETS. OffsetD3D11Device AND OffsetOverlayMonitorTarget ARE FIXED TO A SET VALUE AND MAY CHANGE ARBITRARILY IN THE FUTURE. ⚠️⚠️

## Manually Obtaining Offsets
To manually obtain all offsets, open your local version of dwmcore.dll into the reverse engineering software of your choice and rebase the image to 0x0. Then, follow these steps to get each offset.
### VTable Offsets
There are 4 offsets in the DWM hook that are VTable offsets. They are as follows:
- OffsetGetDevice: Look for COverlaySwapChain::GetDevice, then search through xrefs until you find the vftable for either CLegacySwapChain's IDeviceResource or CDDisplaySwapChain IDeviceResource.
- OffsetGetPhysicalBackBuffer: Look for CDDisplaySwapChain::GetBackBuffer, then there should be an xref located in the CDDisplaySwapChain IDeviceResource vftable.
- OffsetGetD3D11Resource: Look for CDDisplaySwapChainBuffer::GetD3D11Resource, there should be an xref located in the CDDisplaySwapChainBuffer vftable.
- OffsetIsPrimaryMonitor: Look for CDDisplayRenderTarget::IsPrimaryMonitor, there should be an xref located in the CDDisplayRenderTarget IPixelFormat vftable.
Once you know where all the functions live inside of their respective vftables and the vftable bases, simply subtract the address of the function entry in the vftable by the vftable base to get VTable offsets.
### Static Function Offsets
There are 6 static function offsets. They are as follows:
- OffsetPresent: Maps to COverlayContext::Present
- OffsetPresentNeeded1: Maps to CDDisplayRenderTarget::PresentNeeded
- OffsetPresentNeeded2: Maps to CLegacyRenderTarget::PresentNeeded
- OffsetScheduleCompositionPass: Maps to ScheduleCompositionPass
- OffsetIsOverlayPrevented: Maps to CGlobalCompositionSurfaceInfo::IsOverlayPrevented
### Struct Offsets
There are 3 struct offsets. They are as follows:
- OffsetD3D11Device: Decompile CD3DDevice::Init. Look for a line where a function pointer is called with the GUID 8ffde202-a0e7-45df-9e01-e837801b5ea0. This is a QueryInterface call which will load an ID3DDevice1 pointer into the CD3DDevice struct at a set location specified in the last parameter of the QueryInterface call (this + offset).
- OffsetOverlayMonitorTarget: Decompile COverlayContext::COverlayContext. The second parameter is an IOverlayMonitorTarget. Pay attention where that parameter gets written to. That's the offset to the monitor target for a COverlayContext aka OffsetOverlayMonitorTarget.
- OffsetForceDirtyRendering: Maps to CCommonRegistryData::ForceFullDirtyRendering