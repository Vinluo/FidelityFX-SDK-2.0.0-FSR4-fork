# FSR4 文件夹工作分析报告

## 概述

FSR4 (FidelityFX Super Resolution 4) 是 AMD 最新的基于机器学习的超分辨率技术实现。此 fork 包含了完整的 FSR4 实现，具有先进的神经网络模型和多种优化选项。

## 文件结构统计

- **总文件数**: 183 个文件
- **HLSL 着色器文件**: 38 个
- **二进制模型文件**: 12 个 (ML模型权重)
- **C++ 实现文件**: 2 个核心文件
- **头文件**: 10+ 个 API 定义文件

## 目录结构

```
Kits/FidelityFX/upscalers/fsr4/
├── dx12/                           # DirectX 12 后端实现
│   ├── ffx_provider_fsr4_dx12.cpp  # 核心 FSR4 提供者实现 (713行)
│   ├── BuildFSR4*.bat             # 构建脚本
│   └── ml2code_runtime/           # 机器学习运行时系统
│       ├── operators/             # ML 算子库 (100+ 算子)
│       │   ├── float16_NHWC/      # FP16 NHWC 格式算子
│       │   ├── float32/           # FP32 算子
│       │   ├── float8_NHWC/       # FP8 NHWC 格式算子
│       │   ├── int8_NHWC/         # INT8 NHWC 格式算子
│       │   └── uint8_NHWC/        # UINT8 NHWC 格式算子
│       └── tensor_*.hlsli         # 张量操作库
├── include/                       # API 头文件
│   ├── ffx_provider_fsr4.h       # FSR4 提供者接口
│   └── gpu/                      # GPU 相关定义
│       ├── fsr4/                 # FSR4 特定定义
│       │   ├── ffx_fsr4upscaler_resources.h  # 资源标识符
│       │   ├── ffx_fsr4_common_types.h       # 通用类型定义
│       │   └── *.hlsli           # GPU 着色器包含文件
│       └── spd/                  # SPD (Single Pass Downsampler) 支持
└── internal/                     # 内部实现
    ├── shader_selector.*         # 着色器选择系统
    ├── fsr4_permutations.h      # 着色器排列组合
    └── shaders/                 # 着色器集合
        ├── fsr4_model_v07_i8_*/ # INT8 模型变体 (6个质量预设)
        ├── fsr4_model_v07_fp8_*/ # FP8 模型变体
        ├── rcas.hlsl            # RCAS 锐化
        ├── spd_auto_exposure.hlsl # 自动曝光
        └── debug_view.hlsl      # 调试可视化
```

## 核心技术特性

### 1. 机器学习模型架构
- **模型版本**: v0.7 神经网络
- **量化支持**: INT8 和 FP8 两种精度
- **数据布局**: NHWC (批次-高度-宽度-通道) 优化
- **分辨率支持**: 1080p、2160p (4K)、4320p (8K) 专门优化

### 2. 质量预设系统 (6种)
1. **Native AA**: 原生抗锯齿模式
2. **Quality**: 质量模式 (最高画质)
3. **Balanced**: 平衡模式 (画质/性能平衡)
4. **Performance**: 性能模式 (优先帧率)
5. **DRS**: 动态分辨率缩放
6. **Ultra Performance**: 极致性能模式

### 3. 硬件优化特性
- **WMMA 支持**: Wave-level Matrix Multiply Accumulate 硬件加速
- **多精度支持**: FP16, FP32, INT8, FP8 数据类型
- **延迟销毁**: 智能管线资源管理 (5帧延迟销毁)
- **着色器排列**: 根据硬件能力动态选择最优实现

### 4. 颜色空间支持
- Linear (线性)
- PQ (Perceptual Quantizer)
- sRGB (标准 RGB)
- ACES (Academy Color Encoding System)
- OKLab (现代感知颜色空间)
- Reinhard/ReinhardSq 色调映射
- MuLaw 编码

