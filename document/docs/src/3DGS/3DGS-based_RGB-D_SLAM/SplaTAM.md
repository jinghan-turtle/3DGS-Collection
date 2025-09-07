# SplaTAM：Splat, Track & Map

毕竟是首篇开源的 3DGS-SLAM 论文，SplaTAM[<sup>[1]</sup>](#SplaTAM-paper) 依然在 Nerf 隐式和 3DGS 显式快速建图的语境下谈高保真的场景重建 —— 隐式神经场表示面临计算效率低、不易编辑、不能明确模拟空间几何特征和灾难性遗忘的问题，那么与之相对的，SplaTAM 自然要回答如何在 SLAM 中嵌入 3DGS 技术：

> In this context, we explore the question, “How can one use an explicit volumetric representation to design a SLAM solution?” Specifically, we use a radiance field based on 3D Gaussians to Splat (Render), Track, and Map for SLAM. We believe that this representation has the following benefits over existing map representations:

1. Fast rendering and rich optimization：可想而知，高斯泼溅的一大优势便是可以以高达 400 FPS 的速度渲染场景，而为了进一步提高 3DGS 的渲染速度，SplaTAM 去除了高斯椭球外观对观测视角的依赖及其各向异性，即将球谐函数系数矩阵简化为 RGB 值 $\small c\in\mathbb{R}^3$，将协方差矩阵简化为球半径 $\small r$，从而得到高斯椭球的简化表示 $\small f(x)=o\exp(-||x-\mu||^2/2r^2)$，其中 $\small\mu\in\mathbb{R}^3$ 为高斯球均值，$\small o\in[0,1]$ 表示不透明度；同时，SplaTAM 仅在当前视图的可见范围内优化高斯椭球，类比像素 $\small p=(u,v)$ 的颜色渲染公式 $\small C(p)=\sum_{i=1}^nc_if_i(p)\prod_{j=1}^{i-1}( 1-f_j(p) )$，SplaTAM 引入深度损失 $\small D(p)=\sum_{i=1}^nd_if_i(p)\prod_{j=1}^{i-1}( 1-f_j(p) )$ 和像素可见性 $\small S(p)=\sum_{i=1}^nf_i(p)\prod_{j=1}^{i-1}( 1-f_j(p) )$，即该像素是否包含当前视图的信息以获得待优化视图的轮廓。

2. Maps with explicit spatial extent：SplaTAM 认为显式地图具有明确的空间范围，并可通过简单地添加高斯椭球来增加地图容量，允许在编辑部分场景时保持照片级的真实感渲染。正如由上面的像素可见性 $\small S(p)$ 定义的轮廓图，视图边界对于 SLAM 的相机跟踪显然是重要的，因为我们只希望将视图范围内的高斯椭球与新输入的 Ground Truth 图像作比较，而非像隐式建图那样每次都要优化全局的网络。

<!-- 3. Direct gradient flow to parameters：到参数的直接梯度流: 由于场景由具有物理3D位置、颜色和大小的高斯表示, 因此参数和渲染之间存在直接的、几乎线型的梯度流. 而隐式神经表示的优化则需要流经多层非线性网络层. -->

![](./SplaTAM.png){ width=100% style="display: block; margin: 0 auto;" }

## 相机跟踪：可见轮廓内的颜色深度误差

使用常速度模型 $\small E_{t+1} = E_t + (E_t-E_{t-1})$ 初始化相机姿态

//// collapse-code
```Python hl_lines="8"
''' scripts/spatam.py '''
def rgbd_slam(config: dict):
    ...
        # Initialize the camera pose for the current frame
        if time_idx > 0:
            params = initialize_camera_pose(params, time_idx, forward_prop=config['tracking']['forward_prop'])

def initialize_camera_pose(params, curr_time_idx, forward_prop):
    with torch.no_grad():
        if curr_time_idx > 1 and forward_prop:
            # Initialize the camera pose for the current frame based on a constant velocity model
            # Rotation
            prev_rot1 = F.normalize(params['cam_unnorm_rots'][..., curr_time_idx-1].detach())
            prev_rot2 = F.normalize(params['cam_unnorm_rots'][..., curr_time_idx-2].detach())
            new_rot = F.normalize(prev_rot1 + (prev_rot1 - prev_rot2))
            params['cam_unnorm_rots'][..., curr_time_idx] = new_rot.detach()

            # Translation
            prev_tran1 = params['cam_trans'][..., curr_time_idx-1].detach()
            prev_tran2 = params['cam_trans'][..., curr_time_idx-2].detach()
            new_tran = prev_tran1 + (prev_tran1 - prev_tran2)
            params['cam_trans'][..., curr_time_idx] = new_tran.detach()

        else:
            # Initialize the camera pose for the current frame
            params['cam_unnorm_rots'][..., curr_time_idx] = params['cam_unnorm_rots'][..., curr_time_idx-1].detach()
            params['cam_trans'][..., curr_time_idx] = params['cam_trans'][..., curr_time_idx-1].detach()
    
    return params
```
////

## 代码运行：Ubuntu20.04 & Cuda 11.6

```
conda create -n splatam python=3.10
conda activate splatam
conda install -c "nvidia/label/cuda-11.6.0" cuda-toolkit
conda install pytorch==1.12.1 torchvision==0.13.1 torchaudio==0.12.1 cudatoolkit=11.6 -c pytorch -c conda-forge
pip install -r requirements.txt
```

```
python scripts/splatam.py configs/tum/splatam.py
```

&nbsp;

<div id="SplaTAM-paper"></div>
[1] [Keetha N, Karhade J, Jatavallabhula K M, et al. Splatam: Splat track & map 3d gaussians for dense rgb-d slam[C]//Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition. 2024: 21357-21366.](https://spla-tam.github.io/)