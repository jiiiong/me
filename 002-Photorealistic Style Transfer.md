# Photorealistic Style Transfer

[TOC]

## 0 inbox

- 

## 1 Understanding the Code

- in `utils.py `：

  - `load_image`： why do we bother to add one more batch size dimension? 

    > some manipulation on the 2D image requires its corresponding tensor to be 4D. 

  - `im_convert`：

    - matplotlib's `imshow()` function is strict with the image dimension and shape. 

    - `image = tensor.to("cpu").clone().detach()`为什么要分配给cpu

    - `ndarray.transpose(1,2,0)`

      > That is if the index of the initial element is (x,y,z), the position of that element in the resulting array becomes (y,z,x) 

  - `get_features`：nn.module can be parameterized and applies to the corresponding tensor.

- in `transfer.jpynb`:

  - `torch.optim.lr_scheduler` provides several methods to adjust the learning rate based on the number of epochs.

- in `HRNet`: 

  - 1x1 convolution: sometimes called network in network

    - Usage: changes #channels

  - Residual block: using a shortcut to fast forward activation.

    *bottleneck* residual  block: a variant of the residual block that utilizes 1x1 convolutions to create a bottleneck.

  - inception layer: instead of you choosing filter sizes etc. , you just do them all and concatenate all the outputs and let the network to learn what it wants. 

    - problem: computational cost is expensive

      use bottleneck layer: shrink the size temporarily

    - where does this name come from: the paper cites a meme from <<Inception>> that says: `We Need To Go Deeper`

  - Upsample: bring back the resolution. So the high-resolution subnet can combine the information received from the low-res subnet.

    [what does upsample do](https://stackoverflow.com/a/51605122)

## 2 Experiment Preparation

- [the algorithms to be compared](C:\Users\z9911\Desktop\毕业论文\the algorithm to be compared)

  | authors             | paper                                                    | type           | code                                                    |
  | ------------------- | -------------------------------------------------------- | -------------- | ------------------------------------------------------- |
  | Gatys et.al. (2016) | Image style transfer using convolutional neural networks | artistic       | https://github.com/leongatys/PytorchNeuralStyleTransfer |
  | Luan et al. (2017)  | Deep Photo Style Transfer                                | photorealistic | https://github.com/luanfujun/deep-photo-styletransfer   |
  
- GPU:

  GTX1050Ti

- [Image Set](C:\Users\z9911\Desktop\experiment\image_set): all images in *Hi-Res for photorealistic style transfer*

- comparison aspects

  - visual effect: using the *Sobel operator* to extract contour information of the resulting images and compare it with that of the content image.
  - training time: time consumed by training images of different resolutions by different approaches
  - empirical study: style information, semantic information, visual effect
  
- test the effect of different content weights (always set style weight to 1)
