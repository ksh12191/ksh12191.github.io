---
layout: post
title: Convolution Autoencoder toy example
---

CAE를 이용해서 wavelet변환의 mother wavelet 역할을 하는 친구를 찾아보


```python
import matplotlib.pyplot as plt
import numpy as np
import math
import tensorflow as tf
import os
import tensorflow.examples.tutorials.mnist.input_data as input_data
from sklearn.utils import shuffle
%matplotlib inline 

```

# 1. 이미지 예제 


참고 : https://github.com/sjchoi86/tensorflow-101/blob/master/notebooks/cae_mnist.ipynb
이분의 이미지예제를 참고해서 신호에 적용시켜봄

MNIST 이미지 이용해서 CAE model 생성

특징 : training과정에서 test이미지를 모델에 넣어서 CAE가 얼마나 잘되고있는지 보여줌

## 1.1. 이미지 불러오기


```python
mnist = input_data.read_data_sets("data/", one_hot=True)
trainimgs   = mnist.train.images
trainlabels = mnist.train.labels
testimgs    = mnist.test.images
testlabels  = mnist.test.labels
ntrain      = trainimgs.shape[0]
ntest       = testimgs.shape[0]
dim         = trainimgs.shape[1]
nout        = trainlabels.shape[1]
print ("Packages loaded")
```

## 1.2. Define Network

- convolution layer와 bias에 들어갈 변수를 초기화 한다. 
- random_normal로 초기화


```python
n1 = 16
n2 = 32
n3 = 64
ksize = 5
weights = {
    'ce1': tf.Variable(tf.random_normal([ksize, ksize, 1, n1],  stddev=0.1)),
    'ce2': tf.Variable(tf.random_normal([ksize, ksize, n1, n2], stddev=0.1)),
    'ce3': tf.Variable(tf.random_normal([ksize, ksize, n2, n3], stddev=0.1)),
    'cd3': tf.Variable(tf.random_normal([ksize, ksize, n2, n3], stddev=0.1)),
    'cd2': tf.Variable(tf.random_normal([ksize, ksize, n1, n2], stddev=0.1)),
    'cd1': tf.Variable(tf.random_normal([ksize, ksize, 1, n1],  stddev=0.1))
}
biases = {
    'be1': tf.Variable(tf.random_normal([n1], stddev=0.1)),
    'be2': tf.Variable(tf.random_normal([n2], stddev=0.1)),
    'be3': tf.Variable(tf.random_normal([n3], stddev=0.1)),
    'bd3': tf.Variable(tf.random_normal([n2], stddev=0.1)),
    'bd2': tf.Variable(tf.random_normal([n1], stddev=0.1)),
    'bd1': tf.Variable(tf.random_normal([1],  stddev=0.1))
}
```


```python
def cae(_X, _W, _b, _keepprob):
    _input_r = tf.reshape(_X, shape=[-1, 28, 28, 1])
    # Encoder
    _ce1 = tf.nn.sigmoid(tf.add(tf.nn.conv2d(_input_r, _W['ce1']
        , strides=[1, 2, 2, 1], padding='SAME'), _b['be1']))
    _ce1 = tf.nn.dropout(_ce1, _keepprob)
    _ce2 = tf.nn.sigmoid(tf.add(tf.nn.conv2d(_ce1, _W['ce2']
        , strides=[1, 2, 2, 1], padding='SAME'), _b['be2'])) 
    _ce2 = tf.nn.dropout(_ce2, _keepprob)
    _ce3 = tf.nn.sigmoid(tf.add(tf.nn.conv2d(_ce2, _W['ce3']
        , strides=[1, 2, 2, 1], padding='SAME'), _b['be3'])) 
    _ce1 = tf.nn.dropout(_ce3, _keepprob)
    # Decoder
    _cd3 = tf.nn.sigmoid(tf.add(tf.nn.conv2d_transpose(_ce3, _W['cd3']
        , tf.pack([tf.shape(_X)[0], 7, 7, n2]), strides=[1, 2, 2, 1]
        , padding='SAME'), _b['bd3'])) 
    _cd3 = tf.nn.dropout(_cd3, _keepprob)
    _cd2 = tf.nn.sigmoid(tf.add(tf.nn.conv2d_transpose(_cd3, _W['cd2']
        , tf.pack([tf.shape(_X)[0], 14, 14, n1]), strides=[1, 2, 2, 1]
        , padding='SAME') , _b['bd2'])) 
    _cd2 = tf.nn.dropout(_cd2, _keepprob)
    _cd1 = tf.nn.sigmoid(tf.add(tf.nn.conv2d_transpose(_cd2, _W['cd1']
        , tf.pack([tf.shape(_X)[0], 28, 28, 1]), strides=[1, 2, 2, 1]
        , padding='SAME'), _b['bd1'])) 
    _cd1 = tf.nn.dropout(_cd1, _keepprob)
    _out = _cd1
    return {'input_r': _input_r, 'ce1': _ce1, 'ce2': _ce2, 'ce3': _ce3
        , 'cd3': _cd3, 'cd2': _cd2, 'cd1': _cd1
        , 'layers': (_input_r, _ce1, _ce2, _ce3, _cd3, _cd2, _cd1)
        , 'out': _out}
print ("Network ready")

```

