
# 第一步： [commit](https://github.com/Jackpot233333/BEVFormer/commit/a1e59481d5541d81ffe0db13a196131c82a0a13c)
首先对模型进行初步修改，使其能够跑通。修改内容包括以下几个部分：
- 由于 [Installation](docs/install.md) 中指定了 torch 版本，不支持 opset 16，使用 register_custom_op_symbolic 和 parse_args 来实现导出 grid_sample
- 修改 onnx 不支持的算子，如 maxmun、linspace 等
- 将 lidar2img、img_shape 等原本是模型输入的参数改成常数，当前改法比较暴力，仅供参考
- 将 get_bboxes 等后处理移除
- 实现简易的 export_onnx 函数导出模型

修改后可以跑通 `python3 tools/test.py ./projects/configs/bevformer/bevformer_tiny.py ./outputs/bevformer_tiny.onnx` 
得到 onnx 再使用 onnxsim 优化

# 第二步： [commit](https://github.com/Jackpot233333/BEVFormer/commit/122d89e5f5b9a785453af60fa6d57330df45cf59)
这里针对 axera 的后端进行一些优化，试其能够上板。修改内容包括以下几个部分：
- SpatialCrossAttention 中的动态 shape 不支持，改成了静态 shape；这导致计算量变大了，可能还有进一步优化空间
- 移除 inverse sigmoid 操作，这也是后端不支持的部分，改成需要时才做 sigmoid，而不是先做了 sigmoid 再 inverse
- 一些切片操作由于其写法会导出成 ScatterND，虽然 ScatterND 现在支持了，效率肯定不如 Slice，因此一起修改了

修改后可以跑通编译流程

# 第三步： [commit](https://github.com/Jackpot233333/BEVFormer/commit/c03c63fbab6ab75a4c809f42e5ea2b7a6bc1f038)
将 MultiScaleDeformableAttn 包成一个自定义算子，这个算子在 Axera 工具链上是高效支持的