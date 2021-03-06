# CapsNet with capsule-wise convolution

## Project structure 

```naive.py``` - naive implementation of a capslayer written on numpy

```capsnet``` - package with classes

```weights``` - weights of some already trained models

```demo.py``` - CapsNet as an encoder and three layer FC net as encoder

```generating-with-decoder.ipynb``` - notebook which shows what the individual dimensions of a capsule represent

```affNIST_test_capsnet.ipynb``` and ```affNIST_test_cnn.ipynb``` - experiment where we compare accuracy of CapsNet and custom CNN, whick were trained on MNIST, on affNIST. See below


## Train on padded and translated MNIST and then test on affNIST

You should download affNIST from [here](http://www.cs.toronto.edu/~tijmen/affNIST/32x/transformed/test.mat.zip) and extract it into ./affnist/test.mat.

### Generate train data based on standard MNIST dataset

Create dataset in which each example is an MNIST digit placed randomly on a black background of 40×40 pixels

Below is content of generate_datasets.py
```python
import numpy as np
from keras.datasets import mnist

(t_x_train, t_y_train), _ = mnist.load_data()

t_x_train = np.repeat(t_x_train, 8, axis=0)
x_train = np.zeros((t_x_train.shape[0], 40, 40))

for i in range(0, x_train.shape[0]):
    x, y = np.random.randint(0, 12, 2)
    x_train[i, y:y+28, x:x+28] = t_x_train[i]

y_train = np.repeat(t_y_train, 8, axis=0)

np.save('generateddatasets/x_train_only_translation.npy',
        x_train.astype(np.uint8))
np.save('generateddatasets/y_train_only_translation.npy',
        y_train.astype(np.uint8))
```

### Train CapsNet with 1 conv layer, 4 convcaps layers and 1 dense caps layer with routing only on the last layer

```python
l2 = regularizers.l2(l=0.001)

inp = Input(shape=input_shape)
l = inp

l = Conv2D(16, (5, 5), strides=(2, 2), activation='relu', kernel_regularizer=l2)(l)  # common conv layer
l = BatchNormalization()(l)
l = ConvertToCaps()(l)
l = Conv2DCaps(6, 4, (3, 3), (2, 2), r_num=1, b_alphas=[1, 1, 1], kernel_regularizer=l2)(l)
l = Conv2DCaps(5, 5, (3, 3), (1, 1), r_num=1, b_alphas=[1, 1, 1], kernel_regularizer=l2)(l)
l = Conv2DCaps(4, 6, (3, 3), (1, 1), r_num=1, b_alphas=[1, 1, 1], kernel_regularizer=l2)(l)
l = Conv2DCaps(3, 7, (3, 3), (1, 1), r_num=1, b_alphas=[1, 1, 1], kernel_regularizer=l2)(l)

l = FlattenCaps()(l)  # transform to a dense caps layer
l = DenseCaps(10, 8, r_num=3, b_alphas=[1, 8, 8], kernel_regularizer=l2)(l)
l = CapsToScalars()(l)

model = Model(inputs=inp, outputs=l, name='40x40_input_capsnet')
model.summary()
```

```
_________________________________________________________________
Layer (type)                 Output Shape              Param #   
=================================================================
input_5 (InputLayer)         (None, 40, 40, 1)         0         
_________________________________________________________________
conv2d_11 (Conv2D)           (None, 18, 18, 16)        416       
_________________________________________________________________
batch_normalization_14 (Batc (None, 18, 18, 16)        64        
_________________________________________________________________
convert_to_caps_2 (ConvertTo (None, 18, 18, 16, 1)     0         
_________________________________________________________________
conv2d_caps_5 (Conv2DCaps)   (None, 8, 8, 6, 4)        3456      
_________________________________________________________________
conv2d_caps_6 (Conv2DCaps)   (None, 6, 6, 5, 5)        5400      
_________________________________________________________________
conv2d_caps_7 (Conv2DCaps)   (None, 4, 4, 4, 6)        5400      
_________________________________________________________________
conv2d_caps_8 (Conv2DCaps)   (None, 2, 2, 3, 7)        4536      
_________________________________________________________________
flatten_caps_2 (FlattenCaps) (None, 12, 7)             0         
_________________________________________________________________
dense_caps_2 (DenseCaps)     (None, 10, 8)             6720      
_________________________________________________________________
caps_to_scalars_2 (CapsToSca (None, 10)                0         
=================================================================
Total params: 25,992
Trainable params: 25,960
Non-trainable params: 32
```

See affNIST_test_capsnet.ipynb fore more information

This model achieved 0.9772 accuracy on train set and 0.9796 on validation set

### Results on affNIST
```
Test score:  3.78991742519
Test accuracy:  0.704396875
```

### Train CNN with 3 conv layers and two dense layers

```python
l2 = regularizers.l2(l=0.001)

inp = Input(shape=input_shape)
l = inp

l = Conv2D(8, (3, 3), activation='relu', kernel_regularizer=l2)(l)
l = BatchNormalization()(l)
l = MaxPooling2D((2, 2))(l)
l = Conv2D(16, (3, 3), activation='relu', kernel_regularizer=l2)(l) 
l = BatchNormalization()(l)
l = MaxPooling2D((2, 2))(l)
l = Conv2D(32, (3, 3), activation='relu', kernel_regularizer=l2)(l)
l = BatchNormalization()(l)
l = MaxPooling2D((2, 2))(l)
l = Flatten()(l)
l = Dense(72, activation='relu', kernel_regularizer=l2)(l)
l = BatchNormalization()(l)
l = Dense(10, activation='softmax', kernel_regularizer=l2)(l)

model = Model(inputs=inp, outputs=l, name='40x40_input_cnn')
model.summary()
```

```
_________________________________________________________________
Layer (type)                 Output Shape              Param #   
=================================================================
input_1 (InputLayer)         (None, 40, 40, 1)         0         
_________________________________________________________________
conv2d_1 (Conv2D)            (None, 38, 38, 8)         80        
_________________________________________________________________
batch_normalization_1 (Batch (None, 38, 38, 8)         32        
_________________________________________________________________
max_pooling2d_1 (MaxPooling2 (None, 19, 19, 8)         0         
_________________________________________________________________
conv2d_2 (Conv2D)            (None, 17, 17, 16)        1168      
_________________________________________________________________
batch_normalization_2 (Batch (None, 17, 17, 16)        64        
_________________________________________________________________
max_pooling2d_2 (MaxPooling2 (None, 8, 8, 16)          0         
_________________________________________________________________
conv2d_3 (Conv2D)            (None, 6, 6, 32)          4640      
_________________________________________________________________
batch_normalization_3 (Batch (None, 6, 6, 32)          128       
_________________________________________________________________
max_pooling2d_3 (MaxPooling2 (None, 3, 3, 32)          0         
_________________________________________________________________
flatten_1 (Flatten)          (None, 288)               0         
_________________________________________________________________
dense_1 (Dense)              (None, 72)                20808     
_________________________________________________________________
batch_normalization_4 (Batch (None, 72)                288       
_________________________________________________________________
dense_2 (Dense)              (None, 10)                730       
=================================================================
Total params: 27,938
Trainable params: 27,682
Non-trainable params: 256
```

See affNIST_test_cpp.ipynb fore more information

This model achieved 0.9820 accuracy on train set and 0.9851 on validation set

### Results on affNIST
```
Test score:  0.965831407426
Test accuracy:  0.73925
```

### Conclusion
We trained both models for 4 epochs on translated digits from MNIST. The custom CNN achieved better result by the last epoch.
Accuracy of the CNN model on affNIST set also was better than accuracy of CapsNet model: 0.74 vs. 0.70.