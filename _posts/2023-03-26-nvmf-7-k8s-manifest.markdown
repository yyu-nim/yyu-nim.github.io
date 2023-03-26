---
layout: post
title:  "nvmf <7> - availability"
date:   2023-03-26 20:15:00 +0930
categories: rust crossbeam chatgpt nvmf k8s manifest
---

NVMe-oF 가 어떤 이유로든 멈췄다고 생각해보자. Rust로 짠다면 memory safety 이슈로 인한
크래시는 없을 테니 그 부분은 안심할 수 있겠지만, 의도치 않게 무한 루프를 돌면서 객체를 무한 생성해서
Linux OOM 킬러에 의해 종료가 될 수도 있을 것이고, 연동된 C 라이브러리에서의 invalid memory
access bug 이라던지, Kernel 버그로 인해 애플리케이션 포함 전체가 다운된다던지, 
애플리케이션 자체의 버그로 인해 inconsistent state 가 감지되어 assert/panic 에 걸린다던지,
여전히 다양한 시나리오로 인해 unavailability 문제가 발생할 수 있다. 

서비스 중단의 원인이 아주 명확하다면 그걸 해결한 뒤 서비스 재시작을 하는 것이 이상적인 순서이겠으나,
원인이 모호하여 분석에 시간이 좀 더 소요될 수 있고, 항상 발생하는 것이 아니라 우연한 타이밍적인
이슈라고 한다면, "일단은 급한대로" 곧바로 서비스를 재시작하도록 하여 서비스는 available하게 만들고, 
분석을 이어나가는 것이 좋은 방법일 수 있다. 물론 버그의 양상에 따라 다를 수 있으므로 일반화 어려울 수도 있지만.
가령, data corruption이 의심되는 상황이라면 availability vs. data protection 사이에서
어려운 결정을 해야할 수도 있을 것이다.

여하튼, 중단된 서비스를 자동으로 재개시킬 수 있도록 하는 좋은 방법이 무엇이 있을까?
일단은 간단하게 cron을 활용하는 방법이 있을 것이다. 가령, 매분 정해진 스크립트를 실행해서,
우리의 NVMe-oF 애플리케이션의 상태를 체크해서 다운되어 있다면, bring-up 시퀀스를 수행하여
서비스를 시작시키도록 하는 것이다. Puppet/chef 와 같은 인프라 자동화 툴도 마찬가지의 기능을
수행할 수 있을 것이다. 

그런데 이러한 자동화 툴을 쓰더라도, 여전히 "어떻게 애플리케이션의 상태를 체크할 것인지", "어떻게 
bring-up 시퀀스를 수행할 것인지" 부분은 여전히 개발자의 숙제가 될 것이다.
애플리케이션의 상태를 체크할 수 있는 좋은 패턴이 있는지, 애플리케이션의 배포나 config를
선언하여 자동으로 bring-up 할 수 있는 좋은 패턴이 있는지 고민을 하다 보면, 결국 이러한 
결론에 이르게 될 것이다: **"쿠버네티스(k8s)를 사용하자"**. k8s와 관련된 좋은 자료들은 이미 많이 있으므로 그들을 참조하면 어떤 좋은 기능들이 제공되는지
배울 수 있을 것이고, 본 포스팅에서는 NVMe-oF 애플리케이션을 k8s에 어떻게 배포하는게 좋을지에
대해 간단한 고민을 해보려 한다.

k8s에 배포하는 것이 무슨 큰 고민이냐 생각이 들 수도 있지만, 이 배포 방식은 특이하게도
"애플리케이션의 디자인/구조"에 까지 거꾸로 영향을 주는 중대한 결정 요소이다. k8s를 쓰게 되면,
매우 많은 수의 프로세스를 띄우고 이들 간에 통신을 시키는 것이 매우 쉬워지고, 
비정상/정상 종료된 프로세스를 다시 띄우는 것이 쉬워지고,
프로세스의 상태를 지속적으로 polling 하여 원하는 action을 취하는 것이 쉬워진다.
k8s는, 효율적인 리소스 스케줄링을 위해 임의의 시점에 프로세스들을 일부러 죽여 다시 스스로 실행시킬
수도 있고 따라서, 최대한 state 와 logic을 분리하여 process restart를 아주 
"단순한 이벤트"로 여겨지도록 만들어야만 한다.
이것들을 만족시키며&활용하며 애플리케이션을 디자인하면, 그것이 cloud-native design 이 된다 (라고 
개인적으로 생각한다.)

그럼 지금까지의 내용을 바탕으로 간단한 k8s manifest를 생성해 보도록 한다. 하나의
monolithic application을 k8s에 배포하는 것은 큰 재미가 없을 수 있지만, 오늘은
가장 기본적인 manifest만 작성해보고, 추후에 다양한 방식으로 우리의 NVMe-oF를 잘라
cloud-native design으로 변경해 보는 시리즈를 가져보면 좋을 것 같다.

