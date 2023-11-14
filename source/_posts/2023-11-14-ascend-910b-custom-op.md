---
title: Ascend 910B 自定义 PyTorch 算子
date: 2023-11-14
lang: zh-CN
tags:
  - linux
  - pytorch
  - ascend
  - machine-learning
---

## 环境

本文基于的硬件环境为 Ascend 910B3，基于的软件环境包括 [CANN 7.0-RC1](https://www.hiascend.com/developer/download/community/result)、[PyTorch 1.11.0](https://repo.huaweicloud.com/kunpeng/archive/Ascend/PyTorch/)、[Ascend PyTorch Adapter v5.0.rc3-pytorch1.11.0](https://gitee.com/ascend/pytorch/releases/tag/v5.0.rc3-pytorch1.11.0)。其他 CANN 和 PyTorch 版本上的情况可能略有不同。

## 注册过程

### Ascend PyTorch Adapter 中添加自定义算子

> 参考：
> - https://www.hiascend.com/document/detail/zh/canncommercial/70RC1/operatordev/Ascendcopdevg/atlas_ascendc_10_0045.html
> - https://gitee.com/ascend/samples/tree/master/operator/AddCustomSample/FrameworkLaunch/PytorchInvocation

在 `torch_npu/csrc/aten/npu_native_functions.yaml` 中添加 `npu_add_custom` 函数：

```yaml
custom:
  - func: npu_add_custom(Tensor x, Tensor y) -> Tensor  # 添加的函数
```

在 `torch_npu/csrc/aten/ops/op_api` 中添加 `AddCustomKernelNpu.cpp` 文件：

```c++
#include <torch/csrc/autograd/custom_function.h>

#include "torch_npu/csrc/framework/utils/OpAdapter.h"
#include "torch_npu/csrc/aten/NPUNativeFunctions.h"
#include "torch_npu/csrc/aten/ops/op_api/op_api_common.h"

namespace at_npu {
  namespace native {
    using torch::autograd::Function;
    using torch::autograd::AutogradContext;

    at::Tensor NPUNativeFunctions::npu_add_custom(const at::Tensor& x, const at::Tensor& y) {
        at::Tensor result = OpPreparation::ApplyTensor(x); // 创建输出内存

        // calculate the output result of the NPU
        EXEC_NPU_CMD(aclnnAddCustom, x, y, result);
        return result;
    }
  } // namespace native
} // namespace at_npu

```

之后重新编译安装 `torch_npu`。

### CANN 中添加自定义算子的实现

> 参考：
> - https://www.hiascend.com/document/detail/zh/canncommercial/70RC1/operatordev/Ascendcopdevg/atlas_ascendc_10_0023.html

首先定义算子描述文件 `add_custom.json`：

```json
[
    {
        "op": "AddCustom",
        "language": "cpp",
        "input_desc": [
            {
                "name": "x",
                "param_type": "required",
                "format": [
                    "ND"
                ],
                "type": [
                    "fp16"
                ]
            },
            {
                "name": "y",
                "param_type": "required",
                "format": [
                    "ND"
                ],
                "type": [
                    "fp16"
                ]
            }
        ],
        "output_desc": [
            {
                "name": "z",
                "param_type": "required",
                "format": [
                    "ND"
                ],
                "type": [
                    "fp16"
                ]
            }
        ]
    }
]
```

执行

```shell
msopgen gen -i add_custom.json -c ai_core-Ascend910B3 -f pytorch -out . -lan cpp
```

生成算子工程：

```txt
AddCustom
├── build.sh
├── cmake 
│   ├── config.cmake
│   ├── func.cmake
│   ├── intf.cmake
│   ├── makeself.cmake
│   └── util
├── CMakeLists.txt
├── CMakePresets.json          // 修改 ASCEND_CANN_PACKAGE_PATH
├── framework
├── op_host
│   ├── add_custom_tiling.h    // 定义 length 和 tiling 相关信息
│   ├── add_custom.cpp
│   ├── CMakeLists.txt
├── op_kernel
│   ├── CMakeLists.txt
│   ├── add_custom.cpp         // 算子实现
└── scripts
```

`CMakePresets.json` 中修改 `ASCEND_CANN_PACKAGE_PATH` 为 CANN 安装路径。

`op_host/add_custom_tiling.h` 的内容如下（简单实现）：

```c++
#include "register/tilingdata_base.h"

namespace optiling {
BEGIN_TILING_DATA_DEF(AddCustomTilingData)
    TILING_DATA_FIELD_DEF(uint32_t, size);  // 定义 tensor size
END_TILING_DATA_DEF;

REGISTER_TILING_DATA_CLASS(AddCustom, AddCustomTilingData)
}
```

`op_kernel/add_custom.cpp` 是算子的具体实现：

```c++

#include "kernel_operator.h"

#ifdef __DAV_C220_VEC__

extern "C" __global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z, GM_ADDR workspace, GM_ADDR tiling) {
    GET_TILING_DATA(tiling_data, tiling);
    uint32_t M = tiling_data.size;  // 从 tiling_data 中获取 tensor size

    // ...
}

#else

// 重要：CANN 会尝试不同的 ccec 编译参数以推断算子的类型（VEC、CUBE、MIXED），如果不创建一个 stub 函数将会编译失败
extern "C" __global__ __aicore__ void add_custom(GM_ADDR x, GM_ADDR y, GM_ADDR z, GM_ADDR workspace, GM_ADDR tiling) {
    pip_barrier(PIPE_ALL);
}

#endif
```

### 编译部署

```shell
$ bash build.sh
$ ./custom_opp_euleros_aarch64.run
```

PyTorch 中调用：

```python
import torch
import torch_npu

# ...

z = torch.npu_add_custom(x, y)  # 由于是运行时编译，第一次运行时需要等待编译
```

## 注册原理

TODO

## 参考

TODO