## 1.3. Cost Function

- 학습을 위한 cost function 정의


```python
x = tf.placeholder(tf.float32, [None, dim])
y = tf.placeholder(tf.float32, [None, dim])
keepprob = tf.placeholder(tf.float32)
pred = cae(x, weights, biases, keepprob)['out']
cost = tf.reduce_sum(tf.square(cae(x, weights, biases, keepprob)['out'] 
            - tf.reshape(y, shape=[-1, 28, 28, 1])))
learning_rate = 0.001
optm = tf.train.AdamOptimizer(learning_rate).minimize(cost)
init = tf.initialize_all_variables()
print ("Functions ready")
```

## 1.4. training

- 이 코드에 포함된것
    - batch gradient descent
    - batch normalize
    - auto-encoder할때 input에 noise넣어서 noise에 좀 자유러워 지기

- iteration안에서 학습이 잘되고있나 알고싶어 중간중간 input > CAE > output 비교할 수 있는 코드작성 


```python
sess = tf.Session()
sess.run(init)
# mean_img = np.mean(mnist.train.images, axis=0)
mean_img = np.zeros((784))
# Fit all training data
batch_size = 128
n_epochs   = 5
print("Strart training..")
for epoch_i in range(n_epochs):
    for batch_i in range(mnist.train.num_examples // batch_size):
        batch_xs, _ = mnist.train.next_batch(batch_size)
        trainbatch = np.array([img - mean_img for img in batch_xs])
        trainbatch_noisy = trainbatch + 0.3*np.random.randn(
            trainbatch.shape[0], 784)
        sess.run(optm, feed_dict={x: trainbatch_noisy
                                  , y: trainbatch, keepprob: 0.7})
    print ("[%02d/%02d] cost: %.4f" % (epoch_i, n_epochs
        , sess.run(cost, feed_dict={x: trainbatch_noisy
                                    , y: trainbatch, keepprob: 1.})))
    if (epoch_i % 1) == 0:
        n_examples = 5
        test_xs, _ = mnist.test.next_batch(n_examples)
        test_xs_noisy = test_xs + 0.3*np.random.randn(
            test_xs.shape[0], 784)
        recon = sess.run(pred, feed_dict={x: test_xs_noisy, keepprob: 1.})
        fig, axs = plt.subplots(2, n_examples, figsize=(15, 4))
        for example_i in range(n_examples):
            axs[0][example_i].matshow(np.reshape(
                test_xs_noisy[example_i, :], (28, 28))
                , cmap=plt.get_cmap('gray'))
            axs[1][example_i].matshow(np.reshape(
                np.reshape(recon[example_i, ...], (784,))
                + mean_img, (28, 28)), cmap=plt.get_cmap('gray'))
        plt.show()
print("Training done. ")
```

# 2. 신호 예제

우리는 신호로 할꺼니까 신호예제를만들자.
(부끄럽지만 위의 git예제를 그대로 복붙해서만들었다.)

이미지도 신호이긴 하지만 우리는 input 을 ($1\times N$)크기의 신호로, convolution을 ($n \times 1$)의 wavelet 처럼 만들고 싶다. 
- CAE로 wavelet찾기?



## 2.1. 샘플데이터 생성

코사인 함수 + normal 노이즈


```python
X = np.linspace(0.05,2,100)
signal_train = np.cos(2*np.pi*X) + np.random.normal(loc=0.0, scale=0.1, size=(1000,100)) 
signal_test =  np.cos(2*np.pi*X) + np.random.normal(loc=0.0, scale=0.1, size=(200,100)) 
```

어떻게 생겨먹은 샘플인가?


