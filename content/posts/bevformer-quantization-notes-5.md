+++
date = '2026-08-27T19:24:00+08:00'
title = 'BEVFormer Quantization Notes 5'
tags = ['Quantization']
series = ['BEVFormer量化']
toc = true
+++

# BEVFormer量化笔记5

> 本篇接续笔记 4：ONNX 导出修复尝试（第 9 节）。这些尝试最终没有解决问题，有效路线见笔记 8。

## 9. 修复ONNX导出

下面的修复都没有用！

### 9.1. `msda_pytorch_onnx`的for循环展开

这里面每一个动态Slice，TRT FP16/INT8融合进不了，全吐出来。4个level就4x个Slice，加上reshape就是156个Slice的来源。

修改，去掉动态Slice

```py
@staticmethod
def msda_pytorch_onnx(value, value_spatial_shapes, sampling_locations,
                      attention_weights):
    bs, _, num_heads, embed_dims = value.shape
    _, num_queries, _, num_levels, num_points, _ = sampling_locations.shape

    # 取最大 H, W 做 padding
    H_max = int(value_spatial_shapes[:, 0].max().item())
    W_max = int(value_spatial_shapes[:, 1].max().item())

    sampling_grids = 2 * sampling_locations - 1

    # 逐 level 的 grid 是 (bs*nh, numQ*numP, 2)，它们天然按 level 维度分开
    # 把 grid 和 value 按 level 维度一次性处理，避免循环
    # 把 value 按 level pad 到统一尺寸 (H_max, W_max)，然后一次性做 bilinear sample

    value_padded_list = []
    offset = 0
    for level in range(num_levels):
        H = int(value_spatial_shapes[level, 0].item())
        W = int(value_spatial_shapes[level, 1].item())
        v = value[:, offset:offset + H * W]
        v = v.flatten(2).transpose(1, 2).reshape(
            bs * num_heads, embed_dims, H, W)
        # pad to (H_max, W_max)
        v = torch.nn.functional.pad(v, (0, W_max - W, 0, H_max - H))
        value_padded_list.append(v)
        offset += H * W

    # (num_levels * bs*nh, C, H_max, W_max)
    value_all = torch.cat(value_padded_list, dim=0)

    # grid: (bs, numQ, nh, num_levels, numP, 2) -> (bs*nh*num_levels, numQ*numP, 2)
    grid_all = sampling_grids.permute(0, 2, 3, 1, 4, 5).reshape(
        bs * num_heads * num_levels, num_queries * num_points, 2)

    sampled = ONNXPatches.manual_bilinear_sample(
        value_all, grid_all, align_corners=False)

    # (bs*nh*num_levels, C, numQ*numP) -> (bs*nh, num_levels, C, numQ, numP)
    sampled = sampled.view(bs * num_heads, num_levels, embed_dims,
                           num_queries, num_points)

    attention_weights = attention_weights.transpose(1, 2).reshape(
        bs * num_heads, 1, num_queries, num_levels * num_points)
    output = (sampled.permute(0, 3, 1, 4, 2).reshape(
        bs * num_heads, num_queries, num_levels * num_points, embed_dims) *
              attention_weights.transpose(-1, -2)).sum(-2).view(
                  bs, num_heads * embed_dims, num_queries)
    return output.transpose(1, 2).contiguous()
```

效果： 4 次 manual_bilinear_sample → 1 次，4 个 Slice → 1 个 cat + pad，ONNX 图瘦身 4 倍。

### 9.2. 算子选择不当

`manual_bilinear_sample`里面用了`gather`+`expand`代替`grid_sample`。`gather`是很弱的算子，几乎不参与融合，每个`gather`都被独立发射。

降低算子复杂度

```python
    @staticmethod
    def manual_bilinear_sample(value, grid, align_corners=False):
        """Bilinear sampling without F.grid_sample (ONNX-friendly)."""
        N, C, H, W = value.shape
        num_points = grid.shape[1]

        # [-1, 1] -> pixel coords [0, W-1], [0, H-1]
        if align_corners:
            x = (grid[..., 0] + 1) * 0.5 * (W - 1)
            y = (grid[..., 1] + 1) * 0.5 * (H - 1)
        else:
            x = (grid[..., 0] + 1) * 0.5 * W - 0.5
            y = (grid[..., 1] + 1) * 0.5 * H - 0.5

        x = x.clamp(min=0, max=W - 1.001)
        y = y.clamp(min=0, max=H - 1.001)

        # ============================================================
        # FIX: Use int32 instead of long (int64) for TensorRT compat
        # ============================================================
        x0 = x.floor().to(torch.int32)
        y0 = y.floor().to(torch.int32)
        x1 = x0 + 1
        y1 = y0 + 1

        wx1 = x - x0.float()
        wy1 = y - y0.float()
        wx0 = 1.0 - wx1
        wy0 = 1.0 - wy1

        value_flat = value.reshape(N, C, H * W)  # (N, C, H*W)
        idx00 = y0 * W + x0
        idx01 = y0 * W + x1
        idx10 = y1 * W + x0
        idx11 = y1 * W + x1

        # gather value_flat[n, c, idx[n, p]] -> (N, C, num_points)
        v00 = value_flat.gather(2, idx00.unsqueeze(1).expand(-1, C, -1))
        v01 = value_flat.gather(2, idx01.unsqueeze(1).expand(-1, C, -1))
        v10 = value_flat.gather(2, idx10.unsqueeze(1).expand(-1, C, -1))
        v11 = value_flat.gather(2, idx11.unsqueeze(1).expand(-1, C, -1))

        w00 = (wx0 * wy0).unsqueeze(1)
        w01 = (wx1 * wy0).unsqueeze(1)
        w10 = (wx0 * wy1).unsqueeze(1)
        w11 = (wx1 * wy1).unsqueeze(1)

        return v00 * w00 + v01 * w01 + v10 * w10 + v11 * w11
```

