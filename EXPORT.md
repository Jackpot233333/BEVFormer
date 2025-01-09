
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

---
对 SpatialCrossAttention 中的动态 shape 的补充：

在 SpatialCrossAttention 中 query 会先经过一个由 lidar2image 得出的 mask，将需要的点取出，来得到真正进入 deform attn 的 queries_rebatch

通常来说 6v 各个相机朝向固定，在 bevfeature 中不可能占满，单目 query 经过 mask 后的实际 queries_rebatch 长度相比原始 query 长度会大大减小；根据几组 nuscenes 数据集数据简单统计，大约为 30% 左右。即 50*50 的 bev feature，query 长度为 2500，mask 后最大长度都在 600 以下； 100 * 100 的 bev feature，query 长度为 10000，mask 后长度都在 3000 以下

这里 queries_rebatch 的长度是由 lidar2image 来决定的，所以当 lidar2image 是模型输入时，这里的 tensor shape 是动态的，整个模型是一个动态 shape 模型，动态 shape 模型我们是不支持的

目前的改法是不把经过 mask 得到的点取出，而是在原始 query 上把本该被 mask 掉的点置 0，queries_rebatch 的长度就是原始 query 的长度

如果能保证 lidar2image 是静态，将其作为常数而不是模型输入，queries_rebatch 的长度就可以认为是固定，从而使送入 deform attn 的长度为 mask 后而非原始 query 长度，减小计算量，节省模型运行时间

具体改法是把之前为了静态 shape 而对 SpatialCrossAttention 做的修改全部撤销即可，同时在 encoder.point_sampling 中，将原本从外部输入来的 `lidar2img = img_metas[0]['lidar2img']` 改成具体值（比如写死或者从某处 load）

注意！上述速度优化需要保证 lidar2img 静态，一旦 lidar2img 改变，需要重新编译重新部署。因此需要明确实际落地车上 lidar2img 是否为静态，且同一批车互相之间 lidar2img 是否有不同

![image](https://github.com/user-attachments/assets/c7075b5c-7f12-46c4-8d33-2b9c0ba70bf1)