```python
plt.plot(X,signal_train[2,:],'-',mew=1.5)
plt.show()
```

## 2.2. Define Network

파라미터 초기화

ex) 컨벌루션의 크기라던지, 몇개나만들어낼지 등등

---

추가)

컨벌루션 지나면 어떻게 줄어들지?
파라미터는 ?

$$\frac{N-F}{S} + 1$$

이 식에따라서 줄어듦. 



```python
n1 = 1
n2 = 1

ksize_x1= 1
ksize_y1 = 5
ksize_x2= 1
ksize_y2 = 25
weights = {
    'ce1': tf.Variable(tf.random_normal([ksize_x1, ksize_y1, 1, n1],  stddev=0.1), name = 'conv1'),
    'ce2': tf.Variable(tf.random_normal([ksize_x2, ksize_y2, n1, n2],  stddev=0.1), name = 'deconv2'),
    'cd2': tf.Variable(tf.random_normal([ksize_x2, ksize_y2, n1, n2],  stddev=0.1), name = 'deconv2'),
    'cd1': tf.Variable(tf.random_normal([ksize_x1, ksize_y1, 1, n1],  stddev=0.1), name = 'conv2')
}
biases = {
    'be1': tf.Variable(tf.random_normal([n1], stddev=0.1), name = 'bias1'),
    'be2': tf.Variable(tf.random_normal([n2], stddev=0.1), name = 'bias2'),    
    'bd2': tf.Variable(tf.random_normal([n1],  stddev=0.1), name = 'de_bias2'),
    'bd1': tf.Variable(tf.random_normal([1],  stddev=0.1), name = 'de_bias1')
}

```

- 우리는 _W, _b 즉, 파라미터가 필요하다. (이 친구들이 우리가 알고자 하는 새로운 feature를 만들어줄 친구들)
- 그래서 위의 예제와 다르게 return값을 추가적으로 더 받는다.


```python
def cae(_X, _W, _b, _keepprob):
    with tf.name_scope('input'):
        _input_r = tf.reshape(_X, shape=[-1, 1, 100, 1])
    # Encoder
    with tf.name_scope('convolution'):
        with tf.name_scope('weights'):
            _ce1 = tf.nn.tanh(tf.add(tf.nn.conv2d(_input_r, _W['ce1']
                , strides=[1, 1, 1, 1], padding='SAME'), _b['be1']))
        with tf.name_scope('dropout'):
            _ce1 = tf.nn.dropout(_ce1, _keepprob)
        # 두번째 친구는 겹치지 않게 stride를 넣어야할까?
        with tf.name_scope('weights'):
            _ce2 = tf.nn.tanh(tf.add(tf.nn.conv2d(_ce1, _W['ce2']
                , strides=[1, 1, 1, 1], padding='SAME'), _b['be2'])) 
        with tf.name_scope('dropout'):
            _ce2 = tf.nn.dropout(_ce2, _keepprob)

    # Decoder
    with tf.name_scope('deconvolution'):
        with tf.name_scope('weights'):
            _cd2 = tf.nn.tanh(tf.add(tf.nn.conv2d_transpose(_ce2, _W['cd2']
                , tf.pack([tf.shape(_X)[0], 1, 100, n1]), strides=[1, 1, 1, 1]
                , padding='SAME') , _b['bd2'])) 
        with tf.name_scope('dropout'):
            _cd2 = tf.nn.dropout(_cd2, _keepprob)
        with tf.name_scope('weights'):
            _cd1 = tf.nn.tanh(tf.add(tf.nn.conv2d_transpose(_cd2, _W['cd1']
                , tf.pack([tf.shape(_X)[0], 1, 100, 1]), strides=[1, 1, 1, 1]
                , padding='SAME'), _b['bd1'])) 
        with tf.name_scope('dropout'):
            _cd1 = tf.nn.dropout(_cd1, _keepprob)
        _out = _cd1

    return {'input_r': _input_r, 'ce1': _ce1, 'cd1': _cd1,  'ce2': _ce2, 'cd2': _cd2
        , 'layers': (_input_r, _ce1, _cd1, _ce2, _cd2)
        , 'out': _out}, _W, _b
print ("Network ready")

```

## 2.3. Cost Function

- 이미지 예제와 달라진점
    - `CAE_model, weight_tensor, bias_tensor`
    - 모델의 결과, 파라미터를 모두 받을 것이다.


