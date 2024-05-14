---
layout:      post
title:       Interoping Graphics APIs
date:        2024-05-14 10:00:00
author:      Kirill Evdakimov
summary:     Befriending CUDA with DirectX and Vulkan
categories:  Engine
category_id: 123542
thumbnail:  
tags:
 - Engine
 - DirectX
 - DirectX 11
 - DirectX 12
 - Vulkan
 - CUDA
 - C++
---

In the old days of OpenGL and Direct3D 11 to extend the engine externally said engine had to expose just a few graphical structures to the game programmer such as:
``ID3D11Device``, ``ID3D11DeviceContext``, ``ID3D11Texture1D/2D/3D`` and corresponding access views for DX11 and Texture Index for OpenGL.
But everything changed when we got new APIs to work with, they exposed a lot of synchization to the render programmers that initially was done internally by drivers.
So now, we have to think of `ID3D12CommandQueue` and `VkQueue` as another thread that is actually being executed on a separate device in the unknown time-frame.
But what do we do if we want to access our texture on a different Graphics API which also executes at the unknown time-frame? We have to synchronize GPU threads!
Since we gathered here to talk about CUDA we will have to synchronize CUDA GPU threads called `cudaStream_t` with some other GPU thread like `ID3D12CommandQueue`.

But before synchronizing our data we have to talk about how to exposed said data between Graphics APIs and CUDA?
Today I am going to explain the way you can expose Texture data between APIs. Vertex and Index buffers should expose similarly, albeit they are allocated linearly you'd have to accound for that. More on that later in each section.
Each API provide own way of importing resources, so I will describe the way you import memory into each API in their corresponding sections.

I divided this post into 3 major sections: Texture and Synchronization Interops

# Handles
Another piece of crusial information that we have to know first is the way we export and import said memory addresses. Since I had to do this both for Windows and Linux, I looked into both ways those OSes handle this (no pun intended). They do so by using `handles` (for Win32) and `file descriptors` for (POSIX) but basically this is just an OS's shared address to GPU address of the exported memory. I know it a bit convoluted but we work with what we got.

Windows expects you to supply some security attributes to each shared handle so it can be successfully exposed but it still had it's quirks, more on them later.

We can fulfill this requirement with this struct:
>**NOTE**
>
>This is required only for Windows builds and doesn't require to be exposed

```cpp
#include <aclapi.h>

class WindowsSecurityAttributes
{
public:
	WindowsSecurityAttributes()
	{
		descriptor = (SECURITY_DESCRIPTOR*)calloc(1, SECURITY_DESCRIPTOR_MIN_LENGTH + 2 * sizeof(void **));
		assert(descriptor != (PSECURITY_DESCRIPTOR)nullptr);

		PSID *pp_sid = (PSID *)((PBYTE)descriptor + SECURITY_DESCRIPTOR_MIN_LENGTH);
		PACL *pp_acl = (PACL *)((PBYTE)pp_sid + sizeof(PSID *));

		InitializeSecurityDescriptor(descriptor, SECURITY_DESCRIPTOR_REVISION);

		SID_IDENTIFIER_AUTHORITY sid_identifier_authority = SECURITY_WORLD_SID_AUTHORITY;
		AllocateAndInitializeSid(&sid_identifier_authority, 1, SECURITY_WORLD_RID, 0, 0, 0, 0, 0, 0, 0, pp_sid);

		EXPLICIT_ACCESS explicit_access;
		ZeroMemory(&explicit_access, sizeof(EXPLICIT_ACCESS));
		explicit_access.grfAccessPermissions = STANDARD_RIGHTS_ALL | SPECIFIC_RIGHTS_ALL;
		explicit_access.grfAccessMode = SET_ACCESS;
		explicit_access.grfInheritance = INHERIT_ONLY;
		explicit_access.Trustee.TrusteeForm = TRUSTEE_IS_SID;
		explicit_access.Trustee.TrusteeType = TRUSTEE_IS_WELL_KNOWN_GROUP;
		explicit_access.Trustee.ptstrName = (LPTSTR)*pp_sid;

		SetEntriesInAcl(1, &explicit_access, nullptr, pp_acl);

		SetSecurityDescriptorDacl(descriptor, TRUE, *pp_acl, FALSE);

		attributes = (SECURITY_ATTRIBUTES*)calloc(1, sizeof(SECURITY_ATTRIBUTES));
		attributes->nLength = sizeof(attributes);
		attributes->lpSecurityDescriptor = descriptor;
		attributes->bInheritHandle = TRUE;
	}
	~WindowsSecurityAttributes()
	{
		PSID *pp_sid = (PSID *)((PBYTE)descriptor + SECURITY_DESCRIPTOR_MIN_LENGTH);
		PACL *pp_acl = (PACL *)((PBYTE)pp_sid + sizeof(PSID *));

		if (*pp_sid)
			FreeSid(*pp_sid);
		if (*pp_acl)
			LocalFree(*pp_acl);
		free(descriptor);
		free(attributes);
	}

	_SECURITY_ATTRIBUTES *operator&() { return attributes; }

protected:
	_SECURITY_ATTRIBUTES *attributes;
	_SECURITY_DESCRIPTOR *descriptor;
};
```



# Texture Interopability
## Basic outline
The steps we have to perform for each of the APIs are quite similar:
1. We create resource we wish to export with some kind of `SHARED` flag
2. Create shared handle
3. Import handle into target API
4. Tell the API or wrapper how it should interpret such handle

## Interoping resources with Direct3D 12
This API was most straightforward of all. Spent just a few hour understanding how I can synchronize GPU threads.
### Create shared texture
   > **NOTE**
   >
   > This piece of code uses D3D12MA but it will look similarly when using dx12 original functions, you'd have to use `CreateCommitedResource`.

```cpp
if (is_shared)
{
	// Basic check that uses SharedResourceCompatibilityTier from D3D12_FEATURE_DATA_D3D12_OPTIONS4
	// to check if format is supported or not
	// 
	// All formats are outlined at MSDN:
	// https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_shared_resource_compatibility_tier
	if (isSharedResourceFormatSupported(texture_desc.Format) == false)
	{
		// Show some error message that we can't create such texture because GPU or driver doesn't support this format
		return false;
	}
}

D3D12MA::ALLOCATION_DESC allocation_desc{};
allocation_desc.HeapType = D3D12_HEAP_TYPE_DEFAULT;
allocation_desc.Flags = D3D12MA::ALLOCATION_FLAG_COMMITTED;
allocation_desc.ExtraHeapFlags = D3D12_HEAP_FLAG_SHARED;

d3d12_allocator->CreateResource(&allocation_desc, &resource_desc, creation_state, nullptr, &allocation, IID_PPV_ARGS(&resource));
```

### Create shared handle

