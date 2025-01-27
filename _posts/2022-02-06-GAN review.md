---
layout: post
section-type: post
title: Generative Adversarial Nets (GAN, 생성적 적대 신경망) 리뷰 및 코드 구현
category: paper
tags: [ 'deep learning','machine learning','data science','paper' ]
---

안녕하세요, Pythonash 입니다.

이번에는 딥러닝 분야 중에서도 가장 흥미로운 분야 중 하나인 **생성적 적대 신경망** 논문을 읽고 짧게 리뷰한 뒤, 코드로 구현해 보려고 합니다.

사실 개념적인 컨셉이나 구동 알고리즘 등은 A to Z까지 설명된 자료들도 많아서...

실질적으로 코드를 구현해보면 더 도움이 되지 않을까 해서 작성해봤습니다.

# 목차

<a id="toc"></a>
- [1. GAN의 구조](#1)
- [2. Code 구현](#2)
    - [2.1 데이터셋](#2.1) 
    - [2.2 모델구현](#2.2)
    - [2.3 훈련함수 작성](#2.3)
    - [2.4 GAN 훈련](#2.4)
- [3. Review ](#3)
- [4. References](#4)

<a id="1"></a>
# GAN의 구조

심층 생성적 모델(Deep generative model)은 확률적인 계산이 어렵고, 하이퍼볼릭 탄젠트나 ReLU(piecewise linear unit)와 같은 것들이 "생성한다는 측면에서" 작동하기가 쉽지 않다는 점이 있습니다.

여기서 제가 piecewise linear unit을 활성화함수로 비유한 것은 본 논문에서는 piecewise linear unit이라고 되어 있는데, 값이나 구간에 따라 나누어진 함수로 구성된 것들로 문맥상 이러한 **활성화함수**를 말하는 것 같습니다.

아무튼, 이러한 어려운 점을 피하기 위해 논문에서는 새로운 생성적 모델을 제안한다는 겁니다.

제안된 적대적 신경망은 생성적 모델이 서로 적대적으로 피팅된다는 것인데, 여기에서는 잘 아시다시피 **생성자**와 **판별자**가 등장하게 됩니다.

여기에서 **판별자**가 흔히, *진짜인지 가짜인지를 판별하는 것이다*. 정도로 알고 계실텐데 더 정확히 말하자면 ***생성자로부터 생긴 데이터인지 아닌지를 판별*** 하는 것이 기본 골자라고 볼 수 있겠습니다.

본 논문에서는 **판별자**를 **위조 지폐를 찾아내는 경찰**에 비유하고, **생성자**를 **위조 지폐를 만들어 내고 이를 무단으로 사용하는**것으로 비유 합니다.

그리고 이것을 게임이라 치면 서로 경쟁하며 학습함으로써, **위조 지폐가 진짜와 구별할 수 없을때 까지** 진행되게 됩니다.

그리고 이 방법의 장점은 어떤 근사적인 추론이나 흔히 우리가 알고 있는 Markov chain과 같은 것들이 필요 없다는 점인데, 이 부분은 나중에 포스팅 해보겠습니다.

아무튼, 그렇다면 **진짜와 가짜가 구별할 수 없을때 까지가 언제인가?** 인데, 이것을 생성자(G)와 판별자(D)의 서로간의 최소, 최대값을 만족하는 value function으로 다음과 같이 정의하고 있습니다.

논문에서는 이를 minmax game으로 부르며, 생성자와 판별자라는 두 명의 플레이어로 서술하고 있습니다.

![image](https://user-images.githubusercontent.com/91790368/152825399-24c14cac-2430-4819-aa17-09726d935330.png)

이 value function에는 더하기로 연결된 두 가지의 항이 있는데, **전자를 판별자(D)의 항**, **후자를 생성자(G)의 항**이라고 생각해 볼 수 있습니다.

이때, 전자의 항은 판별자의 일종의 목표인 셈인데 여기서 D(x)는 변수 x가 실제 데이터로 부터 왔을 확률을 의미 합니다. 

판별자가 판단해봤을때, 어떤 지폐에 대해서 이 지폐가 진짜인지 위조 지폐인지에 대한 확률을 부여하는 것으로 생각하면 됩니다.

그럼 이 항에 log를 취해줬고, 확률은 0과 1사이의 값을 가지니 자연스레 판별자는 **log D(x)를 최대화** 하기 위해 노력할 것 입니다. 즉, 더 정확하게 맞추려고 노력하겠죠.

반면에 후자의 항에서 D(G(z))는 생성자가 생성한 이미지를 바탕으로 판별자가 판단한 확률을 의미합니다.

이 경우에도 역시, 생성자는 판별자를 최대한 속이기 위해 즉, D(G(z))를 0으로 만들기 위해 노력할 것입니다.

그러니까, 판별자의 경우는 확률을 최대화 하려하고(max) 생성자의 경우는 확률을 최소화 하려고 합니다(min).

그런데 여기에서 재밌는 것은, 만약 생성자가 이미지를 자꾸 부실하게 만들어 낸다면 어떨까요?

그렇게 된다면 판별자는 아주 높은 확률로 생성자의 이미지를 거절(reject)할 것입니다. 즉, 판별자는 아주 당당하게 위조 지폐를 가려내겠죠.

때문에 D(G(z))는 1에 가까워져 갈 것이고, 그렇게 되면 log (1- D(G(z)))값은 log 0에 가까워 지므로 결국 모델이 망가지게 됩니다(논문에서는 saturate이라는 표현 사용).

그러면 이 두 모델이 결국 수렴할 수 있는, 서로의 전역 최적점을 찾을 수 있을까요?

수학적으로는 그렇다고 합니다. 

이 식을 적분으로 풀어쓴 뒤, Kullback-Leibler divergence와 Jensen-Shannon divergence로 식을 전개하면 결국 -log 4라는 값을 찾게 되는데,

이 -log 4가 결국 전역 최적점이 됩니다.

더 자세한 내용이 궁금하시면 해당 논문의 **"4.1 Global Optimality of Pg = Pdata"** 부분을 보시면 될 것 같습니다.

아무튼 정리를 해보자면 다음과 같습니다.

## GAN구조 정리

1. 심층 생성적 모델의 한계점에서 나아가기 위해 **경쟁적 모델을 제안**한다.

2. 이 모델은 기본적으로 **생성자**와 **판별자**가 있는데 이 두 그룹은 각각 **위조 지폐를 찍어내는 범죄자**와 **이 범죄자를 잡으려는 경찰**로 비유할 수 있다.

3. 생성자는 판별자를 속이기 위해, 판별자는 더욱 잘 분류하기 위해 서로 경쟁한다.

<a id="2"></a>
## Code 구현

코드를 구현하고 결과를 살펴보겠습니다.

정확한 코드는 제 [깃허브](https://github.com/Pythonash/Projects/blob/Brain/GAN%EC%9D%84%20%EC%9D%B4%EC%9A%A9%ED%95%9C%20%EC%BB%AC%EB%9F%AC%EC%9D%B4%EB%AF%B8%EC%A7%80%20%EA%B5%AC%ED%98%84%20.ipynb)를 클릭하시면 보다 상세하게 보실 수 있습니다.

<a id="2.1"></a>
## 데이터셋

데이터셋의 경우 케라스에서 제공하는 CIFAR100 데이터셋을 이용했습니다.

보다 선명한 사진으로 하고 싶었는데... 그래도 결국 본질이 중요한거니깐요... 일단 진행하겠습니다.


먼저 데이터셋을 불러오겠습니다. 그리고 나서 이 이미지가 어떻게 생겼는지 보겠습니다.
<pre>
<code>

(x_train, y_train), (x_test, y_test) = tf.keras.datasets.cifar100.load_data()
plt.imshow(x_train[0])

</code>
</pre>

[실행결과]

![image](https://user-images.githubusercontent.com/91790368/152830417-0d6996f1-3c65-46a9-9a30-fa1a834dbf57.png)

아주 우직한 소 한마리가 있네요. 근데 사진이 흐려서 잘 보이진 않습니다 ㅋㅋ.. 소 맞겠죠?

그 다음에는 데이터셋들을 합쳐줄 겁니다. 원래 불러올 때에는 훈련용, 테스트용 데이터셋을 불러왔는데 이걸 하나로 합쳐서 6만개 데이터셋으로 만들어 줄려고 합니다.

<pre>
<code>

df = np.concatenate([x_train, x_test])
print(df.shape)

</code>
</pre>

[실행결과]

<pre>(60000, 32, 32, 3)
</pre>

데이터가 잘 합쳐진 것 같습니다. 각 숫자가 의미하는 것은 데이터 6만개, 32 x 32 크기의 사진, RGB 세 개의 채널을 가진 컬러이미지 라는 뜻 입니다.

이제 모델을 구축해보겠습니다.


<a id="2.2"></a>
## 모델구현

<pre>
<code>

size = 100

generator = tf.keras.models.Sequential([
    tf.keras.layers.Dense(8 * 8 * 128, input_shape = [size]),
    tf.keras.layers.Reshape([8, 8, 128]),
    tf.keras.layers.BatchNormalization(),
    tf.keras.layers.Conv2DTranspose(64, kernel_size = 5, strides=2, padding='same', activation ='elu', kernel_initializer = 'he_normal'),
    tf.keras.layers.BatchNormalization(),
    tf.keras.layers.Conv2DTranspose(32, kernel_size = 5, strides=2, padding='same', activation = 'elu',kernel_initializer = 'he_normal'),
    tf.keras.layers.BatchNormalization(),
    tf.keras.layers.Dropout(0.4),
    tf.keras.layers.Conv2DTranspose(3, kernel_size = 5, strides=1, padding='same', activation = 'elu',kernel_initializer = 'he_normal')
])

discriminator = tf.keras.models.Sequential([
    tf.keras.layers.Conv2D(128, kernel_size = 5, strides =2, padding='same', activation = tf.keras.layers.LeakyReLU(0.3),
                          input_shape=[32,32,3], kernel_initializer = 'he_normal'),
    tf.keras.layers.Dropout(0.5),
    tf.keras.layers.BatchNormalization(),
    tf.keras.layers.Conv2D(256, kernel_size = 5, strides=2, padding='same',
                          activation = tf.keras.layers.LeakyReLU(0.3), kernel_initializer = 'he_normal'),
    tf.keras.layers.Dropout(0.4),
    tf.keras.layers.Flatten(),
    tf.keras.layers.BatchNormalization(),
    tf.keras.layers.Dense(1, activation='sigmoid')
])

gan = tf.keras.models.Sequential([generator, discriminator])
gan.summary()

</code>
</pre>

일단 제일 위에부터 설명드리겠습니다.

**size**는 일종의 seed number와 같이 생각하시면 됩니다. 

> 생성자 모델의 입력과 대응되는 부분인데, 얼만큼의 사이즈를 넣어줄지 결정하는 겁니다.

> 하이퍼 파라미터처럼 여러 값을 시도해보실 수 있습니다.

**generator**는 생성자 모델을 구축한 겁니다.

> 중간 중간 BatchNormalization을 해줬고, 전치합성곱을 쌓아줬습니다. 

> 폭주를 막기위해서 드롭아웃층과 효율적인 학습을 위해 he_normal을 이용한 ReLU함수의 변종을 사용했습니다.

**discriminator**는 판별자 모델을 구축한 겁니다.

> 생성자모델은 출력층에서 (32,32,3)의 차원을 갖는 데이터를 뱉어낼 겁니다. 그러면 다시 (32,32,3)의 차원을 갖는 입력층을 판별자에 쌓아줍니다.

> 그리고 역시 BatchNormalization과 드롭아웃을 추가하고 마지막에 이 데이터를 일렬로 펼치는 Flatten과 마지막에 진짜인지 가짜인지를 가려낼 시그모이드 활성화 유닛을 쌓아줍니다.

**gan**은 생성자와 판별자 모델을 합쳐준 겁니다.

> 이제 생성자가 랜덤 벡터로부터 입력을 받아서 아무 이미지나 뱉어내면, 판별자가 그 이미지를 집어삼킨후 진짜인지 가짜인지를 판별할 겁니다.

> 그러면 그 판별자를 보고, 생성자는 더 잘 속이기 위해 점점 진짜와 같은 이미지를 만들어 낼 겁니다.

[실행결과]

<pre>Model: "sequential_3"
_________________________________________________________________
Layer (type)                 Output Shape              Param #   
=================================================================
sequential_1 (Sequential)   (None, 32, 32, 3)         1086787   
_________________________________________________________________
sequential_2 (Sequential)   (None, 1)                 911617    
=================================================================
Total params: 1,998,404
Trainable params: 1,964,932
Non-trainable params: 33,472
_________________________________________________________________
</pre>

참고로 여기서 Non-trainable params라고해서 업데이트 되지 않는 파라미터들이 있는데 이는 BatchNormalization으로 인해 생기는 파라미터들 입니다.

그럼 이제 이미지 데이터를 Min-Max 스케일링을 해주고, 다시 그 값을 -1과 1사이의 값으로 조정해줄 겁니다.

왜냐하면, 원래 이 모델은 "심층 합성곱 GAN" 이라는 것을 골자로 파라미터를 제가 바꾼 것인데 기존 모델에서는 생성자 모델의 출력층에 하이퍼볼릭 탄젠트 함수(-1과 1사이의 값 출력)를 썼기 때문입니다.

근데 저는 elu로 해도 이게 잘 작동하는지 보기위해 생성자 출력층에 그냥 elu 활성화 함수를 사용했는데, 이걸 참고하시는 분들은 굳이 -1과 1사이로 값을 조정해주지 않고 정규화(Min-Max)만 해주셔도 됩니다.

<pre>
<code>

image = df/255
imgae = image * 2. -1.

</code>
</pre>


## 데이터셋 적재

이제 이 데이터셋들을 tensor로 바꾸고 배치로 꺼내서 학습시켜주기 위해 다음과 같이 작업합니다.

<pre>
<code>

batch_size = 30
dataset = tf.data.Dataset.from_tensor_slices(image)
dataset = dataset.shuffle(2022)
dataset = dataset.batch(batch_size, drop_remainder = True).prefetch(1)

</code>
</pre>

배치 사이즈는 30으로 일단 해줬습니다. 근데 다르게 하시더라도 뒤에 drop_remainder = True로 지정해주시면 괜찮습니다.

또, 여기서 prefetch(1)을 하게 되면 더욱더 빠르게 학습이 가능하다고 알고 있습니다.

학습할때 배치마다 하나씩 데이터를 준비시키는 원리라고 하더라구요.

이제 데이터셋은 준비가 됐고, 다시 모델 구축을 마무리 해보겠습니다.

## 판별자 프리징

GAN의 경우에는 기존 신경망 학습과는 다르게 두 단계에 걸쳐 진행이 됩니다.

먼저 생성자가 데이터를 생성하고 그걸 판별자가 학습합니다.

그 다음 gan을 통해 모델의 가중치를 생성자가 받아 업데이트를 하는 방식인데요, 말하자면 생성자와 판별자는 동시에 학습이 되면 안된다는 겁니다.

또한, 훈련 함수르 따로 작성해줄건데 그때 판별자의 가중치를 얼리고 녹이고를 반복할 것이라 먼저 얼려줍니다.

<pre>
<code>

discriminator.compile(loss = 'binary_crossentropy', optimizer = 'rmsprop')
discriminator.trainable = False

</code>
</pre>

## GAN 컴파일

이제 본격적으로 학습에 들어가기 전에 GAN을 컴파일 해줍니다. 그러면 모델의 구조가 이렇구나~ 하고 인식하게 될겁니다.


<a id="2.3"></a>
## 훈련함수 작성

앞서 말씀드렸듯이, GAN은 조금 특별한 방식으로 훈련이 됩니다.

따라서 기존의 fit()을 호출해서 학습시키지 않고, 따로 훈련함수를 만들어 주겠습니다.

<pre>
<code>

def train_gan(gan, dataset, batch_size, codings_size, n_epochs=50):
  generator, discriminator = gan.layers
  for epoch in range(n_epochs):
    print("Epoch {}/{}".format(epoch + 1, n_epochs))
    for X_batch in dataset:
      noise = tf.random.normal(shape = [batch_size, codings_size])
      generated_images = generator(noise)
      generated_images = tf.cast(generated_images, tf.float64)
      X_fake_and_real = tf.concat([generated_images, X_batch], axis=0)
      y1 = tf.constant([[0.]] * batch_size + [[1.]] * batch_size)
      discriminator.trainable = True
      discriminator.train_on_batch(X_fake_and_real, y1)

      noise = tf.random.normal(shape=[batch_size, codings_size])
      y2 = tf.constant([[1.]] * batch_size)
      discriminator.trainable = False
      gan.train_on_batch(noise, y2)
      
</code>
</pre>

이 코드는 "핸즈온 머신러닝 2판"을 보고 공부할 때 연습했던 건데 원래 코드는 흑백 이미지(채널이 1개)에 맞춰져 있어서 컬러 이미지(채널이 3개)에도 적용할 수 있게 바꿨습니다.

참고로, 중간에 tf.cast(generated_images, tf.float64) 이 부분은 double tensor 에러가 뜨시는 분들을 위해 추가해놓은건데

**생성되는 이미지의 텐서 데이터 타입(코드에서는 generated_images를 말함)**과 **데이터 적재로 꺼내는 텐서 데이터 타입(코드에서는 X_batch를 말함)**이 다르면 double tensor에러로 충돌하게 됩니다.

따라서, 만약 에러가 뜨신다면 데이터 타입을 적절히 변환하셔야 합니다.

**noise** 는 generator에 넣어줄 랜덤 벡터를 말하는 겁니다.

쉽게 말해서, generator는 아무 값이나 입력받고 그걸 적절하게 소화시켜서 이미지를 뱉어낸다고 보시면 됩니다.

또, 먼저 판별자를 녹이고(trainable = True) 학습을 시켜준뒤 그 다음에 판별자를 다시 얼리고(trainable = False) gan을 학습시켜주는 것을 보실 수 있습니다.

이처럼 gan이 학습될때는 판별자의 가중치가 업데이트 되면 안됩니다. 오직 생성자만이 그 가중치를 전달받아 업데이트 되어야 하는 것 입니다.


그래서 이걸 실행시키면 다음과 같습니다.


<a id="2.4"></a>
## GAN 훈련

<pre>
<code>

train_gan(gan, dataset, batch_size, size, n_epochs=10)

</code>
</pre>

이때 저는 에폭을 10으로 설정했는데, 더 잘 훈련을 시키기 위해서는 에폭을 늘려야 합니다(혹은 하이퍼 파라미터 튜닝이 필요합니다).

그럼 10번의 에폭을 마치고나서 생성된 결과를 한번 볼게요.

<pre> 
<code>

plt.imshow(generator(noise)[0])

</code>
</pre>

[실행결과]

![image](https://user-images.githubusercontent.com/91790368/152838511-d29a1bf3-af4d-42e0-a0aa-3a9becd58648.png)

혹시 이게 뭔지 아시는 분...???


<a id="3"></a>
# Review

지금까지 GAN을 구현해 봤습니다.

처음에 논문을 읽고 흥미로워서 한번 책 보고 찾아본다음에 적절히 수정해서 써봤는데 정말 재미있었습니다...근데 왜 생성된 사진이 ㅋㅋㅋㅋ

아마 원본 데이터가 너무 흐려서 그런 것 같습니다(튜닝 안한 것도 있지만요).

제가 고양이를 좋아해서 고양이 사진만 엄청 모아서 GAN으로 저만의 고양이를 만들어 볼까 했는데... 고양이 사진만 몇천 몇만 장 있는걸 구하는게 쉽지는 않더라구요 ㅎ

나중에 크롤링으로 한번 모아야 하나 생각도 해봤습니다.

그리고 이걸 colab에서 따라하실 때 gpu를 꼭 켜고 하세요, 안그러면 시간이 정말....

저같은 경우는 jupyter notebook에 tensorflow gpu를 깔아서 했는데, 사실 코랩에서도 잘 돌아갑니다.

근데 나름 맥북 M1프로인데도 에폭 7번째때부터 열받기 시작하더라구요....ㅋㅋㅋㅋㅋ

아무튼 재밌고 즐거운 포스팅이었습니다.

## 언제든 궁금하신 점이나 피드백은 환영입니다!

여기까지 읽어주셔서 감사합니다.

<a id="4"></a>
# References

Geron, A. (2019). Hands on machine learning with scikit-learn and tensorflow 2nd.

Goodfellow, I., Pouget-Abadie, J., Mirza, M., Xu, B., Warde-Farley, D., Ozair, S., Courville, A., Bengio, Y. (2014). Generatvie Adversarial Nets, Advances in Neural Information Processing Systems 27 (NIPS 2014).
