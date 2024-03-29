---
layout: post
title: "피카츄의 기묘한 딥러닝 체험"
date: 2016-02-28 00:00:00
categories: Code
tags: deeplearning opencv cuda caffe gpgpu
---

최근에 나는 프로그래밍을 이용해서 재밌는 것들을 많이 하려고 노력하는 편인데, 프로그래밍을 이용한 영상 편집이 필요하다는 부탁을 듣고 작업을 하면서 딥러닝이 어떤 것인지 아주 조금 체험할 수 있었다. 다만 그 과정이 정말 기괴하고 힘든 삽질들로 가득했기 때문에 조금이나마 기록을 남기고자 한다. 이 글에서는 Ubuntu 14.04.4를 기준으로 설명해보고자 한다. 또한 이 글을 통해 딥러닝을 같이 체험해보고 싶다면 고사양의 Nvidia GPU가 있는 머신에서 작업하길 권장한다. 아니면 Amazon EC2의 GPU 인스턴스도 좋다. (이 글에서는 CUDA를 활용하고 있다. 나는 i7 6700K와 32GB RAM 환경에서 작업했었는데… 그래픽카드에 대해서는 후술하겠다.)

#### 뭘 했는데?

내가 한 것은 뮤직비디오의 편집이었는데 뮤직비디오의 컨셉이 싸이키델릭이었다. 기계 학습을 통해서 기계가 만들어낸 이미지는 정말로 난해했고, 뮤직비디오를 만드는 형님은 이 난해함을 영상에 담고 싶었지만 기존의 영상 편집 기술로는 어려웠다고 한다. 찾아보니까 이러한 것을 **“DeepDream”**이라고 사람들은 부르며 이러한 비디오를 합성할 수 있는 것이 [GitHub](https://github.com/graphific/DeepDreamVideo)에 있었다. 사실 이때는 내가 이렇게 미친 삽질들을 할지는 미쳐 알지 못했다.

#### DeepDreamVideo는 간단해보였다…

처음엔 이게 단순하게 영상을 프레임 단위로 나누어서 이미지 파일로 만들고, 이미지 파일들을 모델 이미지와 섞어서 DeepDream을 구현한 뒤, 그 이미지들을 다시 영상으로 합치는 것이라고 생각했다. 그랬기에 적당히 CPU와 GPU가 좋은 컴퓨터 하나만 있으면 금방할 수 있을 것이라고 생각했다. 다만 내가 생각했던 것은 너무 단순했고 이 안에서 DeepDream을 구현하는 과정이 정말로 스펙터클했다.

우선 DeepDreamVideo는 영상을 프레임 단위로 분해하고 프레임들을 다시 영상으로 합치기 위해서 ffmpeg나 avconv, mplayer 등을 필요로 한다. 또한 jpg나 png에 맞는 이미지 라이브러리도 필요로 하는데, png의 경우 pngcrush를 요구했다. 나는 ffmpeg를 사용했는데 pngcrush 없이 아래와 같이 ffmpeg 커맨드만으로도 png로 추출할 수 있었다.(도와준 [종이](http://twitter.com/hibiyasleep)에게 고맙다는 인사를 전한다.)

{% highlight bash %}
ffmpeg -threads 8 -i master_file.mov -vf fps=30 %08d.png
{% endhighlight %}

그러면 실제로 DeepDream을 만들어내는 Python 코드를 보면 numpy, protobuf, scikit-image, caffe 등을 필요로 한다. caffe를 제외한 셋은 pip를 통해서 설치 가능하다. 문제는 이 caffe였다.

#### CUDA, Caffe와의 씨름

이제 Caffe에 대해서 이야기를 해보려면 CUDA 이야기를 빼놓을 수가 없다. DeepDreamVideo가 사용하고 있는 Caffe는 Berkeley Vision and Learning Center(BVLC)에서 만든 오픈소스 딥러닝 프레임워크다. 이 딥러닝 과정에 GPU 가속을 사용하기 위해서 CUDA 환경이 필요하다. 처음에 CUDA 환경을 구축할 때, 리누스 토발즈가 엔비디아한테 가운데 손가락을 날리면서 “So nVidia, Fu** you”라고 외치는 것이 떠올랐고… 내가 엔비디아 욕을 하지 않길 원했지만 CUDA 가속 때문에 2시간 정도를 날리고 나니까 자연스럽게 입에서 거친 말들이 나왔다.

![](https://cdn-images-1.medium.com/max/1600/1*tIJf0McU91OAqRQOoRJoWg.jpeg)
<span class="figcaption_hack">엔비디아를 향한 리누스 토발즈의 욕설은 유명하다. 한때 내 마음도 그와 같았다.</span>

내가 CUDA 환경을 구축하고 Caffe를 빌드하는데에는 [Reddit에 정리된 글](https://www.reddit.com/r/deepdream/comments/3cd1yf/howto_install_on_ubuntulinux_mint_including_cuda/)을 참고해서 진행했는데, 글이 매우 잘 정리된 편이지만 몇가지 글에서 조심할 점들을 이야기해보겠다. 이것들을 고치고 나서 나는 CUDA 환경을 구축할 수 있었다.

1.  “/etc/modprobe.d/blacklist.conf”를 설정하지 말아라. 나중에 씁쓸한 오류를 맛보게 될 것이다.
1.  처음에 “sudo su”로 root 유저로 들어가면 “~/”로 하는 것들이 사용자의 유저 디렉토리가 아니라 root 유저 디렉토리에 진행되므로,
나중에 일관성이 깨진다. sudo를 계속 사용하거나 아니면 root 계정에서 진행을 하자. 나는 전자를 추천한다.
1.  Cuda 7.0을 계속 설치해보려고 했지만 Ubuntu 14.04.4에서는 잘 되지 않았다. 여러번 설치와 삭제를 반복해본 뒤에 나는 Cuda
7.5로 설치를 성공했다.
1.  Ubuntu 14.04.4 기준 mdm이라는 service는 존재하지 않는다. Cuda(와 엔비디아의 드라이버)를 설치하기 위해서는 X
Window는 꺼져있어야 한다. 그러기 위해서는 “sudo service lightdm stop / start / restart”를 이용하자.

Caffe 빌드를 위해서는 위에서 언급한 Reddit의 글대로 진행하면 되는데 나는 CUDNN이나 CPU_ONLY, Intel Composer XE같은 것들의 설정을 하지 않고 바로 빌드를 했다. 다른 점이 있다면 나는 [엔비디아에서 Fork한 Caffe](https://github.com/NVIDIA/caffe)를 빌드했다는 점이다. 둘 다 써본 결과, 큰 차이가 없기 때문에 어느
쪽을 사용해도 큰 차이는 없을 것이다.

Google의 DeepDream을 설치할 필요는 없지만, Caffe 내의 모델로 “bvlc_googlenet.caffemodel”는 꼭 설치해야한다.

#### 이제 진짜 DeepDreamVideo를…?

사실 여기까지 보는 사람들은 별 것 아니라고 생각할 것이지만, 본인은 딥러닝 등에 대한 경험이 전무했고 소위 초고수가 아니기 때문에 4시간을 삽질하면서 구축했기 때문에 매우 험난하고 지친 과정이었다.

![](https://cdn-images-1.medium.com/max/1600/1*BtpZhonatJN9AnSxjpssdg.png)
<span class="figcaption_hack">나는 그분처럼 전체 글을 봐도 내가 어떻게 환경을 구축해야할지 감이 잘 오지 않았다.</span>

동영상을 프레임으로 쪼개는 작업은 이미 끝났고, python을 이용해서 아래와 같이 실행을 했다.

{% highlight bash %}
python dreaming_time.py -it png -i results -o output_”$1" -gi /media/user/image-dreamer/models/”$1".jpg —-gpu 0 -b random -t ~/caffe/models/bvlc_googlenet/ -m bvlc_googlenet.caffemodel
{% endhighlight %}

“어? 아… 안 되잖아? 이런 일이 일어날 것 같은 조짐을 느꼈지.” 나는 처음에 GTX960을 이용해서 작업을 진행하려고 했었다. 동영상 파일은 3GB가 넘었고, 프레임들이 저장된 폴더는 13GB가 넘었었다. 2GB의 그래픽 메모리를 가지고 있었던 GTX960은 CUDA에서 Out Of Memory를 외치며 죽어버렸고 나는 이것을 단순히 Caffe에서의 batch_size 문제라고 생각을 했지만… 하드웨어의 한계는 명확했다. 일단 작업하던 컴퓨터를 열고, GTX960을 뽑고 작업하던 스튜디오에서 제일 좋은 성능의 GTX980Ti를 박았다. 당연히 파워를 그대로 썼다간 바로 뻗을거라고 생각해서 스튜디오에 남아있던 1300W짜리 파워를 연결했다.

![](https://cdn-images-1.medium.com/max/1600/1*mik7GgJKDb8N7-JfMHeWew.jpeg)
<span class="figcaption_hack">아무도 ITX 보드에 GTX980Ti를 박을거라고 생각도 못했을걸?</span>

그러고보니 작업용 컴퓨터를 분해하면서 M.2 SSD가 어디에 있는지 한참을 헤맸는데 기가바이트 GA-Z170M-GAMING 5 기준으로는 보드 아랫면에 있었다. 솔직히 좀 충격이었다. 2816 CUDA Core에 6GB 그래픽 메모리가 담긴 크고 아름다운 GTX980Ti를 넣자 그제야 DeepDreamVideo는 멀쩡히 작동하기 시작했다.

#### 마지막 최적화를 향한 나름대로의 발악

전체 뮤직비디오 영상을 프레임으로 쪼개니까 약 8000~9000개의 png 파일로 나뉘었고 이걸 하나하나 DeepDream하게 뽑아내는 작업을 했다. 1 프레임당 3초 정도의 시간이 소요됬으며 이렇게 진행하면 약 7~8시간이 소요되었을 것이라고 나왔다. 이건 아니라고 생각해서 당장 그래픽카드를 오버클럭했는데, Asus의 ROG GTX980Ti Platinum은 애초에 오버클럭용으로 나왔기 때문에 뭔가 걱정은 들지 않았다. 아래 코드의 첫번째 줄과 같이 설정을 해주고 재부팅을 한 뒤 두번째 줄을 실행하자.

{% highlight bash %}
sudo nvidia-xconfig --cool-bits=28
nvidia-settings
{% endhighlight %}

여러가지 정책들을 통해 GPU Fan 소리는 60%로 고정하고, 그래픽 클럭은 100~150MHz의 클럭을 올렸고 그래픽 메모리 클럭은 1000MHz 정도 올렸다. 그럼에도 불구하고 실행 속도가 특별히 개선되거나 하는 일은 없었다. 뮤직비디오를 만드신 형님이 DeepDream을 적용할 구간만 따로 편집해서 다시 주셔서 프레임은 3000~4000개 전후로 작업할 수 있었다.

사실 실제로 돌려보고 나니까 알게된 점은 이미지를 그래픽 메모리에 담아야 해서 그래픽 메모리의 점유율이 높았지 특별히 GPU나 CPU의 연산을 많이 사용하지 않았다. (GPU의 경우 GTX980Ti라서 크게 부담스럽지 않았을 수도 있다.) 다만 하나를 실행하니까 그래픽 메모리 5.2GB를 한번에 Python 프로세스가 차지했을 정도였기 때문에 동시에 여러 딥러닝 프로세스를 돌릴 수는 없었다.

#### 후기와 참조 링크

처음으로 딥러닝을 만져본 후기는 정말 하드웨어빨이 중요하다는 것이었다. 또 관련된 논문을 읽어본 결과 아직 나는 이게 내부에서 어떻게 동작하는지 명확하게 알지도 못하는 정도라는걸 알았고 더 많이 공부하고자 하는 자극이 되었다. 나는 아래 링크들을 많이 참조했으며, 일부 개인적으로 좀 쓰기 편하게 DeepDreamVideo를 수정했다. 그 소스코드는 몇 일 정도 렌더링 작업이 끝나면 GitHub에 문서와 함께 공유하고자 한다.

* [DeepDreamVideo](https://github.com/graphific/DeepDreamVideo)
* [image-dreamer](https://github.com/Dhar/image-dreamer)
* [NVIDIA/caffe](https://github.com/NVIDIA/caffe)
* [DeepDream Dependency Guide](https://www.reddit.com/r/deepdream/comments/3cawxb/what_are_deepdream_images_how_do_i_make_my_own/)
* [Ubuntu CUDA, Caffe Guide](https://www.reddit.com/r/deepdream/comments/3cd1yf/howto_install_on_ubuntulinux_mint_including_cuda/)
* [CUDA Download](https://developer.nvidia.com/cuda-downloads)