```cpp
WindowsSecurityAttributes windows_security_attributes;
device->CreateSharedHandle(texture->getResource(), &windows_security_attributes, GENERIC_ALL, L"", &handle);
```

### Import memory into D3D12
This is quite simple, you just have to call [ID3D12Device::OpenSharedHandle](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-opensharedhandle)
```cpp
device->OpenSharedHandle(handle, IID_PPV_ARGS(&resource));
```

Had no time to actually check how well it works and it's not that important right now.

### Acquiring VRAM Size
For correct exposure to CUDA we have to specify it correct resource size, we can get one like so:
```cpp
vram_size = device->GetResourceAllocationInfo(0, 1, &resource_desc).SizeInBytes;
```

### Linear buffers
Very similar to regular textures, albeit a bit easier. Since they work the same way, I won't go into details here.
The only thing being is that exporting textures as linear buffers have major restrictions in DX12, such as what formats can you use and where the allocated memory can reside on the hardware.

## Interoping resources with Vulkan
I spent way too much time on this one, since there were a lot of difficulties debugging this particular interop setup for both Linux and Windows. And on top of that infinite waits on GPU threads with no ability to signal semaphores or kill the app without physically pressing power button and rebooting PC this way.
Beware.

But alas, we have to do basically same steps as with DX12, albeit with a few quirks.

Firstly set tiling mode in `VkImageCreateInfo` to `VK_IMAGE_TILING_OPTIMAL` this is required, since we are going to access memory allocated by the same driver. Also we have to add `VkExternalMemoryImageCreateInfo` struct and specify handle types we want to access through.
```cpp
VkExternalMemoryImageCreateInfo vkExternalMemImageCreateInfo{};
vkExternalMemImageCreateInfo.sType = VK_STRUCTURE_TYPE_EXTERNAL_MEMORY_IMAGE_CREATE_INFO;
#ifdef _WIN32
	vkExternalMemImageCreateInfo.handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_WIN32_BIT;
#else
	vkExternalMemImageCreateInfo.handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_FD_BIT_KHR;
#endif

image_info.tiling = VK_IMAGE_TILING_OPTIMAL;
image_info.pNext = &vkExternalMemImageCreateInfo;
```

### Device Extensions
Before we can start exporting any memory from vulkan we will have to request some device extensions:
```cpp
device_extensions_set.append(VK_KHR_EXTERNAL_MEMORY_EXTENSION_NAME);
#ifdef _WIN32
	device_extensions_set.append(VK_KHR_EXTERNAL_MEMORY_WIN32_EXTENSION_NAME);
#else
	device_extensions_set.append(VK_KHR_EXTERNAL_MEMORY_FD_EXTENSION_NAME);
#endif
```
Shove this one into initialization sequence and we good to go.

### Create shared texture (VMA)
I had an opportunity to try both ways so I will present you the both ways to actually integrate this.

We begin by defining flags we want to use. Our hw memory region of interest can be accessed by GPU only and it's `HOST_VISIBLE` for some reason.
We also specify that we want to allocated dedicated memory for said texture with `VMA_ALLOCATION_CREATE_DEDICATED_MEMORY_BIT` shoved into allocation flags.
```cpp
VmaAllocationCreateInfo alloc_info{};
alloc_info.usage = VMA_MEMORY_USAGE_AUTO;
alloc_info.priority = 1.0f;

if (is_shared)
{
	alloc_info.flags = VMA_ALLOCATION_CREATE_DEDICATED_MEMORY_BIT;
	alloc_info.requiredFlags = VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT;
	alloc_info.usage = VMA_MEMORY_USAGE_GPU_ONLY;
	alloc_info.memoryTypeBits = VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT | VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT;
```

Next we define some structs to export memory. We specify windows security attributes, access and handle types. We don't have to specify access and security attributes for POSIX.
>**NOTE**
>
>VMA doesn't copy those structs so you will have to either store them somewhere close to your pool if you want to make more allocations using it, or just allocate 1 pool per shared texture. This does look ugly but it can be fixed by writing own allocator that will actually do a better job than VMA ever will.

```cpp
	VkExportMemoryAllocateInfoKHR vulkanExportMemoryAllocateInfoKHR{ VK_STRUCTURE_TYPE_EXPORT_MEMORY_ALLOCATE_INFO_KHR };

	#ifdef _WIN32
		WindowsSecurityAttributes windows_security_attributes;
		VkExportMemoryWin32HandleInfoKHR vulkanExportMemoryWin32HandleInfoKHR{ VK_STRUCTURE_TYPE_EXPORT_MEMORY_WIN32_HANDLE_INFO_KHR };

		vulkanExportMemoryWin32HandleInfoKHR.pAttributes = &windows_security_attributes;
		vulkanExportMemoryWin32HandleInfoKHR.dwAccess = DXGI_SHARED_RESOURCE_READ | DXGI_SHARED_RESOURCE_WRITE;

		vulkanExportMemoryAllocateInfoKHR.pNext = &vulkanExportMemoryWin32HandleInfoKHR;
		vulkanExportMemoryAllocateInfoKHR.handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_WIN32_BIT;
	#else
		vulkanExportMemoryAllocateInfoKHR.handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_FD_BIT_KHR;
	#endif
```

In order to allocated shared memory through VMA we have to create custom pool with previously defined structs.
First we want to find memory type index for our feature allocation.
```cpp
	uint32_t memory_type_index{};
	vmaFindMemoryTypeIndexForImageInfo(vma_allocator, &info, &alloc_info, &memory_type_index);
```