### 5. 运动矢量处理
- **采样方法**: 默认、最近深度、带去遮挡的最近深度
- **去遮挡检测**: 无、SIE (Sony Interactive Entertainment) 方法
- **速度衰减**: 无、清零、SIE MV 抖动抑制

## ML2Code 运行时系统

FSR4 包含了完整的机器学习运行时系统，支持以下算子类型:

### 卷积算子
- **Conv2D**: 标准 2D 卷积 (多种核大小)
- **ConvTranspose2D**: 转置卷积 (上采样)
- **融合算子**: Conv+ReLU, Conv+Swish 等优化组合

### 激活函数
- ReLU, Sigmoid, LeakyReLU
- Swish, SqrSwish (平方 Swish)
- Sin, Cos, Log, Sqrt

### 池化和调整
- AveragePool2D, MaxPool2D
- Resize (双线性插值)
- Pad (填充)

### 数据操作
- Add, Mul, Div (逐元素运算)
- Concat, Split (张量拼接/分割)
- QuantizeLinear, DequantizeLinear (量化/反量化)

### 融合优化算子
- FusedConv2D_DW_Conv2D_PW_Relu_Conv2D_Add (深度可分离卷积块)
- FasterNetBlock, ConvNextBlock (现代网络架构块)
- 多种 Conv+激活函数的融合实现

## 资源管理系统

### 内部资源标识符 (25个)
- 输入资源: 颜色、运动矢量、深度
- 中间缓冲区: LUMA、历史数据、重投影历史
- 输出资源: MLSR 输出、RCAS 输出、最终输出
- ML 专用: NHWC 输入缓冲区、量化输出缓冲区、初始化器等

### 常量缓冲区 (5个)
1. FSR4UPSCALER 主常量
2. 自动曝光常量  
3. SPD 自动曝光常量
4. RCAS 锐化常量
5. 通道权重常量

## 构建和部署

### 自动化构建系统
- **BuildFSR4UpscalerShaders.bat**: 编译所有着色器变体
- **BuildFSR4Initializers.bat**: 生成 ML 模型初始化器
- **Visual Studio 项目**: 完整的开发环境集成

### 着色器排列管理
- 支持 1000+ 种着色器排列组合
- 基于硬件能力自动选择最优变体
- 运行时动态加载减少内存占用

## 技术创新点

1. **混合精度推理**: INT8/FP8 量化显著提升性能
2. **分辨率自适应**: 针对不同分辨率的专门优化
3. **硬件感知**: WMMA 等现代 GPU 特性充分利用  
4. **融合算子**: 减少内存带宽和计算开销
5. **智能资源管理**: 延迟销毁避免渲染卡顿

## 与 FSR3 的主要区别

1. **算法核心**: FSR4 使用深度神经网络，FSR3 使用传统时域算法
2. **质量提升**: ML 模型提供更好的细节保持和伪影抑制
3. **硬件需求**: FSR4 需要支持 WMMA 的现代 GPU
4. **内存占用**: 需要加载 ML 模型权重 (约 12 个 .bin 文件)
5. **性能特征**: 计算密集型 vs 带宽密集型

## 开发状态

- ✅ **核心框架**: 完整实现 (713 行主要代码)
- ✅ **ML 运行时**: 100+ 算子完整实现
- ✅ **多质量预设**: 6种质量级别全部支持
- ✅ **多分辨率优化**: 1080p/2160p/4320p 优化完成
- ✅ **DirectX 12 后端**: 完整的 DX12 集成
- ✅ **构建系统**: 自动化编译和部署
- ✅ **API 层**: 完整的提供者接口

## 总结

这个 FSR4 实现代表了超分辨率技术的重大进步，从传统的时域算法转向基于机器学习的方法。实现质量高、特性完整，包含了现代 GPU 硬件的深度优化，是一个生产就绪的完整解决方案。

相比传统的 FSR2/FSR3，FSR4 在画质上有显著提升，同时保持了良好的性能表现，特别是在支持 WMMA 的现代 GPU 上。这个实现为游戏开发者提供了一个强大的 AI 超分辨率工具。