改动点：

| 原来                               | 现在                                              | 效果                         |
| ---------------------------------- | ------------------------------------------------- | ---------------------------- |
| 4 次 gather                        | 1 次 gather                                       | gather 减 4 倍               |
| `x1 = x0 + 1`（又做一次 int32 加法） | `idx01 = idx00 + 1`（复用已算好的 `idx00`）       | 省 4 个 int32 add            |
| 4 组 `v00*w00 + v01*w01 + ...`     | stack 权重 → 一次 `*` + `sum`                     | TRT 融合成 1 个 PointWiseV2  |
| `.to(torch.int32)` 然后 `.float()` | `.to(torch.float32)` 显式                         | ONNX 图更干净                |

（上面贴的代码是只做了 int32 修正的中间版本；表中的合并方案后来随整条手动模拟路线一起被 plugin 方案取代，见笔记 8。）


### 9.3. `point_sampling`的中间Tensor链

改`point_sampling`：合并mask计算。改动点：

- `bev_mask` 由 bool 改为 float：ONNX/TensorRT 对 bool 张量支持差，float mask 可以直接参与乘法加权；
- 5 次比较（depth/x/y 范围）合并成一次 `bev_mask` 计算，减少中间 Tensor 链；
- `lidar2img` 与 `reference_points` 显式转 `float32`，避免隐式类型提升。

```python
@staticmethod
def point_sampling(self, reference_points, pc_range, img_metas):
    """ONNX-friendly point_sampling: float bev_mask instead of bool."""
    lidar2img = []
    for img_meta in img_metas:
        lidar2img.append(img_meta['lidar2img'])
    lidar2img = np.asarray(lidar2img)
    lidar2img = reference_points.new_tensor(lidar2img)

    reference_points = reference_points.clone()
    reference_points[..., 0:1] = reference_points[..., 0:1] * \
        (pc_range[3] - pc_range[0]) + pc_range[0]
    reference_points[..., 1:2] = reference_points[..., 1:2] * \
        (pc_range[4] - pc_range[1]) + pc_range[1]
    reference_points[..., 2:3] = reference_points[..., 2:3] * \
        (pc_range[5] - pc_range[2]) + pc_range[2]

    reference_points = torch.cat(
        (reference_points, torch.ones_like(reference_points[..., :1])), -1)
    reference_points = reference_points.permute(1, 0, 2, 3)
    D, B, num_query = reference_points.size()[:3]
    num_cam = lidar2img.size(1)

    reference_points = reference_points.view(D, B, 1, num_query,
                                             4).repeat(1, 1, num_cam, 1,
                                                       1).unsqueeze(-1)
    lidar2img = lidar2img.view(1, B, num_cam, 1, 4,
                               4).repeat(D, 1, 1, num_query, 1, 1)

    reference_points_cam = torch.matmul(lidar2img.to(torch.float32),
                                        reference_points.to(
                                            torch.float32))[..., 0]
    eps = 1e-5

    depth = reference_points_cam[..., 2:3].clamp(min=eps)
    reference_points_cam = reference_points_cam[..., 0:2] / depth

    img_w = float(img_metas[0]['img_shape'][0][1])
    img_h = float(img_metas[0]['img_shape'][0][0])
    x = reference_points_cam[..., 0:1] / img_w
    y = reference_points_cam[..., 1:2] / img_h
    reference_points_cam = torch.cat([x, y], dim=-1)

    bev_mask = ((depth > eps) & (x > 0.0) & (x < 1.0) & (y > 0.0) &
                (y < 1.0)).float()

    reference_points_cam = reference_points_cam.permute(2, 1, 3, 0, 4)
    bev_mask = bev_mask.permute(2, 1, 3, 0, 4)[..., 0]

    return reference_points_cam, bev_mask
```

## 10. 结论

9.1–9.3 的改写对 ONNX 图本身是有效的清理（Slice 合并、int32 索引、mask 合并），但没有解决两个根本问题：

1. **性能**：层数膨胀的根源是动态 shape 和 Q/DQ 链，手动改写收益有限；
2. **精度**：`cls_scores` 偏差的根源是 SCA 静态化后 mask 语义与原版不一致，不在这些改写覆盖的范围内。

最终有效的路线见笔记 8：用自定义 TensorRT plugin（`MultiScaleDeformableAttnTRT` / `QKVTRT`）整体替换 MSDA / TSA / decoder MHA，并修复静态 SCA 对被 mask anchor 的处理语义。
