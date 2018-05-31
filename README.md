SqueezeNet
===========
Introduction
-----------
Much of the recent research on deep convolutional neural networks (CNNs) has focused on increasing accuracy on computer vision datasets. For a given accuracy level, there typically exist multiple CNN architectures that achieve that accuracy level. Given equivalent accuracy, a CNN architecture with fewer parameters has several advantages:： 
* More efficient distributed training.  
* Less overhead when exporting new models to clients. 
* Feasible FPGA and embedded deployment.

With this in mind, this paper discovers such an architecture, called SqueezeNet, which reaches equivalent accuracy compared to AlexNet with 1/50 fewer parameters.  

Architecture
-----------
### Architecture Design Strategies
1. Replace `3x3` filters with `1x1` filter：1/9 fewer parameters than before 
2. Decrease the number of input channels to `3x3` filters：using squeeze layers 
3. Downsample late in the network so that convolution layers have large activation maps: large activation maps (due to delayed downsampling) can lead to higher classification accuracy
  
Strategies 1 and 2 are about judiciously decreasing the quantity of parameters in a CNN while attempting to preserve accuracy. Strategy 3 is about maximizing accuracy on a limited budget of parameters.
### The Fire Module
![](https://github.com/Panxj/SqueezeNet/raw/master/images/fire_module.jpg)
  * squeeze layer: decrease `3x3` filters and channels
  * expand layer: expand channels through a mix of `1x1` and `3x3` filters with three hyperparameters: <a href="https://www.codecogs.com/eqnedit.php?latex=s_{1X1}" target="_blank"><img src="https://latex.codecogs.com/gif.latex?s_{1X1}" title="s_{1X1}" /></a>(number of filters in the squeeze layer), <a href="https://www.codecogs.com/eqnedit.php?latex=e_{1X1}" target="_blank"><img src="https://latex.codecogs.com/gif.latex?e_{1X1}" title="e_{1X1}" /></a>(number of  `1x1` filters in the expand layer), <a href="https://www.codecogs.com/eqnedit.php?latex=e_{3X3}" target="_blank"><img src="https://latex.codecogs.com/gif.latex?e_{3X3}" title="e_{3X3}" /></a>(number of `3x3` filters in the expand layer)
  * when using Fire modules, set <a href="https://www.codecogs.com/eqnedit.php?latex=s_{1X1}&space;<&space;e_{1X1}&space;&plus;&space;e_{3X3}" target="_blank"><img src="https://latex.codecogs.com/gif.latex?s_{1X1}&space;<&space;e_{1X1}&space;&plus;&space;e_{3X3}" title="s_{1X1} < e_{1X1} + e_{3X3}" /></a> to limit the number of input channels to the `3x3` filters.

### The SqueezeNet Architecture
SqueezeNet begins with a standalone convolution layer (conv1), followed by 8 Fire modules (fire2-9), ending with a final conv layer (conv10). Gradually increase the number of filters per fire module from the beginning to the end of the network. SqueezeNet performs max-pooling with a stride of 2 after layers conv1, fire4, fire8, and conv10; these relatively late placements of pooling are per Strategy3. As shown followed, the left one is the Macroarchitectural view of SqueezeNet architecture. The middle and right are SqueezeNet with simple bypass and complex bypass correspondingly.

![](https://github.com/Panxj/SqueezeNet/raw/master/images/architecture.jpg)

Overview
-----------
Tabel 1. Directory structure

|file | description|
|:--- |:---|
train.py | Train script
infer.py | Prediction using the trained model
reader.py| Data reader
squeezenet.py| Model definition
data/val.txt|Validation data list
data/train.txt| Train data list
models/SqueezeNet_v1.1| Parameters converted from caffe model
images/\* | Images used in description and some results
output/\* | Parameters saved in training process

Data Preparation
-----------
First, download the ImageNet dataset. We using ILSVRC 2012(ImageNet Large Scale Visual Recognition Challenge) dataset in which,

* trainning set: 1,281,167 imags + labels
* validation set: 50,000 images + labels
* test set: 100,000 images

When trainning, all images are resized to `256` according to the short side and then croped out ones, size of `227 x 227`,  from upper left, upper right, center, lower left, lower right randomly. When testing, press the short edge again to `256`, then crop out `227x227` images from center. Finally, subtracting the mean value, here we use  `[104,117,123]` , which is slightly different from the official offer. The relevant function in `reader.py` is as following:
```python
def random_crop(orig_im,new_shape):
        id = random.randint(0,5)
        # left-top
        if id == 0:
            image = orig_im[:new_shape[0], :new_shape[1]]
        # right-top
        elif id == 1:
            image = orig_im[:new_shape[0], orig_im.shape[1]-new_shape[1]:]
        # left-bottom
        elif id == 2:
            image = orig_im[orig_im.shape[0]-new_shape[0]:,:new_shape[1]]
        # right-bottom
        elif id == 3:
            image = orig_im[orig_im.shape[0]-new_shape[0]:, orig_im.shape[1]-new_shape[1]:]
        # center
        else:
            off_w = (orig_im.shape[1] - new_shape[1]) / 2
            off_h = (orig_im.shape[0] - new_shape[0]) / 2
            image = orig_im[off_h:off_h + new_shape[0], off_w:off_w + new_shape[1]]
        return image
def center_crop(orig_im,new_shape=(227,227)):
    off_w = (orig_im.shape[1] - new_shape[1]) / 2
    off_h = (orig_im.shape[0] - new_shape[0]) / 2
    image = orig_im[off_h:off_h + new_shape[0], off_w:off_w + new_shape[1]]
    return image

def load_img(img_path, resize = 256, crop=227, flip=False):
    im = cv2.imread(img_path)
    if flip:
        im  = im[:,::-1,:]    #horizental-flip
    h, w = im.shape[:2]
    h_new, w_new = resize, resize
    if h > w:
        h_new = resize * h / w
    else:
        w_new = resize * w / h
    im = cv2.resize(im,(h_new, w_new), interpolation=cv2.INTER_CUBIC)
    #b, g, r = cv2.split(im)
    #im = cv2.merge([b - mean_value[0], g - mean_value[1], r - mean_value[2]])
    im = random_crop(im,(crop,crop))
    im = np.array(im).astype(np.float32)
    im = im.transpose((2, 0, 1))  # HWC => CHW
    im = im - mean_value
    im = im.flatten()
    #im = im / 255.0
    return im
```

data in train.txt is as followed：
```
n01440764/n01440764_10026.JPEG 0
n01440764/n01440764_10027.JPEG 0
n01440764/n01440764_10029.JPEG 0
n01440764/n01440764_10040.JPEG 0
n01440764/n01440764_10042.JPEG 0
n01440764/n01440764_10043.JPEG 0
```
data in val.txt is as followed：
```
ILSVRC2012_val_00000001.JPEG 65
ILSVRC2012_val_00000002.JPEG 970
ILSVRC2012_val_00000003.JPEG 230
ILSVRC2012_val_00000004.JPEG 809
ILSVRC2012_val_00000005.JPEG 516
ILSVRC2012_val_00000006.JPEG 57
```
Training
-----------
#### 1. Determine the architecture
The author [github](https://github.com/DeepScale/SqueezeNet) provide two versions of SqueezeNet model。 SqueezeNet_v1.0 is the implement one of architecture described in paper while SqueezeNet_v1.1 changes somewhere to reduce the computation by 2.4x with the accuracy preserved. Changes in SqueezeNet_v1.1 is shown as follwed：

Tabel 2. changes in SqueezeNet_v1.1
 
 | | SqueezeNet_v1.0 | SqueezeNet_v1.1|
 |:---|:---|:---|
 |conv1| 96 filters of resolution 7x7|64 filters of resolution 3x3|
 |pooling layers| pool_{1,4,8} | pool_{1,3,5}|
 |computation| 1.72GFLOPS/image| 0.72 GFOLPS/image:2.4x less computation|
 |ImageNet accuracy| >=80.3% top-5| >=80.3% top-5|
 
Here, we implement the latter one, SqueezeNet_v1.1.
#### 2. caffe2paddle
To train simply and quickly, we first transfor the [caffe](http://caffe.berkeleyvision.org/)-style parameters into ones can be used in [PaddlePaddle](http://www.paddlepaddle.org/) as the pretrained model and then we literaly `finetune` the model. 
We perfome the parameter conversion according to the method described [here](https://github.com/PaddlePaddle/models/tree/develop/image_classification/caffe2paddle). Our converted parameters are placed under directory `models/SqueezeNet_v1.1`. 
In `train.py`, we set the parameters converted from caffe into the paddle model as the initial value as followed:
```python
#Load pre-trained params
if args.model is not None:
    for layer_name in parameters.keys():
        layer_param_path = os.path.join(args.model,layer_name)
        if os.path.exists(layer_param_path):
            h,w = parameters.get_shape(layer_name)
            parameters.set(layer_name,load_parameter(layer_param_path,h,w))
```

#### 3. train
`python train.py --model models/SqueezeNet_v1.1 --trainer 2` <br>
--model: path/to/parameters converted from caffe. <br>
--trainer: trainer count used in Paddle. <br>
Below is a description about this script:
1. Call paddle.init with 2 GPUs.
2. `reader()`
3. During the training process, it will print some log information.

Testing
-----------
Run `python test.py` to preform model evaluation process using the trained model.
```python
# test code
test(
    val_file_list='./data/val.txt',
    model_path='./output/checkpoints/squeezenet_final.tar.gz',
    GPU=True,
    trainer=2)
```
Here `infer_file_list` specifies image path list to be inferred, `model_path` specifies directory to the trained parameters.
```
evaluation result.
```

Infering
-----------
Run `python infer.py` to perform the image classification using the trained model.
```python
# infer code
infer(
    infer_file_list='./data/infer.txt',
    model_path='./output/checkpoints/squeezenet.tar.gz',
    GPU=True,
    trainer=2)
```
Here `infer_file_list` specifies image path list to be inferred, `model_path` specifies directory to the trained parameters.
```
infer example result.
```

References
-----------
[SqueezeNet: AlexNet-level accuracy with 50x fewer parameters and <0.5MB model size](https://arxiv.org/abs/1602.07360)


