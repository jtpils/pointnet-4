# chainer-pointnet

Various point cloud based deep neural network implementation by 
[Chainer](https://github.com/chainer/chainer) [1].

It includes PointNet, PointNet++, Kd-Network and Kd context net (3DContextNet).

## Installation

Please install [Chainer](https://github.com/chainer/chainer) 
(and [cupy](https://github.com/cupy/cupy) if you want to use GPU) beforehand,
```bash
# Install chainer
pip install chainer

# Install cupy if you use GPU (please change based on your CUDA version)
# For example, if you use CUDA 8.0,
pip install cupy-cuda80
# similarly cupy-cuda90 or cupy-cuda91 is avaialble.
```

and then type following to install master branch.

```bash
git clone https://github.com/corochann/chainer-pointnet.git
pip install -e chainer-pointnet
```

Also, some extension library is used in some of the code,
```bash
# Chainer Chemistry
git clone https://github.com/pfnet-research/chainer-chemistry.git
pip install -e chainer-chemistry
# ChainerEX
git clone https://github.com/corochann/chainerex.git
pip install -e chainerex
```


## Model structure

### PointNet [2]

Implementations are in `models/pointnet`.

Both classification and segmentation network are implemented.

`models/pointnet_cls` can be used for classification task.
`trans` option represents to use `TransformNet` or not.
`trans=False` corresponds `PointNetVanilla` (basic),
and `trans=True` corresponds `PointNet` in the paper, respectively.

In my experiment `PointNetVanilla` performs already very well,
the gain in `PoinetNet` is few (maybe only 1-2% gain) while computation becomes
 much huge (around 3 times slower).
 
Original implementation (in tensorflow) can be found on github under MIT license.

 - [charlesq34/pointnet](https://github.com/charlesq34/pointnet)

#### tips

I found the batch normalizations in the last linear layers are quite important.
The accuracy dramatically changes (10% or more) with BatchNormalization at FC
layers.

### PointNet++ [3]

Implementations are in `models/pointnet2`.

Both classification and segmentation network are implemented.

Just note that thanks to [cupy](https://github.com/cupy/cupy), 
there is no C language implementation for farthest point sampling, grouping or 
feature propagation on GPU. 
You don't need to build any C binary to use this function.
Please refer `utils/sampling.py` and `utils/grouping.py` for sampling & grouping.

Original implementation (in tensorflow) can be found on github under MIT license.

 - [charlesq34/pointnet2](https://github.com/charlesq34/pointnet2)


### Kd-Network [4]

Implementations are in `models/kdnet`

Both classification and segmentation network are implemented.

[Original implementation](https://github.com/Regenerator/kdnets) 
constructs KDTree by their own implementation, 
which supports random splitting.

However in my implementation, `scipy.spatial.cKDTree` is used for 
easy and faster implementation, by following [fxia22/kdnet.pytorch](https://github.com/fxia22/kdnet.pytorch).
So implementation in this repo does not support random tree splitting.

Original implementation (in tensorflow) can be found on github under MIT license.

 - [Regenerator/kdnets](https://github.com/Regenerator/kdnets)

Also, pytorch implementations can be found on github

 - [fxia22/kdnet.pytorch](https://github.com/fxia22/kdnet.pytorch)

### Kd Context Network (3DContextNet) [7]

Originally called 3DContextNet, but I named `KDContextNet` in my python program.

Implementations are in `models/kdcontextnet`

Both classification and segmentation network are implemented.

I could not find the implementation so far, and this is "inferred" 
implementation. 
Especially for segmentation part, how to "upsample" is not written in detail.
So this implementation might be different from author's implementation.

## Experiments

Experiments in each dataset is located under `expriments` folder.
Each folder is independent, so you can refer independently.

### ModelNet40 [5]

This is point cloud classification task of 40 category.
Download script & code is from [charlesq34/pointnet](https://github.com/charlesq34/pointnet)

 - http://modelnet.cs.princeton.edu/

The dataset is automatically downloaded and preprocessed. 
Dataset shape is `train: (9840, 3, num_point=1024, 1), test (2468, 3, num_point=1024, 1)`.
`ch=3`, meaning it only contains `(x, y, z)` coordinate information.

You can simply execute train code to train `PointNet` or `PointNetVanilla`.

```angular2html
# use gpu with id 0, train PointNetVanilla
$ python train.py -g 0 --trans=false --method=point_cls --out=results/point_vanilla

# use gpu with id 0, train PointNet 
$ python train.py -g 0 --method=point_cls --out=results/point

# use gpu with id 0, train PointNet++ 
$ python train.py -g 0 --method=point2_cls_ssg --out=results/point2_ssg
$ python train.py -g 0 --method=point2_cls_msg --out=results/point2_msg

# use gpu with id 0, train KDNet 
$ python train.py -g 0 --method=kdnet_cls --dropout_ratio=0.3 --use_bn=1 --out=results/kdnet

# use gpu with id 0, train KDContextNet 
$ python train.py -g 0 --method=kdcontextnet_cls --dropout_ratio=0.3 --use_bn=1 --out=results/kdcontextnet
python train.py -g 0 --method=kdcontextnet_cls --dropout_ratio=0.3 --use_bn=1 --out=results/kdcontextnet_level369
```

```text
python train.py --use_bn=1 --dropout_ratio=0.3
# PointNetVanilla
epoch       main/loss   main/cls_loss  main/trans_loss1  main/trans_loss2  main/accuracy  validation/main/loss  validation/main/accuracy  lr          elapsed_time
250         0.111644    0.111644                                           0.958367       0.606223              0.872596                  1e-05       3560.81
# PointNet
250         0.119699    0.117399       0.00227531        2.40329e-05       0.95684        0.587751              0.871795                  1e-05       10358
# PointNet2 SSG
250         0.0226786                                                      0.989821       0.631495              0.898638                  1e-05       54240.2
# PointNet2 MSG
250         0.0217696                                                      0.991653       0.610621              0.892628                  1e-05       160451

# KDNet with bn & dropout_ratio=0.3
250         0.10106                                                        0.962235       1.01367               0.820913                  1e-05       20324
# KDContextNet with bn & dropout_ratio=0.3
250         0.126861                                                       0.952769       0.835642              0.825321                  1e-05       31900.6
# KDContextNet with bn & dropout_ratio=0.3 & normalize=1 & residual=1
250         0.0804102                                                      0.9716         0.824233              0.838542                  1e-05       28028.5
```

KDNet seems "overfit" to the train data, meaning that its representation power is strong but it fails to generalize to test data.
Maybe this is due to its property of non-rotational invariance.

### S3DIS [6]

Stanford Large-Scale 3D Indoor Spaces Dataset (S3DIS) for point cloud segmentation task.
Download the dataset from,

 - [S3DIS Dataset](http://buildingparser.stanford.edu/dataset.html)
 
Prerocessing code adopted from [charlesq34/pointnet](https://github.com/charlesq34/pointnet)
under `third_party` directory.

Dataset shape:
```text
train shape (20291, 9, 4096, 1) (20291, 4096)
test  shape (3294, 9, 4096, 1) (3294, 4096)
```
`ch=9`, it contains `0-2 ch: XYZ, 3-5ch: RGB, 6-8 normalized XYZ` coordinate
information respectively.


Steps:

1. Go to download link: download S3DIS dataset from
 [S3DIS Dataset](http://buildingparser.stanford.edu/dataset.html).
 You need to send application form.
 
2. Download `Stanford3dDataset_v1.2_Aligned_Version.zip` file (4GB),
 place it under `s3dis/data` directory.

2'. Fix mis label manually.

`Stanford3dDataset_v1.2_Aligned_Version/Area_5/hallway_6/Annotations/ceiling_1.txt`
has wrong charcter at line 180389. Please fix it manually.

3. Preprocessing

`collect_indoor3d_data.py` is for data re-organization and 
`gen_indoor3d_h5.py` is to generate HDF5 files. (cite from [charlesq34/pointnet](https://github.com/charlesq34/pointnet/tree/master/sem_seg#dataset))

```angular2html
$ cd third_party
$ python collect_indoor3d_data.py
$ python gen_indoor3d_h5.py
```

4. Training

```bash
# use gpu with id 0, train PointNetVanilla
$ python train.py -g 0 --method=point_seg --trans=false --out=results/pointnet_vanilla

# use gpu with id 0, train PointNet 
$ python train.py -g 0 --method=point_seg --out=results/pointnet

# use gpu with id 0, train PointNet++
$ python train.py -g 0 --method=point2_seg_ssg --out=results/pointnet2

# use gpu with id 0, train KDNet 
$ python train.py -g 0 --method=kdnet_seg --dropout_ratio=0 --use_bn=1
$ python train.py -g 0 --method=kdnet_seg --dropout_ratio=0.3 --use_bn=1 --out=results/kdnet

# use gpu with id 0, train KDContextNet 
$ python train.py -g 0 --method=kdcontextnet_seg --dropout_ratio=0.3 --use_bn=1 --out=results/kdcontextnet
```

```text
python train.py --use_bn=1 --dropout_ratio=0.3
# PointNetVanilla
epoch       main/loss   main/cls_loss  main/trans_loss1  main/trans_loss2  main/accuracy  validation/main/loss  validation/main/accuracy  lr          elapsed_time
250         0.0305516   0.0305516                                          0.988418       0.690604              0.891049                  1e-05       101405
```

### ScanNet

Point cloud semantic segmentation task of indoor scenes.

 - [ScanNet: Richly-annotated 3D Reconstructions of Indoor Scenes (CVPR 2017 Spotlight)](https://www.youtube.com/watch?v=Olx4OnoZWQQ)

## LICENSE
MIT License.

No warranty or support for this implementation.
Each model performance is not guaranteed, and may not achieve the score reported in each paper. Use it at your own risk.

Please see the [LICENSE](https://github.com/corochann/chainer-pointnet/blob/master/LICENSE) file for details.

I appreciate the authors who open sourced their code for the reference under permissive license.

## Reference

[1] Seiya Tokui, Kenta Oono, Shohei Hido, and Justin Clayton. 
Chainer: a next-generation open source framework for deep learning. 
In *Proceedings of Workshop on Machine Learning Systems (LearningSys) in Advances in Neural Information Processing System (NIPS) 28*, 2015.

 - [paper](http://learningsys.org/papers/LearningSys_2015_paper_33.pdf)
 - [official page](https://chainer.org/)
 - [code: chainer/chainer](https://github.com/chainer/chainer)

[2] Qi, Charles R and Su, Hao and Mo, Kaichun and Guibas, Leonidas J.
PointNet: Deep Learning on Point Sets for 3D Classification and Segmentation. 
*arXiv preprint arXiv:1612.00593*, 2016.

 - [paper on arXiv](https://arxiv.org/abs/1612.00593)
 - [project page](http://stanford.edu/~rqi/pointnet/)
 - [code: charlesq34/pointnet](https://github.com/charlesq34/pointnet)
 - CVPR oral

[3] Qi, Charles R and Yi, Li and Su, Hao and Guibas, Leonidas J.
PointNet++: Deep Hierarchical Feature Learning on Point Sets in a Metric Space. 
*arXiv preprint arXiv:1706.02413* 2017.

 - [paper on arXiv](https://arxiv.org/abs/1706.02413)
 - [project page](http://stanford.edu/~rqi/pointnet2/)
 - [code: charlesq34/pointnet2](https://github.com/charlesq34/pointnet2)
 - NIPS 2017

[4] Roman, Klokov and Victor, Lempitsky. 
Escape from Cells: Deep Kd-Networks for the Recognition of 3D Point Cloud Models. 
*arXiv preprint arXiv:1704.01222* 2017.

 - [paper on arXiv](https://arxiv.org/abs/1704.01222)
 - [project page](http://sites.skoltech.ru/compvision/kdnets)
 - [code: Regenerator/kdnets](https://github.com/Regenerator/kdnets)
 - ICCV 2017

[5] Z, Wu and S, Song and A, Khosla and F, Yu and L, Zhang and X, Tang and J, Xiao. 
3D ShapeNets: A Deep Representation for Volumetric Shapes. 
Proceedings of 28th IEEE Conference on Computer Vision and Pattern Recognition (CVPR2015), 2015.

 - [project page](http://modelnet.cs.princeton.edu/)

[6] Iro, Armeni and Alexander, Sax and Amir, R., Zamir and Silvio, Savarese. 
Joint 2D-3D-Semantic Data for Indoor Scene Understanding. 
*arXiv preprint arXiv:1702.01105* 2017.

 - [paper on arXiv](https://arxiv.org/abs/1702.01105)
 - [project page](http://buildingparser.stanford.edu/dataset.html)
 - [code: alexsax/2D-3D-Semantics](https://github.com/alexsax/2D-3D-Semantics)

[7] Wei, Zeng and Theo, Gevers. 
3DContextNet: K-d Tree Guided Hierarchical Learning of Point Clouds Using Local and Global Contextual Cues. 
*arXiv preprint arXiv:1711.11379* 2017.

 - [paper on arXiv](https://arxiv.org/abs/1711.11379)