```python
learning_rate = 0.001
dim = 100
x = tf.placeholder(tf.float32, [None, dim], name = 'input_x')
y = tf.placeholder(tf.float32, [None, dim], name = 'output_y')
keepprob = tf.placeholder(tf.float32, name = 'keepprob')

with tf.name_scope('model'):
    CAE_model, weight_tensor, bias_tensor =  cae(x, weights, biases, keepprob)

with tf.name_scope('train'):
    cost = tf.reduce_sum(tf.square(CAE_model['out'] 
                - tf.reshape(y, shape=[-1, 1, 100, 1])))
    ce_summ = tf.scalar_summary("sum of square", cost)
    optm = tf.train.AdamOptimizer(learning_rate).minimize(cost)
    
with tf.name_scope('accuracy'):
    with tf.name_scope('correct_prediction'):
            correct_prediction = tf.equal(tf.argmax(y, 1), tf.argmax(CAE_model['out'], 1))
    with tf.name_scope('accuracy'):
            accuracy = tf.reduce_mean(tf.cast(correct_prediction, tf.float32))
            tf.scalar_summary('accuracy', accuracy)
print ("Functions ready")
```

## 2.4. Training 
- batch normalize나 noise추가 따위 다 뺌
- parameter를 tensor에서 꺼내온다


```python

init = tf.initialize_all_variables()
# Create a summary to monitor cost tensor
tf.scalar_summary("loss", cost)
# Create a summary to monitor accuracy tensor
tf.scalar_summary("accuracy", acc)
# Merge all summaries into a single op
merged = tf.merge_all_summaries()


gpu_options = tf.GPUOptions(per_process_gpu_memory_fraction=0.4)
sess = tf.Session(config=tf.ConfigProto(gpu_options=gpu_options))
with tf.Session() as sess:
    sess.run(init)
    saver = tf.train.Saver()


    #to write logs to Tensorboard
    writer = tf.train.SummaryWriter("./board/CAE", graph=tf.get_default_graph())

    # Fit all training data
    n_epochs   = 10
    print("Strart training..")


    #epoch는 시그날의 수 * number of epoch로
    for epoch_i in range(n_epochs*signal_train.shape[0]):

        train_batch = shuffle(signal_train, n_samples = 1)
        sess.run(optm, feed_dict={x: train_batch
                                  , y: train_batch, keepprob: 0.7})

        if (epoch_i % signal_train.shape[0]) == 0:
            print ("[%02d/%02d] cost: %.4f" % (epoch_i, n_epochs*signal_train.shape[0]
            , sess.run( cost, feed_dict={x: train_batch
                                        , y: train_batch, keepprob: 1.})))
            summary, acc = sess.run([merged, accuracy], feed_dict={x: signal_test
                                                                   , y: signal_test, keepprob: 1.})
            writer.add_summary(summary, epoch_i)
            print('Accuracy at step %s: %s' % (epoch_i, acc))


    #모델에서 생기는 결과값, 파라미터 변수에 넣기
    model = sess.run(CAE_model, feed_dict={x: signal_test, keepprob: 1.})
    recon = model['out']
    reconv1 = model['ce1']
    reconv2 = model['ce2']
    weights = sess.run(weight_tensor, feed_dict={x: signal_test, keepprob: 1.})
    bias = sess.run(bias_tensor, feed_dict={x: signal_test, keepprob: 1.})

    print ("save result...")

    print("Training done... ")
```


```python
plt.plot(weights['ce2'].reshape(25))
```


```python
fig = plt.figure(figsize = (10, 6))
ax1 = fig.add_subplot(5, 1, 1, title = "input data+noise", ylim = (-1.5, 1.5))
ax1.plot(X, signal_test[0, :])

ax2 = fig.add_subplot(5, 1, 2, title = "input data", ylim = (-1.5, 1.5))
ax2.plot(X, np.cos(2*np.pi*X))

ax3 = fig.add_subplot(5, 1, 3, title = "CAE", ylim = (-1.5, 1.5))
ax3.plot(X, recon[0, :].reshape(100))

ax4 = fig.add_subplot(5, 1, 4, title = "conv 1", ylim =(-1.5, 1.5))
ax4.plot(X, reconv1[0, :, :, 0].reshape(100))

ax5 = fig.add_subplot(5, 1, 5, title = "conv 2", ylim =(-1.5, 1.5))
ax5.plot(X, reconv2[0, :, :, 0].reshape(100))
plt.tight_layout()
```


```python

```


```python

```


```python

```