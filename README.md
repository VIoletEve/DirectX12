# DirectX12
***
## Ssao

首先创建一个Ssao辅助类
头文件详情如下

```

#ifndef SSAO_H
#define SSAO_H

#pragma once

#include "../../Common/d3dUtil.h"
#include "FrameResource.h"
 
 
class Ssao
{
public:

	Ssao(ID3D12Device* device, 
        ID3D12GraphicsCommandList* cmdList, 
        UINT width, UINT height);
    Ssao(const Ssao& rhs) = delete;
    Ssao& operator=(const Ssao& rhs) = delete;
    ~Ssao() = default; 

    //创建相应的srv时需要相应的格式
    static const DXGI_FORMAT AmbientMapFormat = DXGI_FORMAT_R16_UNORM;
    static const DXGI_FORMAT NormalMapFormat = DXGI_FORMAT_R16G16B16A16_FLOAT;

    //模糊运算需要用的最大模糊半径
    static const int MaxBlurRadius = 5;

	UINT SsaoMapWidth()const;
    UINT SsaoMapHeight()const;

    //传入一个数组得到偏移向量
    void GetOffsetVectors(DirectX::XMFLOAT4 offsets[14]);
    //计算高斯模糊
    std::vector<float> CalcGaussWeights(float sigma);

	ID3D12Resource* NormalMap();
	ID3D12Resource* AmbientMap();
	
    //返回描述符地址
    CD3DX12_CPU_DESCRIPTOR_HANDLE NormalMapRtv()const;
	CD3DX12_GPU_DESCRIPTOR_HANDLE NormalMapSrv()const;
    CD3DX12_GPU_DESCRIPTOR_HANDLE AmbientMapSrv()const;

    //传入cpu和gpu的Srv堆描述符句柄，传入Rtv堆的描述符句柄，并且将它们在堆中偏移到正确的位置
	void BuildDescriptors(
        ID3D12Resource* depthStencilBuffer,
		CD3DX12_CPU_DESCRIPTOR_HANDLE hCpuSrv,
		CD3DX12_GPU_DESCRIPTOR_HANDLE hGpuSrv,
		CD3DX12_CPU_DESCRIPTOR_HANDLE hCpuRtv,
        UINT cbvSrvUavDescriptorSize,
        UINT rtvDescriptorSize);

    //创建各个资源的描述符，并放入堆中
    void RebuildDescriptors(ID3D12Resource* depthStencilBuffer);

    void SetPSOs(ID3D12PipelineState* ssaoPso, ID3D12PipelineState* ssaoBlurPso);

	void OnResize(UINT newWidth, UINT newHeight);
  
    //将渲染目标设置为AO图，并且绘制一个全屏幕的AO图
    //主深度缓冲区绑定到管道，但读写深度缓冲区是禁用的，我们不需要计算AO图的深度缓冲区
	void ComputeSsao(
        ID3D12GraphicsCommandList* cmdList, 
        FrameResource* currFrame, 
        int blurCount);
 

private:
 
    //模糊AO图，使用边缘保持模糊，我们不模糊不连续点，让边缘不受影响
    void BlurAmbientMap(ID3D12GraphicsCommandList* cmdList, FrameResource* currFrame, int blurCount);
	  void BlurAmbientMap(ID3D12GraphicsCommandList* cmdList, bool horzBlur);

    void BuildResources();
    void BuildRandomVectorTexture(ID3D12GraphicsCommandList* cmdList);
 
	  void BuildOffsetVectors();


private:
	  ID3D12Device* md3dDevice;

    Microsoft::WRL::ComPtr<ID3D12RootSignature> mSsaoRootSig;
    
    ID3D12PipelineState* mSsaoPso = nullptr;
    ID3D12PipelineState* mBlurPso = nullptr;
	 
    Microsoft::WRL::ComPtr<ID3D12Resource> mRandomVectorMap;
	  Microsoft::WRL::ComPtr<ID3D12Resource> mRandomVectorMapUploadBuffer;
    Microsoft::WRL::ComPtr<ID3D12Resource> mNormalMap;
    Microsoft::WRL::ComPtr<ID3D12Resource> mAmbientMap0;
    Microsoft::WRL::ComPtr<ID3D12Resource> mAmbientMap1; 

    CD3DX12_CPU_DESCRIPTOR_HANDLE mhNormalMapCpuSrv;
    CD3DX12_GPU_DESCRIPTOR_HANDLE mhNormalMapGpuSrv;
    CD3DX12_CPU_DESCRIPTOR_HANDLE mhNormalMapCpuRtv;

    CD3DX12_CPU_DESCRIPTOR_HANDLE mhDepthMapCpuSrv;
    CD3DX12_GPU_DESCRIPTOR_HANDLE mhDepthMapGpuSrv;

    CD3DX12_CPU_DESCRIPTOR_HANDLE mhRandomVectorMapCpuSrv;
    CD3DX12_GPU_DESCRIPTOR_HANDLE mhRandomVectorMapGpuSrv;

    // Need two for ping-ponging during blur.
    CD3DX12_CPU_DESCRIPTOR_HANDLE mhAmbientMap0CpuSrv;
    CD3DX12_GPU_DESCRIPTOR_HANDLE mhAmbientMap0GpuSrv;
    CD3DX12_CPU_DESCRIPTOR_HANDLE mhAmbientMap0CpuRtv;

    CD3DX12_CPU_DESCRIPTOR_HANDLE mhAmbientMap1CpuSrv;
    CD3DX12_GPU_DESCRIPTOR_HANDLE mhAmbientMap1GpuSrv;
    CD3DX12_CPU_DESCRIPTOR_HANDLE mhAmbientMap1CpuRtv;

	  UINT mRenderTargetWidth;
	  UINT mRenderTargetHeight;

    DirectX::XMFLOAT4 mOffsets[14];

	  D3D12_VIEWPORT mViewport;
	  D3D12_RECT mScissorRect;
};

#endif // SSAO_H

```
***
Ssao辅助类Cpp文件如下:
```


#include "Ssao.h"
#include <DirectXPackedVector.h>

using namespace DirectX;
using namespace DirectX::PackedVector;
using namespace Microsoft::WRL;

Ssao::Ssao(
    ID3D12Device* device,
    ID3D12GraphicsCommandList* cmdList, 
    UINT width, UINT height)

{
    md3dDevice = device;

    OnResize(width, height);

	BuildOffsetVectors();
	BuildRandomVectorTexture(cmdList);
}

UINT Ssao::SsaoMapWidth()const
{
    return mRenderTargetWidth / 2;
}

UINT Ssao::SsaoMapHeight()const
{
    return mRenderTargetHeight / 2;
}

void Ssao::GetOffsetVectors(DirectX::XMFLOAT4 offsets[14])
{
    std::copy(&mOffsets[0], &mOffsets[14], &offsets[0]);
}

std::vector<float> Ssao::CalcGaussWeights(float sigma)
{
    float twoSigma2 = 2.0f*sigma*sigma;

    // Estimate the blur radius based on sigma since sigma controls the "width" of the bell curve.
    // For example, for sigma = 3, the width of the bell curve is 
    int blurRadius = (int)ceil(2.0f * sigma);

    assert(blurRadius <= MaxBlurRadius);

    std::vector<float> weights;
    weights.resize(2 * blurRadius + 1);

    float weightSum = 0.0f;

    for(int i = -blurRadius; i <= blurRadius; ++i)
    {
        float x = (float)i;

        weights[i + blurRadius] = expf(-x*x / twoSigma2);

        weightSum += weights[i + blurRadius];
    }

    // Divide by the sum so all the weights add up to 1.0.
    for(int i = 0; i < weights.size(); ++i)
    {
        weights[i] /= weightSum;
    }

    return weights;
}

ID3D12Resource* Ssao::NormalMap()
{
    return mNormalMap.Get();
}

ID3D12Resource* Ssao::AmbientMap()
{
    return mAmbientMap0.Get();
}

CD3DX12_CPU_DESCRIPTOR_HANDLE Ssao::NormalMapRtv()const
{
    return mhNormalMapCpuRtv;
}

CD3DX12_GPU_DESCRIPTOR_HANDLE Ssao::NormalMapSrv()const
{
    return mhNormalMapGpuSrv;
}

CD3DX12_GPU_DESCRIPTOR_HANDLE Ssao::AmbientMapSrv()const
{
    return mhAmbientMap0GpuSrv;
}

void Ssao::BuildDescriptors(
    ID3D12Resource* depthStencilBuffer,
    CD3DX12_CPU_DESCRIPTOR_HANDLE hCpuSrv,
    CD3DX12_GPU_DESCRIPTOR_HANDLE hGpuSrv,
    CD3DX12_CPU_DESCRIPTOR_HANDLE hCpuRtv,
    UINT cbvSrvUavDescriptorSize,
    UINT rtvDescriptorSize)
{
    //各个描述符句柄在堆中的位置

    mhAmbientMap0CpuSrv = hCpuSrv;
    mhAmbientMap1CpuSrv = hCpuSrv.Offset(1, cbvSrvUavDescriptorSize);
    mhNormalMapCpuSrv = hCpuSrv.Offset(1, cbvSrvUavDescriptorSize);
    mhDepthMapCpuSrv = hCpuSrv.Offset(1, cbvSrvUavDescriptorSize);
    mhRandomVectorMapCpuSrv = hCpuSrv.Offset(1, cbvSrvUavDescriptorSize);

    mhAmbientMap0GpuSrv = hGpuSrv;
    mhAmbientMap1GpuSrv = hGpuSrv.Offset(1, cbvSrvUavDescriptorSize);
    mhNormalMapGpuSrv = hGpuSrv.Offset(1, cbvSrvUavDescriptorSize);
    mhDepthMapGpuSrv = hGpuSrv.Offset(1, cbvSrvUavDescriptorSize);
    mhRandomVectorMapGpuSrv = hGpuSrv.Offset(1, cbvSrvUavDescriptorSize);

    mhNormalMapCpuRtv = hCpuRtv;
    mhAmbientMap0CpuRtv = hCpuRtv.Offset(1, rtvDescriptorSize);
    mhAmbientMap1CpuRtv = hCpuRtv.Offset(1, rtvDescriptorSize);

    //创建描述符，并把放入堆中
    RebuildDescriptors(depthStencilBuffer);
}

void Ssao::RebuildDescriptors(ID3D12Resource* depthStencilBuffer)
{
    //创建normalMap的srv，并通过描述符句柄放入堆中
    D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
    srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
    srvDesc.ViewDimension = D3D12_SRV_DIMENSION_TEXTURE2D;
    srvDesc.Format = NormalMapFormat;
    srvDesc.Texture2D.MostDetailedMip = 0;
    srvDesc.Texture2D.MipLevels = 1;
    md3dDevice->CreateShaderResourceView(mNormalMap.Get(), &srvDesc, mhNormalMapCpuSrv);

    //创建深度/模板缓冲区的srv，放入堆中
    srvDesc.Format = DXGI_FORMAT_R24_UNORM_X8_TYPELESS;
    md3dDevice->CreateShaderResourceView(depthStencilBuffer, &srvDesc, mhDepthMapCpuSrv);

    //创建随机向量的srv，放入堆中
    srvDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
    md3dDevice->CreateShaderResourceView(mRandomVectorMap.Get(), &srvDesc, mhRandomVectorMapCpuSrv);

    //创建AO图的srv，放入堆中
    srvDesc.Format = AmbientMapFormat;
    md3dDevice->CreateShaderResourceView(mAmbientMap0.Get(), &srvDesc, mhAmbientMap0CpuSrv);
    md3dDevice->CreateShaderResourceView(mAmbientMap1.Get(), &srvDesc, mhAmbientMap1CpuSrv);

    //创建各个资源的rtv，放入堆中
    D3D12_RENDER_TARGET_VIEW_DESC rtvDesc = {};
    rtvDesc.ViewDimension = D3D12_RTV_DIMENSION_TEXTURE2D;
    rtvDesc.Format = NormalMapFormat;
    rtvDesc.Texture2D.MipSlice = 0;
    rtvDesc.Texture2D.PlaneSlice = 0;
    md3dDevice->CreateRenderTargetView(mNormalMap.Get(), &rtvDesc, mhNormalMapCpuRtv);

    rtvDesc.Format = AmbientMapFormat;
    md3dDevice->CreateRenderTargetView(mAmbientMap0.Get(), &rtvDesc, mhAmbientMap0CpuRtv);
    md3dDevice->CreateRenderTargetView(mAmbientMap1.Get(), &rtvDesc, mhAmbientMap1CpuRtv);
}

void Ssao::SetPSOs(ID3D12PipelineState* ssaoPso, ID3D12PipelineState* ssaoBlurPso)
{
    //传入pso
    mSsaoPso = ssaoPso;
    mBlurPso = ssaoBlurPso;
}

void Ssao::OnResize(UINT newWidth, UINT newHeight)
{
    if(mRenderTargetWidth != newWidth || mRenderTargetHeight != newHeight)
    {
        mRenderTargetWidth = newWidth;
        mRenderTargetHeight = newHeight;

        // 渲染一半的分辨率
        mViewport.TopLeftX = 0.0f;
        mViewport.TopLeftY = 0.0f;
        mViewport.Width = mRenderTargetWidth / 2.0f;
        mViewport.Height = mRenderTargetHeight / 2.0f;
        mViewport.MinDepth = 0.0f;
        mViewport.MaxDepth = 1.0f;

        mScissorRect = { 0, 0, (int)mRenderTargetWidth / 2, (int)mRenderTargetHeight / 2 };
        //根据新的宽高重新创造资源
        BuildResources();
    }
}

void Ssao::ComputeSsao(
    ID3D12GraphicsCommandList* cmdList,
    FrameResource* currFrame, 
    int blurCount)
{
	cmdList->RSSetViewports(1, &mViewport);
    cmdList->RSSetScissorRects(1, &mScissorRect);

	// 先绘制一个初始ssao到AO图0

    // 将AO图0资源状态变为渲染目标状态
    cmdList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(mAmbientMap0.Get(),
        D3D12_RESOURCE_STATE_GENERIC_READ, D3D12_RESOURCE_STATE_RENDER_TARGET));
  
	float clearValue[] = {1.0f, 1.0f, 1.0f, 1.0f};
    cmdList->ClearRenderTargetView(mhAmbientMap0CpuRtv, clearValue, 0, nullptr);
     
	// 设置渲染目标
    cmdList->OMSetRenderTargets(1, &mhAmbientMap0CpuRtv, true, nullptr);

    // 绑定ssao的常量缓冲区
    auto ssaoCBAddress = currFrame->SsaoCB->Resource()->GetGPUVirtualAddress();
    cmdList->SetGraphicsRootConstantBufferView(0, ssaoCBAddress);
    cmdList->SetGraphicsRoot32BitConstant(1, 0, 0);

	// 绑定法线图和深度图
    cmdList->SetGraphicsRootDescriptorTable(2, mhNormalMapGpuSrv);

    // 绑定随机向量图
    cmdList->SetGraphicsRootDescriptorTable(3, mhRandomVectorMapGpuSrv);

    cmdList->SetPipelineState(mSsaoPso);

	// 绘制全屏四边形，由两个三角形组成
	cmdList->IASetVertexBuffers(0, 0, nullptr);
    cmdList->IASetIndexBuffer(nullptr);
    cmdList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
	cmdList->DrawInstanced(6, 1, 0, 0);
   
	// 改回通用可读，方便在着色器中使用
    cmdList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(mAmbientMap0.Get(),
        D3D12_RESOURCE_STATE_RENDER_TARGET, D3D12_RESOURCE_STATE_GENERIC_READ));

    //模糊AO图
    BlurAmbientMap(cmdList, currFrame, blurCount);
}
 
void Ssao::BlurAmbientMap(ID3D12GraphicsCommandList* cmdList, FrameResource* currFrame, int blurCount)
{
    cmdList->SetPipelineState(mBlurPso);

    auto ssaoCBAddress = currFrame->SsaoCB->Resource()->GetGPUVirtualAddress();
    cmdList->SetGraphicsRootConstantBufferView(0, ssaoCBAddress);
 
    for(int i = 0; i < blurCount; ++i)
    {
        BlurAmbientMap(cmdList, true);
        BlurAmbientMap(cmdList, false);
    }
}

void Ssao::BlurAmbientMap(ID3D12GraphicsCommandList* cmdList, bool horzBlur)
{
	ID3D12Resource* output = nullptr;
	CD3DX12_GPU_DESCRIPTOR_HANDLE inputSrv;
	CD3DX12_CPU_DESCRIPTOR_HANDLE outputRtv;
	
	// Ping-pong the two ambient map textures as we apply
	// horizontal and vertical blur passes.
	if(horzBlur == true)
	{
		output = mAmbientMap1.Get();
		inputSrv = mhAmbientMap0GpuSrv;
		outputRtv = mhAmbientMap1CpuRtv;
        cmdList->SetGraphicsRoot32BitConstant(1, 1, 0);
	}
	else
	{
		output = mAmbientMap0.Get();
		inputSrv = mhAmbientMap1GpuSrv;
		outputRtv = mhAmbientMap0CpuRtv;
        cmdList->SetGraphicsRoot32BitConstant(1, 0, 0);
	}
 
    cmdList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(output,
        D3D12_RESOURCE_STATE_GENERIC_READ, D3D12_RESOURCE_STATE_RENDER_TARGET));

	float clearValue[] = { 1.0f, 1.0f, 1.0f, 1.0f };
    cmdList->ClearRenderTargetView(outputRtv, clearValue, 0, nullptr);
 
    cmdList->OMSetRenderTargets(1, &outputRtv, true, nullptr);
	
    // Normal/depth map still bound.


    // Bind the normal and depth maps.
    cmdList->SetGraphicsRootDescriptorTable(2, mhNormalMapGpuSrv);

    // Bind the input ambient map to second texture table.
    cmdList->SetGraphicsRootDescriptorTable(3, inputSrv);
	
	// Draw fullscreen quad.
	cmdList->IASetVertexBuffers(0, 0, nullptr);
    cmdList->IASetIndexBuffer(nullptr);
    cmdList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
	cmdList->DrawInstanced(6, 1, 0, 0);
   
    cmdList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(output,
        D3D12_RESOURCE_STATE_RENDER_TARGET, D3D12_RESOURCE_STATE_GENERIC_READ));
}
 
void Ssao::BuildResources()
{
	// 释放旧资源，如果存在
    mNormalMap = nullptr;
    mAmbientMap0 = nullptr;
    mAmbientMap1 = nullptr;

    //创建法线图资源
    D3D12_RESOURCE_DESC texDesc;
    ZeroMemory(&texDesc, sizeof(D3D12_RESOURCE_DESC));
    texDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
    texDesc.Alignment = 0;
    texDesc.Width = mRenderTargetWidth;
    texDesc.Height = mRenderTargetHeight;
    texDesc.DepthOrArraySize = 1;
    texDesc.MipLevels = 1;
    texDesc.Format = Ssao::NormalMapFormat;
    texDesc.SampleDesc.Count = 1;
    texDesc.SampleDesc.Quality = 0;
    texDesc.Layout = D3D12_TEXTURE_LAYOUT_UNKNOWN;
    texDesc.Flags = D3D12_RESOURCE_FLAG_ALLOW_RENDER_TARGET;


    float normalClearColor[] = { 0.0f, 0.0f, 1.0f, 0.0f };
    CD3DX12_CLEAR_VALUE optClear(NormalMapFormat, normalClearColor);
    ThrowIfFailed(md3dDevice->CreateCommittedResource(
        &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
        D3D12_HEAP_FLAG_NONE,
        &texDesc,
		D3D12_RESOURCE_STATE_GENERIC_READ,
        &optClear,
        IID_PPV_ARGS(&mNormalMap)));

	// AO图取一半分辨率
    texDesc.Width = mRenderTargetWidth / 2;
    texDesc.Height = mRenderTargetHeight / 2;
    texDesc.Format = Ssao::AmbientMapFormat;

    float ambientClearColor[] = { 1.0f, 1.0f, 1.0f, 1.0f };
    optClear = CD3DX12_CLEAR_VALUE(AmbientMapFormat, ambientClearColor);

    //创建AO图资源
    ThrowIfFailed(md3dDevice->CreateCommittedResource(
        &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
        D3D12_HEAP_FLAG_NONE,
        &texDesc,
        D3D12_RESOURCE_STATE_GENERIC_READ,
        &optClear,
        IID_PPV_ARGS(&mAmbientMap0)));

    ThrowIfFailed(md3dDevice->CreateCommittedResource(
        &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
        D3D12_HEAP_FLAG_NONE,
        &texDesc,
        D3D12_RESOURCE_STATE_GENERIC_READ,
        &optClear,
        IID_PPV_ARGS(&mAmbientMap1)));
}

void Ssao::BuildRandomVectorTexture(ID3D12GraphicsCommandList* cmdList)
{
    //创建随机向量纹理
    D3D12_RESOURCE_DESC texDesc;
    ZeroMemory(&texDesc, sizeof(D3D12_RESOURCE_DESC));
    texDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
    texDesc.Alignment = 0;
    texDesc.Width = 256;
    texDesc.Height = 256;
    texDesc.DepthOrArraySize = 1;
    texDesc.MipLevels = 1;
    texDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
    texDesc.SampleDesc.Count = 1;
    texDesc.SampleDesc.Quality = 0;
    texDesc.Layout = D3D12_TEXTURE_LAYOUT_UNKNOWN;
    texDesc.Flags = D3D12_RESOURCE_FLAG_NONE;

    ThrowIfFailed(md3dDevice->CreateCommittedResource(
        &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
        D3D12_HEAP_FLAG_NONE,
        &texDesc,
        D3D12_RESOURCE_STATE_GENERIC_READ,
        nullptr,
        IID_PPV_ARGS(&mRandomVectorMap)));
    
    //为了将cpu内存中的数据复制到随机向量纹理的默认堆中，我们需要创建一个上传堆
    //为此需要计算资源中子资源的数量
    const UINT num2DSubresources = texDesc.DepthOrArraySize * texDesc.MipLevels;
    //然后通过GetRequiredIntermediateSize得到上传堆的大小
    const UINT64 uploadBufferSize = GetRequiredIntermediateSize(mRandomVectorMap.Get(), 0, num2DSubresources);

    //创建随机向量资源的上传堆
    ThrowIfFailed(md3dDevice->CreateCommittedResource(
        &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
        D3D12_HEAP_FLAG_NONE,
        &CD3DX12_RESOURCE_DESC::Buffer(uploadBufferSize),
        D3D12_RESOURCE_STATE_GENERIC_READ,
        nullptr,
        IID_PPV_ARGS(mRandomVectorMapUploadBuffer.GetAddressOf())));

    //创建初始化随机向量数据
    XMCOLOR initData[256 * 256];
    for(int i = 0; i < 256; ++i)
    {
        for(int j = 0; j < 256; ++j)
        {
			// 随机向量在[0,1],我们在shader中会将它映射到[-1,1]
            XMFLOAT3 v(MathHelper::RandF(), MathHelper::RandF(), MathHelper::RandF());

            initData[i * 256 + j] = XMCOLOR(v.x, v.y, v.z, 0.0f);
        }
    }

    //创建子资源
    D3D12_SUBRESOURCE_DATA subResourceData = {};
    subResourceData.pData = initData;
    subResourceData.RowPitch = 256 * sizeof(XMCOLOR);
    subResourceData.SlicePitch = subResourceData.RowPitch * 256;

    
    //将资源变为复制目标状态
    cmdList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(mRandomVectorMap.Get(),
        D3D12_RESOURCE_STATE_GENERIC_READ, D3D12_RESOURCE_STATE_COPY_DEST));
    //然后上传更新子资源，将子资源的数据复制到上传堆，再由上传堆传入默认堆，以更新随机向量资源
    UpdateSubresources(cmdList, mRandomVectorMap.Get(), mRandomVectorMapUploadBuffer.Get(),
		0, 0, num2DSubresources, &subResourceData);
    //然后再将资源状态改为通用可读，方便在着色器使用
    cmdList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(mRandomVectorMap.Get(),
        D3D12_RESOURCE_STATE_COPY_DEST, D3D12_RESOURCE_STATE_GENERIC_READ));
}
 
void Ssao::BuildOffsetVectors()
{
    // 我们选择以点为中心的立方体的8个顶点，以及各个面的中心作为偏移向量
	
	//8个立方体顶点
	mOffsets[0] = XMFLOAT4(+1.0f, +1.0f, +1.0f, 0.0f);
	mOffsets[1] = XMFLOAT4(-1.0f, -1.0f, -1.0f, 0.0f);

	mOffsets[2] = XMFLOAT4(-1.0f, +1.0f, +1.0f, 0.0f);
	mOffsets[3] = XMFLOAT4(+1.0f, -1.0f, -1.0f, 0.0f);

	mOffsets[4] = XMFLOAT4(+1.0f, +1.0f, -1.0f, 0.0f);
	mOffsets[5] = XMFLOAT4(-1.0f, -1.0f, +1.0f, 0.0f);

	mOffsets[6] = XMFLOAT4(-1.0f, +1.0f, -1.0f, 0.0f);
	mOffsets[7] = XMFLOAT4(+1.0f, -1.0f, +1.0f, 0.0f);

	// 6个立方体面中心点
	mOffsets[8] = XMFLOAT4(-1.0f, 0.0f, 0.0f, 0.0f);
	mOffsets[9] = XMFLOAT4(+1.0f, 0.0f, 0.0f, 0.0f);

	mOffsets[10] = XMFLOAT4(0.0f, -1.0f, 0.0f, 0.0f);
	mOffsets[11] = XMFLOAT4(0.0f, +1.0f, 0.0f, 0.0f);

	mOffsets[12] = XMFLOAT4(0.0f, 0.0f, -1.0f, 0.0f);
	mOffsets[13] = XMFLOAT4(0.0f, 0.0f, +1.0f, 0.0f);

    for(int i = 0; i < 14; ++i)
	{
		// 偏移向量的长度为0.25到1的随机值
		float s = MathHelper::RandF(0.25f, 1.0f);
		
		XMVECTOR v = s * XMVector4Normalize(XMLoadFloat4(&mOffsets[i]));
		//存到偏移向量数组中
		XMStoreFloat4(&mOffsets[i], v);
	}
}

```
***