> Hey GPT, I'd like to onboard my NVMe-oF application to k8s. Could you help with writing the manifest up?
> > ChatGPT: Sure, I can help you with that! To create a manifest for your NVMe-oF application in Kubernetes, you'll need to define a few different components. Here's a basic outline of what you'll need:

> > * Namespace - create a Kubernetes namespace to isolate your application from other resources in your cluster. For example:

```yaml
apiVersion: v1
kind: Namespace
metadata:
name: nvmeof-app
```

> > * ConfigMap - a ConfigMap is a way to store configuration data for your application. You can define the IP addresses of your NVMe-oF targets in a ConfigMap. For example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nvmeof-config
data:
  targets.conf: |
    192.168.1.10
    192.168.1.11
    192.168.1.12
```

> > * PersistentVolume - you'll need to define a PersistentVolume to represent your NVMe-oF storage. This will depend on your specific storage configuration, but here's an example:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nvmeof-storage
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  nvme:
    targetPortal: 192.168.1.10
    port: 4420
    nsID: 1
    namespace: nvme-of
```

> > * PersistentVolumeClaim - create a PersistentVolumeClaim to request storage resources from the PersistentVolume you just defined. For example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nvmeof-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  volumeName: nvmeof-storage 
```

> > * Deployment - finally, create a Deployment to manage your application. This will define the container image, the number of replicas, and any required environment variables or command line arguments. For example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nvmeof-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nvmeof-app
  template:
    metadata:
      labels:
        app: nvmeof-app
    spec:
      containers:
        - name: nvmeof-container
          image: your-nvmeof-image:latest
          volumeMounts:
            - name: nvmeof-storage
              mountPath: /mnt/nvmeof
          env:
            - name: TARGETS_CONFIG_FILE
              value: /etc/nvmeof/targets.conf
      volumes:
        - name: nvmeof-storage
          persistentVolumeClaim:
            claimName: nvmeof-pvc
```

사실 내가 원하는 답은 아니었다. 얼핏 보기에, NVMe-oF 타겟에 대한 manifest라기 보다는
NVMe-oF 호스트 (Initiator) 에 대한 manifest 가 나온 것 같아서, ChatGPT를 시켜
약간 손을 보려 하다가, 특이한 section이 눈에 들어왔다. 그것은 PersistentVolume 부분의
**"spec.nvme"** 항목. k8s specification에 NVMe/NVMe-oF 에 대한 내용이
추가 되었나 싶어, ChatGPT에 물어보았다:

> What does "spec.nvme" mean in the PersistentVolume manifest? Is it a standard?
> > ChatGPT: The spec.nvme field in the PersistentVolume manifest is not a standard Kubernetes field, but rather a custom field used by some vendors to specify the parameters for an NVMe device.
> > While the spec.nvme field is not part of the official Kubernetes specification, it is supported by some storage vendors and can be used to create PersistentVolumes for NVMe storage devices. However, it's worth noting that not all Kubernetes distributions or storage vendors may support this field, and you should check the documentation for your specific environment before using it.
> 
> What storage vendors support "spec.nvme" field?
> > ChatGPT: The spec.nvme field is typically used by storage vendors that provide NVMe storage solutions for Kubernetes. Some examples of storage vendors that support the spec.nvme field include:
> >
> > > Pure Storage - Pure Storage provides a Kubernetes plugin called the Pure Service Orchestrator (PSO) that supports NVMe-oF storage. The spec.nvme field can be used to specify the NVMe-oF target portal and namespace parameters for a Pure Storage FlashArray.
> > >
> > > Dell EMC - Dell EMC provides a Kubernetes plugin called the Dell EMC Container Storage Interface (CSI) that supports NVMe-oF storage. The spec.nvme field can be used to specify the NVMe-oF target portal and namespace parameters for a Dell EMC PowerMax or Unity storage array.
> > >
> > > NetApp - NetApp provides a Kubernetes plugin called Trident that supports NVMe-oF storage. The spec.nvme field can be used to specify the NVMe-oF target portal and namespace parameters for a NetApp AFF or FAS storage system. 

이런 양질의 정보 (?)가 질문 몇번에 나온다는게 너무 신기했고, 이 내용 보는 순간 원래의
k8s manifest 작성보다, storage vendor들이 k8s 에 어떻게 custom spec을 끼워넣어
사용을 하는지, 특히 NVMe-oF 관련 어떤 기능들을 추가했는지 궁금해지기 시작했다. 
깨달음을 준 ChatGPT에게 감사하며, 나머지 내용은 추후 포스트에서 이어가겠다.