We pass it to `VmaPoolCreateInfo` struct along with our extensions defined previously.
Notice that we specify `blockSize` to 0. This is actually from VMA docs:
> **NOTE** On VMA
>
> New versions of this library support creating dedicated allocations in custom pools.
>It is supported only when [VmaPoolCreateInfo::blockSize](https://gpuopen-librariesandsdks.github.io/VulkanMemoryAllocator/html/struct_vma_pool_create_info.html#aa4265160536cdb9be821b7686c16c676 "Size of a single VkDeviceMemory block to be allocated as part of this pool, in bytes....") = 0.

```cpp
	VmaPoolCreateInfo pool_info{};
	pool_info.memoryTypeIndex = memory_type_index;
	pool_info.blockSize = 0;
	pool_info.maxBlockCount = 1;
	pool_info.pMemoryAllocateNext = &vulkanExportMemoryAllocateInfoKHR;

	vmaCreatePool(vma_allocator, &pool_info, &pool);
```

The only thing left is actual allocation and getting correct vram size of it.
```cpp
	alloc_info.pool = pool;

	vmaCreateImage(vma_allocator, &info, &alloc_info, &image, &allocation, nullptr);

	VkMemoryRequirements memory_req{};
	vkGetImageMemoryRequirements(vulkan_device, image, &memory_req);
	vram_size = memory_req.size;
}
```



### Create shared texture (Native)
Integrating this into own memory allocator was quite easy, if we don't count the amount of headaches I had due to poor debug info, no documentation notes and other shenanigans.

First we create image and ask it's size:
```cpp
vkCreateImage(vulkan_device, &image_info, nullptr, image);

VkMemoryRequirements memory_req{};
vkGetImageMemoryRequirements(vulkan_device, image, &memory_req);
vram_size = memory_req.size;
```

Steps pretty similar to VMA but we have to add another struct here, so that vulkan knows that we want dedicated allocation:
```cpp
	VkMemoryDedicatedAllocateInfoKHR dedicatedInfo{ VK_STRUCTURE_TYPE_MEMORY_DEDICATED_ALLOCATE_INFO };
	dedicatedInfo.image = image;
```

Same as before with VMA, but notice that we specify `dedicatedInfo` into some structs:
```cpp
	VkExportMemoryAllocateInfoKHR vulkanExportMemoryAllocateInfoKHR{ VK_STRUCTURE_TYPE_EXPORT_MEMORY_ALLOCATE_INFO_KHR };

	#ifdef _WIN32
		WindowsSecurityAttributes windows_security_attributes;
		VkExportMemoryWin32HandleInfoKHR vulkanExportMemoryWin32HandleInfoKHR{ VK_STRUCTURE_TYPE_EXPORT_MEMORY_WIN32_HANDLE_INFO_KHR };
		vulkanExportMemoryWin32HandleInfoKHR.pNext = &dedicatedInfo;
		vulkanExportMemoryWin32HandleInfoKHR.pAttributes = &windows_security_attributes;
		vulkanExportMemoryWin32HandleInfoKHR.dwAccess = DXGI_SHARED_RESOURCE_READ | DXGI_SHARED_RESOURCE_WRITE;

		vulkanExportMemoryAllocateInfoKHR.pNext = &vulkanExportMemoryWin32HandleInfoKHR;
		vulkanExportMemoryAllocateInfoKHR.handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_WIN32_BIT;
	#else
		vulkanExportMemoryAllocateInfoKHR.pNext = &dedicatedInfo;
		vulkanExportMemoryAllocateInfoKHR.handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_FD_BIT_KHR;
	#endif
```

Then we just fill allocate info struct:
```cpp
VkMemoryAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
allocInfo.allocationSize = memory_req.size;
allocInfo.memoryTypeIndex = FindMemoryType(memory_req.memoryTypeBits, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT);
```

And finally allocate memory and then bind it by using `vkAllocateMemory` and `vkBindImageMemory` respectively.

> **WARNING** vkGetMemoryWin32HandleKHR return's nullptr handles with no errors!
>
> vkGetMemoryWin32HandleKHR may return `VK_SUCCESS` but handle will be `nullptr`.
> Please double check if you actually fill and added VkExportMemoryAllocateInfoKHR into pNext chain!
> Works this way on both NVIDIA and AMD.

### Importing memory into Vulkan
For importing memory into Vulkan we can use [VkImportMemoryWin32HandleInfoKHR](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkImportMemoryWin32HandleInfoKHR.html) and [VkImportMemoryFdInfoKHR](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkImportMemoryFdInfoKHR.html) for POSIX handles.
```cpp
VkImportMemoryWin32HandleInfoKHR desc{};
desc.sType = VK_STRUCTURE_TYPE_IMPORT_MEMORY_WIN32_HANDLE_INFO_KHR;
desc.handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_WIN32_BIT; // Same as before, just the other way around
desc.handle = my_handle;
```

They both for the same way:
1. We create an image to be imported with vkCreateImage.
2. Specify our Import memory structure into pNext of the [VkMemoryAllocateInfo](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkMemoryAllocateInfo.html) struct.
3. Allocate image as usual

### Linear buffers
Very similar to regular textures, albeit a bit easier. Since they work the same way, I won't go into details here.

## Extra: Interoping resources with Direct3D 11
This one was simple almost the same way as DX12, it took me just about an hour to implement, so i included it since we will probably support this one for a bit longer.

### Create shared texture
I only did that for textures since buffers required a bit of a extra hassle with keyed mutexes. And since we're going to remove DX11 support in the future in favor of DX12 and Vulkan, it wouldn't really be great to add 2 extra functions to lock/unlock keyed mutex just for the DX11.

But alas, in order to create a DX11 texture that will support shared handles we specify 2 flags, NTHANDLE tells to the API that we want to request NTHandle and the second one tells that we want to be able to share this resource to the external API.
```cpp
misc_flags |= D3D11_RESOURCE_MISC_SHARED_NTHANDLE | D3D11_RESOURCE_MISC_SHARED;
```

If we were to use Keyed mutex here, we would replace ``D3D11_RESOURCE_MISC_SHARED_NTHANDLE`` with ``D3D11_RESOURCE_MISC_SHARED_NTHANDLE`` instead.

### Create shared handle
This one requires a bit of a tinkering. First you ``QueryInterface`` for the ``IDXGIResource1`` and then you can create shared handle from it.
```cpp
IDXGIResource1 *dxgi_resource = nullptr;
resource->QueryInterface(__uuidof(IDXGIResource1), (void **)&dxgi_resource);

WindowsSecurityAttributes windows_security_attributes;

DWORD flags = DXGI_SHARED_RESOURCE_READ | DXGI_SHARED_RESOURCE_WRITE;
dxgi_resource->CreateSharedHandle(&windows_security_attributes, flags, L"", &handle);
```

### Import shared handle
I haven't looked into this yet, but it seems straightforward enough:
```cpp
device->OpenSharedResource(handle, __uuidof(ID3D11Texture2D), (void **)&texture);
```

### Acquiring VRAM Size
Since old APIs like DX11 allocate things for you, the only way you can check for the size of a resource is by calculating it yourself with a simple expression like so:
```cpp
texture_size_in_bytes = 0;

for (num_mips)
{
	texture_size_in_bytes += width * height * depth;
	if (width > 1) width >>= 1;
	if (height > 1) height >>= 1;
	if (depth > 1) depth >>= 1;
}

texture_size_in_bytes *= array_size * (isCubeOrCubeArray() ? 6 : 1) * (1 << multisampling_mode) * format_size_in_bits / 8;
```

## Interoping resources with CUDA
Finally the one we want to interop with.

We will have to fill a couple of structs, such as `cudaExternalMemoryHandleDesc` and ``cudaExternalMemoryMipmappedArrayDesc``.

The first one is pretty easy.
Depending on your graphics API that your engine runs on, you will have to choose between a few options of resource types:
```cpp
cudaExternalMemoryHandleDesc desc{};
switch (Render::GetAPI())
{
	case Render::API::DIRECT3D11: desc.type = cudaExternalMemoryHandleTypeD3D11Resource; break;
	case Render::API::DIRECT3D12: desc.type = cudaExternalMemoryHandleTypeD3D12Resource; break;
	case Render::API::VULKAN:
		#ifdef _WIN32
			desc.type = cudaExternalMemoryHandleTypeOpaqueWin32;
		#else
			desc.type = cudaExternalMemoryHandleTypeOpaqueFd;
		#endif
		break;
}
```

Specify handles we got from Vulkan or DX11/12
```cpp
#ifdef _WIN32
	desc.handle.win32.handle = resource->GetHandleWin32();
#else
	desc.handle.fd = resource->GetHandleFd();
#endif
```

Here we will have to set `vram_size` that we calculated for our texture previously and tell CUDA that we allocated dedicated memory. Not really sure about the way DX12 allocates memory but when you call CreateCommitedResource it will be dedicated, based on my experiences with debugging vulkan allocations. By default vulkan won't do dedicated allocations, hence you have to specify some helper structs for that on vulkan side. Obviously you don't have to do this but the reason I did it, is that so code written for a single Engine's API will be similar for a different Engine API. Not a lot of the users actually know how memory is allocated behind the scenes so to help with that I tried my best to make it tiny bit easier to work with (: .
```cpp
desc.size = texture->GetVRAMSize();
desc.flags = cudaExternalMemoryDedicated;

cudaImportExternalMemory(&external_memory, &desc);
```


``cudaExternalMemoryMipmappedArrayDesc`` asks for texture dimensions, format and some flags.
I will go with 2D texture for simplicity:
```cpp
cudaExternalMemoryMipmappedArrayDesc mip_mapped_array_desc{};
mip_mapped_array_desc.extent = make_cudaExtent(texture->GetWidth(), texture->GetHeight(), 0);
```
For more info on specifying texture dimensions look up [cudaMallocMipmappedArray](https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__MEMORY.html#group__CUDART__MEMORY_1g9abd550dd3f655473d2640dc85be9774) in the CUDA docs.

Format is going to be RGBA 8 bit per channel.
```cpp
mip_mapped_array_desc.formatDesc = { 8, 8, 8, 8, cudaChannelFormatKindUnsigned };
```

I have 1 mip level.
```cpp
mip_mapped_array_desc.numLevels = 1;
```

For flags we ask for Load and Store access (basically UAV)
```cpp
mip_mapped_array_desc.flags = cudaArraySurfaceLoadStore;
```
And if this texture is ever going to be used as render target via Engine we say that by adding another flag:
```cpp
mip_mapped_array_desc.flags |= cudaArrayColorAttachment;
```

Now the rest is easy, get 0th mip map and create surface object so that we can read/write anything from/to the texture using CUDA kernels.
```cpp
cudaArray_t cu_array{};
cudaGetMipmappedArrayLevel(&cu_array, mip_array, 0);

cudaResourceDesc resource_desc{};
resource_desc.resType = cudaResourceTypeArray;
resource_desc.res.array.array = cu_array;
cudaCreateSurfaceObject(&surface, &resource_desc);
```


> **WARNING**
>
> If you encounter an invalid argument error at this step, please check that you have specified correct texture size at the previous step and that  your external_memory object is not 0x00000000.




# Synchronization Interopability
We already half way through our journey! Enter the land of perma-frozen GPU thread-lands and synchronization of the GPU trains!
Next stop is - *insanity*.

## Timeline
At first glance we have a very simple syncronization problem we would like to solve.
*I used API as an umbrella term for DX11/DX12/Vulkan graphics APIs.*
![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/CUDA_API_Texture_Sync_Chart.png?raw=true)

But in reality our diagram should look like this:
![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/CPU_API_CUDA_Texture_Sync_Chart.png?raw=true)

We still have to account for CPU, this way we can actually see where each of the API call is being made at and debug accordingly.
But to make this image a bit simplier to understand let's put actual calls into perspective:
![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/CPU_API_CUDA_Sync_Chart.png?raw=true)

![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/API_CUDA_Sync_Chart.png?raw=true)

## Direct3D 12
Let's start with DX12 since it's easier to grasp.

We need 3 things: A new fence, a handle for that fence and Win32 event:
```cpp
WindowsSecurityAttributes windows_security_attributes;

fence_value = 0;
device->CreateFence(fence_value, D3D12_FENCE_FLAG_SHARED, D3D12_IID_PPV_ARGS(&fence));
device->CreateSharedHandle(fence, &windows_security_attributes, GENERIC_ALL, L"", &handle);

fence_event = CreateEvent(nullptr, FALSE, FALSE, nullptr);
```

Win32 event is used for CPU <-> GPU signaling and locking on one of the threads (either CPU or GPU).
Where's a new fence is our sync primitive between GPU threads.
Now, let's start with 4 basic operations that we need to implement:
1. **CPU Wait GPU**
   Before we can destroy any of the resources, and we actually *do* care when we can destroy our resources,
   we need to know when we can do it. By far the simpliest and most basic way is to just wait for it.
   
   And since other GPU thread outside of our engine may still want to execute some stuff we have to do this in order:
   1. Stop Engine GPU thread
      you already should have that function, it may be called something like `WaitGPU` or `WaitGraphics`
      we want to modify it a bit:
	  
      1. Signal our shared fence from the CPU
      2. Then the rest is usual
	  
      We want to signal shared fence from the CPU, so it knows that it should do some work, instead of waiting to be fed.
	  
   2. Increment fence value
   3. Signal our fence on Command Queue with current fence value
   4. Wait for the Event, similar to `WaitGPU` that we already should have in our engine.
   5. Increment `fence_value` one last time.
   
   
2. **CPU Signal GPU**
   
   ```cpp
   fence->Signal(fence_value);
   ```
   
   That is all there is to it (:
   
3. **GPU Wait**
   
   Very simple and elegant
   
   ```cpp
   CommandQueue->Wait(fence, fence_value);
   ```
   
4. **GPU Signal**
   
   Before we issue our signal, we need to increment our `fence_value`.
   Very simple and very elegant!
   
   ```cpp
   CommandQueue->Signal(fence, fence_value);
   ```

Just before we call `ExecuteCommandLists` where our texture is going to be accessed, we need to add some guards:
At the frame, when our fence is going to be used for the first time, we skip GPU Wait, all subsequent calls should issue GPU Wait here.
After `ExecuteCommandLists` call always issue GPU Signal.

Then you want to insert an event, callback or something like that for the Engine API, so the other API can insert it's GPU Wait, Work and GPU Signal here, right before `IDXGISwapchain::Present()` call.
## Vulkan
Vulkan implementation isn't as elegant as DX12's is, but it has same points of interest.

Let's start by creaing Vulkan's Timeline semaphore, it requires `VK_KHR_timeline_semaphore` extension. And we also need `VK_KHR_external_semaphore` and `VK_KHR_external_semaphore_fd` / `VK_KHR_external_semaphore_win32` for POSIX and Win32 respectively.
We use Timeline semaphore, since it acts similar to DX12 fence.
```cpp
VkSemaphoreCreateInfo semaphore_desc{};
semaphore_desc.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;

VkExportSemaphoreCreateInfoKHR export_semaphore_desc{};
export_semaphore_desc.sType = VK_STRUCTURE_TYPE_EXPORT_SEMAPHORE_CREATE_INFO_KHR;
#ifdef _WIN32
	// FIXME: WindowsSecurityAttributes issue with CUDA external semaphore signal
	//
	// For some unknown reason timline semaphores won't signal on Windows via CUDA
	// when we provide VkExportSemaphoreWin32HandleInfoKHR struct
	// This might be avaliable as an optional toggle in feature for CUDA workaround

	//WindowsSecurityAttributes windows_security_attributes;

	//VkExportSemaphoreWin32HandleInfoKHR export_semaphore_handle_desc{};
	//export_semaphore_handle_desc.sType = VK_STRUCTURE_TYPE_EXPORT_SEMAPHORE_WIN32_HANDLE_INFO_KHR;
	//export_semaphore_handle_desc.pAttributes = &windows_security_attributes;
	//export_semaphore_handle_desc.dwAccess = DXGI_SHARED_RESOURCE_READ | DXGI_SHARED_RESOURCE_WRITE;

	//export_semaphore_desc.pNext = &export_semaphore_handle_desc;
	export_semaphore_desc.handleTypes = VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_WIN32_BIT;
#else
	export_semaphore_desc.handleTypes = VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_FD_BIT;
#endif

VkSemaphoreTypeCreateInfoKHR timeline_info{};
timeline_info.sType = VK_STRUCTURE_TYPE_SEMAPHORE_TYPE_CREATE_INFO;
timeline_info.pNext = &export_semaphore_desc;
timeline_info.initialValue = fence_value;
timeline_info.semaphoreType = VK_SEMAPHORE_TYPE_TIMELINE;

semaphore_desc.pNext = &timeline_info;
vkCreateSemaphore(vk_device, &semaphore_desc, nullptr, &semaphore);

#ifdef _WIN32
	VkSemaphoreGetWin32HandleInfoKHR semaphore_handle_desc{};
	semaphore_handle_desc.sType = VK_STRUCTURE_TYPE_SEMAPHORE_GET_WIN32_HANDLE_INFO_KHR;
	semaphore_handle_desc.handleType = VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_WIN32_BIT;

	semaphore_handle_desc.semaphore = semaphore;
	vkGetSemaphoreWin32HandleKHR(vk_device, &semaphore_handle_desc, &handle);
#else
	VkSemaphoreGetFdInfoKHR semaphore_handle_desc{};
	semaphore_handle_desc.sType = VK_STRUCTURE_TYPE_SEMAPHORE_GET_FD_INFO_KHR;
	semaphore_handle_desc.handleType = VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_FD_BIT;

	semaphore_handle_desc.semaphore = semaphore;
	vkGetSemaphoreFdKHR(vk_device, &semaphore_handle_desc, &handle);
#endif
```
Another strange bit here is absence of `VkExportSemaphoreWin32HandleInfoKHR`. For some unknown reason when you export semaphore with this struct and import handle into CUDA.
The latter starts to act strangely. GPU Signal doesn't work and always returns `cudaErrorUnknown`, while Wait does work as expected.
Sync point for the Vulkan implementation is the same as in D3D12, so I will skip it.


All 4 functions that we need are going to be very similar to DX12, albeit not so beautiful and elegant.
### CPU Wait GPU
Since we are using timeline semaphores we need to add one more struct to this whole thing, `VkTimelineSemaphoreSubmitInfoKHR`, it will allow as to specify `fence_value` we want to signal or wait on.
```cpp
vk_queue->WaitGPU();

fence_value++;

VkSubmitInfo submit{};
submit.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
submit.pSignalSemaphores = &semaphore;
submit.signalSemaphoreCount = 1;

VkTimelineSemaphoreSubmitInfoKHR timeline_semaphore_info{};
timeline_semaphore_info.sType = VK_STRUCTURE_TYPE_TIMELINE_SEMAPHORE_SUBMIT_INFO;
timeline_semaphore_info.pSignalSemaphoreValues = &fence_value;
timeline_semaphore_info.signalSemaphoreValueCount = 1;
submit.pNext = &timeline_semaphore_info;

vkQueueSubmit(vk_queue, 1, &submit, VK_NULL_HANDLE);

fence_value++;
```

Implementation is very similar to DX12.

### CPU Signal GPU
Here we can use `vkSignalSemaphoreKHR` function to signal our semaphore from the CPU.
```cpp
VkSemaphoreSignalInfoKHR info{};
info.sType = VK_STRUCTURE_TYPE_SEMAPHORE_SIGNAL_INFO_KHR;
info.semaphore = semaphore;
info.value = fence_value;

vkSignalSemaphoreKHR(vk_queue, &info);
```

### GPU Wait
Not so elegant solution by Vulkan:
```cpp
VkPipelineStageFlags wait_mask = VK_PIPELINE_STAGE_ALL_COMMANDS_BIT;

VkSubmitInfo submit{};
submit.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
submit.pWaitSemaphores = &semaphore;
submit.waitSemaphoreCount = 1;
submit.pWaitDstStageMask = &wait_mask;

VkTimelineSemaphoreSubmitInfoKHR timeline_semaphore_info{};
timeline_semaphore_info.sType = VK_STRUCTURE_TYPE_TIMELINE_SEMAPHORE_SUBMIT_INFO;
timeline_semaphore_info.pWaitSemaphoreValues = &fence_value;
timeline_semaphore_info.waitSemaphoreValueCount = 1;
submit.pNext = &timeline_semaphore_info;

vkQueueSubmit(vk_queue, 1, &submit, VK_NULL_HANDLE);

VkSemaphoreWaitInfo wait_info{};
wait_info.sType = VK_STRUCTURE_TYPE_SEMAPHORE_WAIT_INFO;
wait_info.semaphoreCount = 1;
wait_info.flags = VK_SEMAPHORE_WAIT_ANY_BIT;
wait_info.pValues = &fence_value;
wait_info.pSemaphores = &semaphore;

vkWaitSemaphores(vk_device, &wait_info, 0xFFFFFFFF);
```

Same as `CommandQueue->Wait()` but not so nice.

### GPU Signal
Same as before in `GPU Wait`, the only difference being is that we use our fence to signal, not to wait on it.
```cpp
fence_value++;

VkSubmitInfo submit{};
submit.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
submit.pSignalSemaphores = &semaphore;
submit.signalSemaphoreCount = 1;

VkTimelineSemaphoreSubmitInfoKHR timeline_semaphore_info{};
timeline_semaphore_info.sType = VK_STRUCTURE_TYPE_TIMELINE_SEMAPHORE_SUBMIT_INFO;
timeline_semaphore_info.pSignalSemaphoreValues = &fence_value;
timeline_semaphore_info.signalSemaphoreValueCount = 1;
submit.pNext = &timeline_semaphore_info;

vkQueueSubmit(vk_queue, 1, &submit, VK_NULL_HANDLE);
```

## Extra: Direct3D 11
Remember how Vulkan was similar to DX12? DX11 is no exception here, I'll quickly glance over 4 basic functions and we will end this section with CUDA Interop.

It has same API calls to create new fence, and requires same 3 things:
```cpp
WindowsSecurityAttributes windows_security_attributes;

device5->CreateFence(fence_value, D3D11_FENCE_FLAG_SHARED, D3D12_IID_PPV_ARGS(&fence));
fence->CreateSharedHandle(&windows_security_attributes, GENERIC_ALL, L"", &handle);

fence_event = CreateEvent(nullptr, FALSE, FALSE, nullptr);
```

### CPU Wait GPU
Since we have no Command Queue exposed in D3D11 we have to do [ID3D11DeviceContext::Flush()](https://learn.microsoft.com/en-us/windows/win32/api/d3d11/nf-d3d11-id3d11devicecontext-flush) here instead.
But everything after that is same as before.
```cpp
context4->Flush();

fence_value++;

context4->Signal(fence, fence_value);
fence->SetEventOnCompletion(fence_value, fence_event);
WaitForSingleObjectEx(fence_event, INFINITE, FALSE);

fence_value++;
```

### CPU Signal GPU
For some reason I couldn't find anything related to that.

### GPU Wait
Very elegant api calls by DirectX 11:
```cpp
context4->Wait(fence, fence_value);
```

### GPU Signal
Here's one more:
```cpp
context4->Signal(fence, fence_value);
```

### Sync Point
Again, since we don't have Command Queue exposed, we have to go a bit different route in order to simulate same behavior.
What currently works for me is to do the same thing as in DX12 implementation but instead of wrapping ExecuteCommandList calls, I wrap my logic around [ID3D11DeviceContext::Flush()](https://learn.microsoft.com/en-us/windows/win32/api/d3d11/nf-d3d11-id3d11devicecontext-flush) right before Present calls.

## CUDA
Finally we are almost done with this.
Since we want our CUDA work to be synchronized with Vulkan/D3D12/D3D11 work we need to have some kind of thread.
Luckily for us CUDA has Streams which act as individual Command Queues we can launch our work kernels on and use them to sync-up our GPU work between APIs.

API for that is actually pretty easy, we create CUDA Stream in a single line and immediately call for sync. Why? I am not sure.. But without it may not work for some reason.
Might be something along the lines of "threads are always doing some work but you need to sync it before you submit any of your own".
It may require some more testing on my end but I am already at my limit with all of this, so I'll toss it into the endless list of *things I have to look into later*.
```cpp
cudaStreamCreateWithFlags(&stream, cudaStreamNonBlocking);
cudaStreamSynchronize(stream);
```

Now that we have another GPU thread, let's import a fence from the APIs I described above.
The process is remarkably simple and quite intuitive:
1) Specify type and handle of the imported fence
2) Get new CUDA object back
```cpp
cudaExternalSemaphoreHandleDesc desc{};
#ifdef _WIN32
	switch (Render::GetAPI())
	{
		case Render::API::DIRECT3D11: desc.type = cudaExternalSemaphoreHandleTypeD3D11Fence; break;
		case Render::API::DIRECT3D12: desc.type = cudaExternalSemaphoreHandleTypeD3D12Fence; break;
		case Render::API::VULKAN: desc.type = cudaExternalSemaphoreHandleTypeTimelineSemaphoreWin32; break;
	}

	desc.handle.win32.handle = fence->GetHandleWin32();
#else
	desc.type = cudaExternalSemaphoreHandleTypeTimelineSemaphoreFd;
	desc.handle.fd = fence->GetHandleFd();
#endif

cudaImportExternalSemaphore(external_semaphore, &desc);
```


The only 2 things left is to make 2 functions for GPU Wait and Signal operations and launch our CUDA kernel.

### GPU Wait
Both Wait and Signal functions ask for the CUDA Stream you want them to operate on.
We can specify the one we created recently, as for the value, our Engine Fence should have which value we should be interested in.
```cpp
cudaExternalSemaphoreWaitParams desc{};
desc.params.fence.value = fence->GetValue();

cudaWaitExternalSemaphoresAsync(&external_semaphore, &desc, 1, stream);
```

### GPU Signal
As you may have noticed, each Signal op is preceeded with value incrementation. This trend continues here as well.
```cpp
fence->IncrementValue();

cudaExternalSemaphoreSignalParams desc{};
desc.params.fence.value = fence->GetValue();
cudaSignalExternalSemaphoresAsync(&external_semaphore, &desc, 1, stream);
```

Nothing fancy to describe here either.

### Sync Point
Lastly we want to start our work for the frame and we can do so by subscribing to specific callback I described earlier in DX12 sync section.
It should be right after our `Wait`/`ExecuteCommandList`/`Signal` chain and right before `Present` calls.
```cpp
if (have_work_kernel == false)
{
	CUDAWait();
	LaunchKernel();
}
CUDASignal();
```

# Resource Destruction / Re-creation
Okay, it was quite the journey but we still have to tidy up our resources at some point, so we are not done yet.
We still have 3 more topics to cover like:
1) Closing Handles
2) Destroying Resources
3) Re-creating Resources

