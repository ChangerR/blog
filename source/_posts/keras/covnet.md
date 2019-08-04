---
title: 使用keras构建minist卷积神经网络  
tags:
  - keras
  - python
categories:
  - python
  - keras
date: 2019-08-04 16:59:19
---

```python
from keras import layers
from keras import models
```

    Using TensorFlow backend.
    


```python
from keras.datasets import mnist
import matplotlib.pyplot as plt
from keras.utils import to_categorical
import numpy as np
```


```python
model = models.Sequential()
model.add(layers.Conv2D(32, (3,3), activation='relu', input_shape=(28,28,1)))
model.add(layers.MaxPooling2D((2,2)))
model.add(layers.Conv2D(64, (3,3), activation='relu'))
model.add(layers.MaxPooling2D((2,2)))
model.add(layers.Conv2D(64, (3,3), activation='relu'))
model.add(layers.Flatten())
model.add(layers.Dense(64, activation='relu'))
model.add(layers.Dense(10, activation='softmax'))


```

    WARNING: Logging before flag parsing goes to stderr.
    W0804 15:59:32.254890 14660 deprecation_wrapper.py:119] From c:\users\changer\tools\python36\lib\site-packages\keras\backend\tensorflow_backend.py:74: The name tf.get_default_graph is deprecated. Please use tf.compat.v1.get_default_graph instead.
    
    W0804 15:59:32.268918 14660 deprecation_wrapper.py:119] From c:\users\changer\tools\python36\lib\site-packages\keras\backend\tensorflow_backend.py:517: The name tf.placeholder is deprecated. Please use tf.compat.v1.placeholder instead.
    
    W0804 15:59:32.270891 14660 deprecation_wrapper.py:119] From c:\users\changer\tools\python36\lib\site-packages\keras\backend\tensorflow_backend.py:4138: The name tf.random_uniform is deprecated. Please use tf.random.uniform instead.
    
    W0804 15:59:32.280911 14660 deprecation_wrapper.py:119] From c:\users\changer\tools\python36\lib\site-packages\keras\backend\tensorflow_backend.py:3976: The name tf.nn.max_pool is deprecated. Please use tf.nn.max_pool2d instead.
    
    


```python
model.summary()
```

    _________________________________________________________________
    Layer (type)                 Output Shape              Param #   
    =================================================================
    conv2d_1 (Conv2D)            (None, 26, 26, 32)        320       
    _________________________________________________________________
    max_pooling2d_1 (MaxPooling2 (None, 13, 13, 32)        0         
    _________________________________________________________________
    conv2d_2 (Conv2D)            (None, 11, 11, 64)        18496     
    _________________________________________________________________
    max_pooling2d_2 (MaxPooling2 (None, 5, 5, 64)          0         
    _________________________________________________________________
    conv2d_3 (Conv2D)            (None, 3, 3, 64)          36928     
    _________________________________________________________________
    flatten_1 (Flatten)          (None, 576)               0         
    _________________________________________________________________
    dense_1 (Dense)              (None, 64)                36928     
    _________________________________________________________________
    dense_2 (Dense)              (None, 10)                650       
    =================================================================
    Total params: 93,322
    Trainable params: 93,322
    Non-trainable params: 0
    _________________________________________________________________
    


```python
(train_images, train_labels), (test_images, test_labels) = mnist.load_data()
train_images = train_images.reshape((60000, 28 , 28, 1))
train_images = train_images.astype('float32') / 255

test_images = test_images.reshape((10000, 28 , 28, 1))
test_images = test_images.astype('float32') / 255

train_labels = to_categorical(train_labels)
test_labels = to_categorical(test_labels)
```


```python
model.compile(optimizer='rmsprop',
             loss = 'categorical_crossentropy',
             metrics=['accuracy'])

model.fit(train_images, train_labels, epochs=5, batch_size=64)
```

    W0804 15:59:32.730477 14660 deprecation_wrapper.py:119] From c:\users\changer\tools\python36\lib\site-packages\keras\optimizers.py:790: The name tf.train.Optimizer is deprecated. Please use tf.compat.v1.train.Optimizer instead.
    
    W0804 15:59:32.746480 14660 deprecation_wrapper.py:119] From c:\users\changer\tools\python36\lib\site-packages\keras\backend\tensorflow_backend.py:3295: The name tf.log is deprecated. Please use tf.math.log instead.
    
    W0804 15:59:32.805478 14660 deprecation.py:323] From c:\users\changer\tools\python36\lib\site-packages\tensorflow\python\ops\math_grad.py:1250: add_dispatch_support.<locals>.wrapper (from tensorflow.python.ops.array_ops) is deprecated and will be removed in a future version.
    Instructions for updating:
    Use tf.where in 2.0, which has the same broadcast rule as np.where
    W0804 15:59:32.868476 14660 deprecation_wrapper.py:119] From c:\users\changer\tools\python36\lib\site-packages\keras\backend\tensorflow_backend.py:986: The name tf.assign_add is deprecated. Please use tf.compat.v1.assign_add instead.
    
    

    Epoch 1/5
    60000/60000 [==============================] - 7s 111us/step - loss: 0.1763 - acc: 0.9450 1s - loss:
    Epoch 2/5
    60000/60000 [==============================] - 5s 79us/step - loss: 0.0483 - acc: 0.9854: 0s - loss: 0.0488 - acc:
    Epoch 3/5
    60000/60000 [==============================] - 5s 78us/step - loss: 0.0324 - acc: 0.9902
    Epoch 4/5
    60000/60000 [==============================] - 5s 77us/step - loss: 0.0256 - acc: 0.9923: 1s - loss
    Epoch 5/5
    60000/60000 [==============================] - 5s 77us/step - loss: 0.0199 - acc: 0.9939: 1s
    




    <keras.callbacks.History at 0x1de86781828>




```python
plt.imshow(test_images[1].reshape((28,28)), cmap=plt.cm.binary)
```




    <matplotlib.image.AxesImage at 0x1e01a9b8e48>


```python
model.predict(test_images[1:2])
```




    array([[3.45825661e-08, 1.02685015e-04, 9.99897242e-01, 1.25615013e-13,
            2.13491239e-10, 5.40613527e-15, 1.43496537e-09, 3.17772197e-11,
            3.05692971e-11, 5.21330949e-15]], dtype=float32)




```python
model.evaluate(test_images, test_labels)
```

    10000/10000 [==============================] - 1s 57us/step
    




    [0.0386850821582374, 0.9889]




```python

```