What's interesting here is that they all rely on the previous ones.
We can't re-create resource handles without destroying the old ones and we can't destroy old resources without closing their corresponding handles.
But fear not everything here is actually more straightforward than you might've fought.

### Closing Handles
Let's start with this one. Every time our resource is cleared, destroyed or changed memory location we need to close old handle.
On Windows we just have to call [CloseHandle](https://learn.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-closehandle) for that. But what if the underlying handle is null? Let's add check for that!
But what if the handle isn't null but it's invalid for some reason? We can check that with [GetHandleInformation](https://learn.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-gethandleinformation)!
The only thing left is to notify CUDA or other API that we handed this handle(no pun intended) to that it is no longer valid.
In the end we get something along the lines of:
```cpp
DWORD flags;
if (handle != nullptr && GetHandleInformation(handle, &flags))
{
	event_close_handle.Run();
	CloseHandle(handle);
}
handle = nullptr;
```

Perfect, now let's do the Linux version of this. Again, closing handle is as simple as calling [close](https://www.man7.org/linux/man-pages/man2/close.2.html) function.
Checking if handle is valid is the same as checking for return value of [fcntl](https://www.man7.org/linux/man-pages/man2/fcntl.2.html) with `F_GETFD` flag.
```cpp
if (handle != -1 && fcntl(handle, F_GETFD))
{
	event_close_handle.run();
	close(handle);
}
handle = -1;
```

### Destroying Resources
For the internal version of a resource clear i've used destructor of external memory object which automatically closes handle when it is destroyed. And it is destroyed the moment original resource is destroyed or re-created.

For the external one I did only *bare minimum* so that original samples are not overwhelming.
But I left some comments at the destruction of shared resource wrapper:
```cpp
// NOTE:
// This is not optimal for Mesh Dynamic re-creation between frames
// as it's introduces high-cost GPU <-> CPU Sync point before we can destory old resources.
//
// What you want to do instead is to have double-buffered Mesh Dynamics
// and to have a separate thread which will carry destruction of the current Mesh Dynamic
// while the other one is used to render something.
//
// Also you can use a single fence for all of your shared resources so that you will have to wait
// single time instead of waiting for multiple fences, this also may improve performance
// as far fewver sync Wait/Signal pairs will be issued and waited on.
```

So, the idea is simple: before we can destroy this resource, we must ensure that it is no longer needed on neither of the GPU Timelines.
```cpp
cudaStreamSynchronize(stream);
CUDASignal();
fence->WaitGPU(); // Wait for Engine's Queue to finish
```

Now we certain that resource is no longer needed and we can call `cudaFree`, `cudaDestroySurfaceObject` or `cudaFreeMipmappedArray`. After that we can destroy `cudaDestroyExternalMemory` and `cudaDestroyExternalSemaphore`.

### Re-creating Resources
If you haven't skipped other parts of this post, you must've noticed `event_close_handle` before. The only thing we missing is `event_create_handle`, it should be right after `CreateSharedHandle` call for external memory and similar in Vulkan.
Those events fire correspondingly when we close or create a handle to the resource externally.
On the create handle event we can call `create_handle` function that will make a new CUDA handle for the resource, no things to change here.
Closing handle is simple as destroying CUDA's external memory, since engine already destroyed resource internally and called `closeHandle` which fired `event_close_handle`.

# Tips & Tricks
This should do it but if you wish to go by the same path as me, i've collected a few more Tips that may help you to save some time.

### Sync
Use a bit of `nsys` from time to time, it may help you understand sync timeline a bit better.

### Permafreeze

#### CPU permafreeze
Use `procexp64` from Sysinternals Suite to firstly kill some *App* threads that have been Waiting for eternity, and then try to terminate it.

#### GPU permafreeze
If you have done something wrong, your App may freeze indefinitely due to lack proper Signal/Wait chain on GPU.
Windows TDR won't handle this at all, especially for Vulkan. Althrough 1 time TDR happend to kill an DX11 GPU permarozen app after 10 or so minutes.
But If you want to continue your journey then you have 2 choises:
1) Restart PC by long-press of the hardware button
2) Rename resulting binaries and try again

You can also wrap all of your CUDA calls into some macro that contains an assert:
```cpp
#define cudaCheck(x) \
{ \
	cudaError_t res = x; \
	if (res != cudaSuccess) \
		Log::Assert("%s: %s Failed with error '%s' (%d)", __FUNCTION__, #x, cudaGetErrorString(res), res); \
}


...

cudaCheck(cudaSignalExternalSemaphoresAsync(...));
```
So it can, hopefully, prevent permafreezing your App.

### Linux
On linux you may want to use `cuda-gdb` to catch a few strange bugs that may occur.

If for some reason your app can't call CUDA Driver and CUDA Runtime APIs, try this (you may want to change path to `libcuda.so`):
``LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libcuda.so cuda-gdb -ex=r --args ../bin/app_x64d <my_args_go_here_as_usual>``

If everything works fine, check that you don't have ``libcuda.so.6`` or something like this neither in ``../bin/`` nor in the `$PATH`.
I copied some random .so way back when i was experimenting at it tripped me for like a whole week or so.
I tried everything from re-installing drivers 5 times to other obscure things.
Knowing what libs app was loading at the startup would've helped a lot, much like VS does in it's output window.

### Memory artifacts
If you are unsure what may cause weird image artifacts then use this kernel to debug root cause of the artifacts:

```cpp
__device__ unsigned int RGBAFloatToInt(float4 rgba)
{
	rgba.x = __saturatef(rgba.x); // clamp to [0.0, 1.0]
	rgba.y = __saturatef(rgba.y);
	rgba.z = __saturatef(rgba.z);
	rgba.w = __saturatef(rgba.w);
	return  ((unsigned int)(rgba.w * 255.0f) << 24) |
			((unsigned int)(rgba.z * 255.0f) << 16) |
			((unsigned int)(rgba.y * 255.0f) << 8) |
			((unsigned int)(rgba.x * 255.0f));
}

__global__ static void Kernel(cudaSurfaceObject_t surface, int nWidth, int nHeight) {
    int x = (threadIdx.x + blockIdx.x * blockDim.x) * 2;
    int y = (threadIdx.y + blockIdx.y * blockDim.y) * 2;
    if (x + 1 >= nWidth || y + 1 >= nHeight)
        return;
	
    float4 rgba{};
    rgba.x = (x & 0xFF) / 255.0f;
    rgba.y = (y & 0xFF) / 255.0f;
    rgba.z = 0.0f;
    rgba.w = 1.0f;
    int color = RGBAFloatToInt(rgba);
    surf2Dwrite(color, surface, x * sizeof(uchar4), y);
    surf2Dwrite(color, surface, x * sizeof(uchar4), y + 1);
    surf2Dwrite(color, surface, (x + 1) * sizeof(uchar4), y);
    surf2Dwrite(color, surface, (x + 1) * sizeof(uchar4), y + 1);
}

__global__ static void KernelLinear(void *ptr, int nWidth, int nHeight)
{
    int x = (threadIdx.x + blockIdx.x * blockDim.x) * 2;
    int y = (threadIdx.y + blockIdx.y * blockDim.y) * 2;
    if (x + 1 >= nWidth || y + 1 >= nHeight)
        return;
	
	uint8_t *pDst = ptr + (x + y * nWidth) * sizeof(uchar4);
	
	float4 rgba{};
	rgba.x = (x & 0xFF) / 255.0f;
	rgba.y = (y & 0xFF) / 255.0f;
	rgba.z = 0.0f;
	rgba.w = 1.0f;
	unsigned int color = RGBAFloatToInt(rgba);
	
	*(uint2 *)pDst = uint2{
		color,
		color,
	};
	*(uint2 *)(pDst + nWidth * sizeof(uchar4)) = uint2{
		color,
		color,
	};
}

void LaunchKernel(cudaStream_t stream, cudaSurfaceObject_t surface, int nWidth, int nHeight) {
    Kernel<<<dim3(nWidth / (16 * 2), nHeight / (8 * 2)), dim3(16, 8), 0, stream>>>(surface, nWidth, nHeight);
}
void LaunchKernelLinear(cudaStream_t stream, void *ptr, int nWidth, int nHeight) {
    KernelLinear<<<dim3(nWidth / (16 * 2), nHeight / (8 * 2)), dim3(16, 8), 0, stream>>>(ptr, nWidth, nHeight);
}

```

Then produce square image 1024x1024 with this kernel and use artifact table to find the root cause.
Your goal is to get this image and to have correct parameters on both sides:
![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/Pasted%20image%2020240309132129.png?raw=true)


Here's the artifact table I produced. Top row denotes export memory, left most column describes import memory parameters:

|                                    | **Non-Dedicated**, Optimal Tiling    | **Dedicated**, Optimal Tiling        | Non-Dedicated, Linear Tiling                                                      | Dedicated, Linear Tiling                                                          |
| ---------------------------------- | ------------------------------------ | ------------------------------------ | --------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **Non-Dedicated<br>Texture 2D**    | ![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/Pasted%20image%2020240309132129.png?raw=true) | ![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/Pasted%20image%2020240309132129.png?raw=true) | _check notes_ | _check notes_ |
| **Dedicated<br>Texture 2D**        | ![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/Pasted%20image%2020240309131844.png?raw=true) | ![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/Pasted%20image%2020240309132129.png?raw=true) | _check notes_ | _check notes_ |
| **Non-Dedicated<br>Linear Buffer** | ![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/Pasted%20image%2020240309140535.png?raw=true) | ![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/Pasted%20image%2020240309140535.png?raw=true) | ![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/Pasted%20image%2020240309132129.png?raw=true)                                              | ![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/Pasted%20image%2020240309132129.png?raw=true)                                              |
| **Dedicated<br>Linear Buffer**     | ![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/Pasted%20image%2020240309140535.png?raw=true) | ![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/Pasted%20image%2020240309140535.png?raw=true) | ![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/Pasted%20image%2020240309132129.png?raw=true)                                              | ![](https://github.com/all500234765/all500234765.github.io/blob/master/_data/Pasted%20image%2020240309132129.png?raw=true)                                              |

Notes:
1. You can't do that. It will fail on `cudaExternalMemoryGetMappedMipmappedArray`

From: [CUDA Vulkan VkImage Interop - CUDA / CUDA Programming and Performance - NVIDIA Developer Forums](https://forums.developer.nvidia.com/t/cuda-vulkan-vkimage-interop/278691/4)

> **NOTE**
>
> Note from NVIDIA forums
> The underlying reason there\`s this requirement is because if the driver knows that an image uses a dedicated allocation,
> then it knows there\`s only one image in an allocation and that that image has offset 0, which allows it to do different optimizations,
> including a different image layout in memory.
> 
> (Thanks to Vivek Kini for this info).

> **WARNING**
>
> The two APIs must agree on the depth of the image - in particular, one must be careful to use a depth of 0 (instead of 1) for a 2D CUDA image.
> If it\`s 1, then that\`s a 3D width x height x 1 texture, and may use a different layout (and will produce incorrect results if accessed using `surf2D()`.


# Final thoughts
It was a painful journey but alas I still managed to finish it.
What does CUDA actually bring here on the table? Well, a lot! We now have a way to encode and decode videos with CUVID / Video Codec SDK, trace scenes with OptiX if we ever expose Vertex/Index Buffers and easy to access objects in the scene info. And other stuff that can be done via CUDA Toolkit.

I know there is a lot to take in but this is the most feature complete post that I ever seen regarding interopability of CUDA with other APIs. I really hope that this comes handy to some one in search of some shining light at the end of this insanity hell-hole.

There's still a lot to cover regarding this topic that I had no time to try yet:

>**NOTE**
>
>[ID3D12Device3::OpenExistingHeapFromAddress](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device3-openexistingheapfromaddress) requires VirtualAlloc instead of malloc.
>AgilitySDK has that fixed with [ID3D12Device13::OpenExistingHeapFromAddress1](https://github.com/microsoft/DirectX-Headers/blob/main/include/directx/d3d12.idl#L5715) starting from around 1.610.0

1. Importing CPU memory into Graphics API via - [ID3D12Device3::OpenExistingHeapFromAddress](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device3-openexistingheapfromaddress) or [VkImportMemoryHostPointerInfoEXT](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkImportMemoryHostPointerInfoEXT.html) api call for Vulkan.
2. Importing GPU memory into DX11/DX12/Vulkan - I just glanced over this and never had enough time to actually test this.
3. Allocating shared non-dedicated memory - I probably will have to do this at some point but now assuming that non of the users will ever want to share lots of textures, I skipped this.
4. Memory allocator toggle so that every texture buffer will be allocated as shared memory - not sure if I'll do this, only the time will tell.
5. Sharing memory across GPUs - I am sure it's a broad topic and involves lots of synchronizations, memory management and other hard stuff to work correctly, so I am only interested in this for some research for now.
6. Transfering Vulkan's Queue Ownership - it seems like I never needed it for current implementation at least.
7. Wrapping DX12/DX11/Vulkan into own DLL, so I can issue some GPU Signal calls and kill the app, instead of rebooting whole PC, so I can continue working.
8. Win32 named handles
9. What `WindowsSecurityAttributes` actually does and how it affects interprocess interops


Theoretically this API now allows me to integrate any graphics API, albeit at the cost of running 2 separate drivers and syncing between them.
In theory I can friend any of the APIs below with the internal ones:
1. DirectX 11 (native Handle)
2. DirectX 12 (native Handle)
3. Vulkan (via KHR extensions)
4. OpenGL (via KHR extensions)
5. CUDA (native Handle/Posix FD)
6. OpenCL (not sure about this one, but may work in theory similar to CUDA)

This opens new doors for interopabilities! Now it's possible to integrate DirectX 11 interops, while the engine still running on DX12 or Vulkan internally!
The old way of doing things was to export `ID3D11Device`, `ID3D11Texture2D`, `ID3D11ShaderResourceView` and other similar pointers, while it was really easy and all sync work was done automatically by the drivers, now we have unified interfaces for that that have not that hard interfaces to grasp. Albeit they require a bit of understanding of underlaying memory layouts and allocations.

P.S. Optimization of synchronization path is left to the reader as an exercise.
