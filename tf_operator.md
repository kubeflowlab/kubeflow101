## 7.1.tf_operator
### 7.1.1.CRD

Kubernetes基础资源类型的表达能力是有限的，哪怕是经过了内置的 controller 的扩展。因此 K8s 支持 Custom Resource Definition，也就是我们一直提到的 CRD。通过这一特性，用户可以自己定义资源类型，并对其提供支持。相比于之前的方式，这样的实现更加原生一点，Kubernetes 会将其视为资源的一种，apiserver 中的各种检查对其也是起作用的。因此 CRD 是可以起到四两拨千斤的作用，与其相关的生命周期的管理是由用户代码进行的，与此同时 Kubernetes 的一些公共特性，比如 kubectl 展示，namespace，权限管理等等都是可以被复用的。以 kubeflow/tf-operator 为代表的 operator 都是利用了 CRD 这一特性对 TensorFlow，Prometheus 或 etcd 等不同的系统或框架进行了支持。

让我们从一个 Kubernetes 用户的角度，来看CRD的支持。首先，与声明其他资源的方式一样，在创建自定义的资源时，我们需要一个定义的文件，通常是 YAML 格式的。随后，我们的会利用 kubectl create 命令来创建出对应的资源，这时，Kubernetes 会负责处理一系列对象创建的工作，随后我们可以利用 kubectl describe 命令来获得创建的对象的相关信息。

CRD 是一切的开始，因此从这里出发。CRD 的定义并没有太多值得注意的地方，只是有一些惯例需要稍微关注一下。比如 CRD 的 name 通常是 plural 和 group 的结合。另外，一般来说 CRD 的作用域是 namespaced 就可以了。还有 kind 一般采用驼峰命名法等等。这里给出一个例子，不再赘述。

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tfjobs.kubeflow.org
spec:
  group: kubeflow.org
  version: v1alpha2
  scope: Namespaced
  names:
    kind: TFJob
    singular: tfjob
    plural: tfjobs
```

### 7.1.2 TFJob 是什么？

TFJob 是 Kubernetes 的定制资源，您可以使用它在 Kubernetes 上运行 TensorFlow 训练作业。TFJob 的 Kubeflow 实现在 tf-operator(https://github.com/kubeflow/tf-operator) 中。

TFJob是一个带有 YAML 表示的资源，如下图所示:

```yaml
apiVersion: kubeflow.org/v1
kind: TFJob
metadata:
  generateName: tfjob
  namespace: kubeflow
spec:
  tfReplicaSpecs:
    PS:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: tensorflow
            image: gcr.io/your-project/your-image
            command:
              - python
              - -m
              - trainer.task
              - --batch_size=32
              - --training_steps=1000
    Worker:
      replicas: 3
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: tensorflow
            image: gcr.io/your-project/your-image
            command:
              - python
              - -m
              - trainer.task
              - --batch_size=32
              - --training_steps=1000
```

如果你想让你的 TFJob pod 访问 credentials secrets ，比如当你进行基于 GKE 的 Kubeflow 安装时自动创建的 GCP credentials ，你可以像这样挂载和使用你的secret:

```yaml
apiVersion: kubeflow.org/v1
kind: TFJob
metadata:
  generateName: tfjob
  namespace: kubeflow
spec:
  tfReplicaSpecs:
    PS:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: tensorflow
            image: gcr.io/your-project/your-image
            command:
              - python
              - -m
              - trainer.task
              - --batch_size=32
              - --training_steps=1000
            env:
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/secrets/user-gcp-sa.json"
            volumeMounts:
            - name: sa
              mountPath: "/etc/secrets"
              readOnly: true
          volumes:
          - name: sa
            secret:
              secretName: user-gcp-sa
    Worker:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
          - name: tensorflow
            image: gcr.io/your-project/your-image
            command:
              - python
              - -m
              - trainer.task
              - --batch_size=32
              - --training_steps=1000
            env:
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: "/etc/secrets/user-gcp-sa.json"
            volumeMounts:
            - name: sa
              mountPath: "/etc/secrets"
              readOnly: true
          volumes:
          - name: sa
            secret:
              secretName: user-gcp-sa
```
如果您不熟悉 Kubernetes 资源，请参阅本书之前章节对于 Kubernetes 对象的解释。

TFJob 与内置 controller 的不同之处在于，TFJob 规范旨在管理分布式 TensorFlow 训练作业。分布式 TensorFlow 作业通常包含以下0个或多个步骤：

- Chief 负责控制训练过程并执行诸如检查模型之类的任务。
- Ps 作为参数服务器;这些服务器为模型参数提供了一个分布式数据存储。
- Worker 真正在做模型训练的工作。在某些情况下，worker 0 也可以充当 chief。
- Evaluator 当模型被训练时，evaluator 可以用来计算评估指标。

TFJob 规范中的字段 tfReplicaSpecs 包含从副本类型(如上所列)到该副本的 TFReplicaSpec 的映射。TFReplicaSpec 由3个字段组成

+ replicas 要为这个 TFJob 生成的这种类型的副本的数量。
+ template 一个描述要为每个副本创建的pod的PodTemplateSpec。
  + pod必须包含一个名为 tensorflow 的容器。
+ restartPolicy 确定 pod 退出时是否会重新启动。允许的值如下
  + Always 意味着 pod 总是会重新启动。这个策略对参数服务器很好，因为它们从不退出，并且应该在发生故障时重新启动。
  + OnFailure 表示如果 Pod 由于故障退出，它将重新启动。
    + 非零退出码表示失败。
    + 退出码为0表示成功，pod 将不会重新启动。
    + 这项规则对 chief 和 worker 都有好处
  + ExitCode 表示重启行为依赖于 tensorflow 容器的退出码，如下所示:
    + 退出代码0表示流程已成功完成，不会重新启动。
    + 以下退出码表示永久错误，容器将不会重新启动:
      + 1: 一般的错误
      + 2: 误用shell内置程序
      + 126: 调用的命令无法执行
      + 127: 命令没有找到
      + 128: 退出无效参数
      + 139: 容器被 SIGSEGV 终止(无效内存引用)
    + 下面的退出码显示一个可重试错误，容器将重新启动:
      + 130: 以SIGINT结尾的容器(键盘 Control-C)
      + 137: 容器收到一个SIGKILL
      + 143：容器收到一个SIGTERM
    + 退出代码138对应于SIGUSR1，保留给用户指定的可重试错误。
    + 其他退出代码是未定义的，并且不能保证其行为。
    有关退出代码的背景信息，请参阅GNU终止信号指南和Linux文档项目。
  + Never 意味着终止的 Pod 将永远不会重新启动。这个策略应该很少使用，因为 Kubernetes 会因为各种原因终止 pod (例如节点变得不健康)，而这个策略将阻止作业的恢复。
### 7.1.2 快速上手

#### 提交一个 TensorFlow 训练作业

注意: 在提交训练作业之前，您应该将 kubeflow 部署到集群中。这样做可以确保在提交训练作业时TFJob自定义资源可用。

#### 运行 MNist 示例
Kubeflow 附带了一个适合运行简单 MNist 模型的示例(https://github.com/kubeflow/tf-operator/tree/master/examples/v1/mnist_with_summaries)。

```
git clone https://github.com/kubeflow/tf-operator
cd tf-operator/examples/v1/mnist_with_summaries
# Deploy the event volume
kubectl apply -f tfevent-volume
# Submit the TFJob
kubectl apply -f tf_job_mnist.yaml
```
监视作业:
kubectl -n kubeflow get tfjob mnist -o yaml
删除它：
kubectl -n kubeflow delete tfjob mnist

#### 定制 TFJob

通常，您可以在TFJob yaml文件中更改以下值:
+ 将镜像更改为指向包含你代码的 docker 镜像
+ 更改副本的数量和类型
+ 更改分配给每个资源的资源(请求和限制)
+ 设置任何环境变量
  + 例如，您可能需要配置各种环境变量来与诸如GCS或S3之类的数据存储进行通信
+ 如果您想使用pv存储，请添加pv。
 Using GPUs
To use GPUs your cluster must be configured to use GPUs.

Nodes must have GPUs attached.
The Kubernetes cluster must recognize the nvidia.com/gpu resource type.
GPU drivers must be installed on the cluster.
For more information:
Kubernetes instructions for scheduling GPUs
GKE instructions
EKS instructions
To attach GPUs specify the GPU resource on the container in the replicas that should contain the GPUs; for example.
### 7.1.3 使用 GPU

要使用gpu，您的集群必须配置为使用gpu。
+ 节点必须添加 gpu。
+ Kubernetes 集群必须识别 nvidia.com/gpu 资源类型。
+ GPU 驱动程序必须安装在集群上。
+ 更多信息:
  + Kubernetes 指令调度 gpu（ 参阅 https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/）
  + GKE指令（ 参阅  https://cloud.google.com/kubernetes-engine/docs/concepts/gpus）
  + EKS指令（ 参阅  https://docs.aws.amazon.com/eks/latest/userguide/gpu-ami.html）
要添加 GPU，请在应该包含 GPU 的副本中指定容器上的 GPU 资源， 举例：


```yaml
apiVersion: "kubeflow.org/v1"
kind: "TFJob"
metadata:
  name: "tf-smoke-gpu"
spec:
  tfReplicaSpecs:
    PS:
      replicas: 1
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
          - args:
            - python
            - tf_cnn_benchmarks.py
            - --batch_size=32
            - --model=resnet50
            - --variable_update=parameter_server
            - --flush_stdout=true
            - --num_gpus=1
            - --local_parameter_device=cpu
            - --device=cpu
            - --data_format=NHWC
            image: gcr.io/kubeflow/tf-benchmarks-cpu:v20171202-bdab599-dirty-284af3
            name: tensorflow
            ports:
            - containerPort: 2222
              name: tfjob-port
            resources:
              limits:
                cpu: '1'
            workingDir: /opt/tf-benchmarks/scripts/tf_cnn_benchmarks
          restartPolicy: OnFailure
    Worker:
      replicas: 1
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
          - args:
            - python
            - tf_cnn_benchmarks.py
            - --batch_size=32
            - --model=resnet50
            - --variable_update=parameter_server
            - --flush_stdout=true
            - --num_gpus=1
            - --local_parameter_device=cpu
            - --device=gpu
            - --data_format=NHWC
            image: gcr.io/kubeflow/tf-benchmarks-gpu:v20171202-bdab599-dirty-284af3
            name: tensorflow
            ports:
            - containerPort: 2222
              name: tfjob-port
            resources:
              limits:
                nvidia.com/gpu: 1
            workingDir: /opt/tf-benchmarks/scripts/tf_cnn_benchmarks
          restartPolicy: OnFailure
```
跟随 tensorflow 的指导 (https://www.tensorflow.org/totorials/using_gpu) 使用 GPU 。

### 7.1.4 监控作业

了解作业的状态：

```bash
kubectl get -o yaml tfjobs ${JOB}
```
下面是一个作业的示例输出：

```yaml
apiVersion: kubeflow.org/v1
kind: TFJob
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"kubeflow.org/v1","kind":"TFJob","metadata":{"annotations":{},"name":"mnist","namespace":"kubeflow"},"spec":{"cleanPodPolicy":"None","tfReplicaSpecs":{"Worker":{"replicas":1,"restartPolicy":"Never","template":{"spec":{"containers":[{"command":["python","/var/tf_mnist/mnist_with_summaries.py","--log_dir=/train","--learning_rate=0.01","--batch_size=150"],"image":"gcr.io/kubeflow-ci/tf-mnist-with-summaries:1.0","name":"tensorflow","volumeMounts":[{"mountPath":"/train","name":"training"}]}],"volumes":[{"name":"training","persistentVolumeClaim":{"claimName":"tfevent-volume"}}]}}}}}}
  creationTimestamp: "2019-07-16T02:44:38Z"
  generation: 1
  name: mnist
  namespace: kubeflow
  resourceVersion: "10429537"
  selfLink: /apis/kubeflow.org/v1/namespaces/kubeflow/tfjobs/mnist
  uid: a77b9fb4-a773-11e9-91fe-42010a960094
spec:
  cleanPodPolicy: None
  tfReplicaSpecs:
    Worker:
      replicas: 1
      restartPolicy: Never
      template:
        spec:
          containers:
          - command:
            - python
            - /var/tf_mnist/mnist_with_summaries.py
            - --log_dir=/train
            - --learning_rate=0.01
            - --batch_size=150
            image: gcr.io/kubeflow-ci/tf-mnist-with-summaries:1.0
            name: tensorflow
            volumeMounts:
            - mountPath: /train
              name: training
          volumes:
          - name: training
            persistentVolumeClaim:
              claimName: tfevent-volume
status:
  completionTime: "2019-07-16T02:45:23Z"
  conditions:
  - lastTransitionTime: "2019-07-16T02:44:38Z"
    lastUpdateTime: "2019-07-16T02:44:38Z"
    message: TFJob mnist is created.
    reason: TFJobCreated
    status: "True"
    type: Created
  - lastTransitionTime: "2019-07-16T02:45:20Z"
    lastUpdateTime: "2019-07-16T02:45:20Z"
    message: TFJob mnist is running.
    reason: TFJobRunning
    status: "True"
    type: Running
  replicaStatuses:
    Worker:
      running: 1
  startTime: "2019-07-16T02:44:38Z"

```
#### 条件

TFJob 又一个 TFJobStatus，其中包含一系列 TFJobConditions，去验证 TFJob 是否通过这些条件。TFJobCondition 数组的每个元素都有六个可能的字段:

+ lastUpdateTime 字段提供了该条件最后一次更新的时间。
+ lastTransitionTime 字段提供条件上一次从一个状态转换到另一个状态的时间。
+ message 字段是一条可读的消息，指示关于转换的详细信息。
+ reason 字段是特殊的、一个驼峰的单词，用于条件的上一次转换。
+ status字段是一个字符串，可能的值为“True”、“False” 和 “Unknown”。
+ type 字段是一个字符串，包含以下可能的值:
  + TFJobCreated 意味着 TFJob 已被系统接受，但一个或多个 pod 或者 service 还没有被启动。
  + TFJobRunning 意味着该 TFJob 的所有子资源(例如 service 或者 pod)都已成功调度并启动，并且作业正在运行。
  + TFJobRestarting 表示该 TFJob 的一个或多个子资源(例如 service 或者 pod)出现问题，正在重新启动。
  + TFJobSucceeded 表示作业成功完成。
  + TFJobFailed 表示作业失败。

一个作业的成功或失败决定如下：

+ 一个作业是否有 chief 的成功或失败是由 chief 的状态决定的。
+ 一个作业是否没有 chief 的成功或失败是由 worker 决定的。
+ 在这两种情况下，如果被监视的进程退出时退出码为0，则 TFJob 都将成功。
+ 在非零退出码的情况下，行为由副本的 restartPolicy 决定。
+ 如果 restartPolicy 允许重新启动，那么流程将被重新启动，TFJob将继续执行。
  + 对于 restartPolicy ExitCode，行为依赖于退出码。
  + 如果 restartPolicy 不允许重新启动， 非零退出码则视为永久故障，作业被标记为失败。

tfReplicaStatuses
tfReplicaStatuses provides a map indicating the number of pods for each replica in a given state. There are three possible states

Active is the number of currently running pods.
Succeeded is the number of pods that completed successfully.
Failed is the number of pods that completed with an error.

#### tfReplicaStatuses

tfReplicaStatuses 提供一个映射，指示给定状态下每个副本的 pod 数量。有三种可能的状态：
+ Active 是当前运行的 pod 的数量。
+ Succeeded 是成功完成的 pod 的数量。
+ Failed 是出错的 pod 的数量。
Events
During execution, TFJob will emit events to indicate whats happening such as the creation/deletion of pods and services. Kubernetes doesn’t retain events older than 1 hour by default. To see recent events for a job run

kubectl describe tfjobs ${JOB}
which will produce output like

#### 事件

在执行期间，TFJob 将发出事件来指示发生了什么，比如创建/删除 pod 和 service。默认情况下，Kubernetes 不会保留超过1小时的事件。查看作业运行的最近事件：

```
kubectl describe tfjobs ${JOB}
```
会产生像这样的输出：

```
Name:         mnist
Namespace:    kubeflow
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"kubeflow.org/v1","kind":"TFJob","metadata":{"annotations":{},"name":"mnist","namespace":"kubeflow"},"spec":{"cleanPodPolicy...
API Version:  kubeflow.org/v1
Kind:         TFJob
Metadata:
  Creation Timestamp:  2019-07-16T02:44:38Z
  Generation:          1
  Resource Version:    10429537
  Self Link:           /apis/kubeflow.org/v1/namespaces/kubeflow/tfjobs/mnist
  UID:                 a77b9fb4-a773-11e9-91fe-42010a960094
Spec:
  Clean Pod Policy:  None
  Tf Replica Specs:
    Worker:
      Replicas:        1
      Restart Policy:  Never
      Template:
        Spec:
          Containers:
            Command:
              python
              /var/tf_mnist/mnist_with_summaries.py
              --log_dir=/train
              --learning_rate=0.01
              --batch_size=150
            Image:  gcr.io/kubeflow-ci/tf-mnist-with-summaries:1.0
            Name:   tensorflow
            Volume Mounts:
              Mount Path:  /train
              Name:        training
          Volumes:
            Name:  training
            Persistent Volume Claim:
              Claim Name:  tfevent-volume
Status:
  Completion Time:  2019-07-16T02:45:23Z
  Conditions:
    Last Transition Time:  2019-07-16T02:44:38Z
    Last Update Time:      2019-07-16T02:44:38Z
    Message:               TFJob mnist is created.
    Reason:                TFJobCreated
    Status:                True
    Type:                  Created
    Last Transition Time:  2019-07-16T02:45:20Z
    Last Update Time:      2019-07-16T02:45:20Z
    Message:               TFJob mnist is running.
    Reason:                TFJobRunning
    Status:                True
    Type:                  Running
  Replica Statuses:
    Worker:
      Running:  1
  Start Time:  2019-07-16T02:44:38Z
Events:
  Type    Reason                   Age    From         Message
  ----    ------                   ----   ----         -------
  Normal  SuccessfulCreatePod      8m6s   tf-operator  Created pod: mnist-worker-0
  Normal  SuccessfulCreateService  8m6s   tf-operator  Created service: mnist-worker-0
```

这里的事件表明已经成功创建了 pod 和 service。

### 7.1.5 TensorFlow 日志

日志记录遵循标准的 K8s 日志记录，因此你可以使用 kubectl 为任何没有删除的 pod 获取标准输出/错误。首先找到作业 controller 为副本创建的 pod， pod将被命名

```
${JOBNAME}-${REPLICA-TYPE}-${INDEX}
```
一旦确定了 pod，就可以使用 kubectl 获取日志：

```
kubectl logs ${PODNAME}
```
TFJob 规范中的 CleanPodPolicy 控制在作业终止时 pod 的删除。策略可以是以下值之一：
+ Running 策略意味着只有当作业完成时(例如参数服务器)，才会立即删除仍然运行的pod; 不会删除已完成的pod，以便保留日志。这是默认值。
+ All 策略意味着，当作业完成时，所有的 pod 甚至已完成的 pod 都将被立即删除。
+ None 策略意味着在作业完成时不会删除任何 pod。

如果你的集群利用了 Kubernetes 集群日志记录，那么你的日志也可能被发送到适当的数据存储区，以便进行进一步的分析。

### 7.1.6 故障定位

下面是一些步骤，您可以按照这些步骤来排除作业中的故障

+ 你现在的作业状态如何?运行以下命令：
    ```yaml
    kubectl -n ${NAMESPACE} get tfjobs -o yaml ${JOB_NAME}
    ```
  + 如果生成的输出不包含作业的状态，则通常表示作业规范无效。
  + 如果 TFJob 规范无效，那么 tf operator 日志中应该有一条日志消息

    ```yaml
    kubectl -n ${KUBEFLOW_NAMESPACE} logs `kubectl get pods --selector=name=tf-job-operator -o jsonpath='{.items[0].metadata.name}'` 
    ```
  + KUBEFLOW_NAMESPACE 是你部署 TFJob operator 的命名空间。
+ 检查作业的事件，看看是否创建了 pod
  + 获取事件的方法有很多;如果你是1个小时内启动的作业，你就可以做：
   
    ```yaml
    kubectl -n ${NAMESPACE} describe tfjobs -o yaml ${JOB_NAME} 
    ```

  + 输出的底部应该包含作业发出的事件列表， 如：

 	```yaml
Events:
  Type     Reason                          Age                From         Message
  ----     ------                          ----               ----         -------
  Warning  SettedPodTemplateRestartPolicy  19s (x2 over 19s)  tf-operator  Restart policy in pod template will be overwritten by restart policy in replica spec
  Normal   SuccessfulCreatePod             19s                tf-operator  Created pod: tfjob2-worker-0
  Normal   SuccessfulCreateService         19s                tf-operator  Created service: tfjob2-worker-0
  Normal   SuccessfulCreatePod             19s                tf-operator  Created pod: tfjob2-ps-0
  Normal   SuccessfulCreateService         19s                tf-operator  Created service: tfjob2-ps-0
  ```

  + Kubernetes只保存事件1小时(参见 https://github.com/kubernetes/kubernetes/issues/52521)
    + 根据集群设置事件的不同，可以将其持久化到外部存储中，可在更长的时间内进行访问。
    + 在 GKE 上，事件被保存在 stackdriver 中。
  + 如果没有创建 pod 和 service，则说明没有处理 TFJob ;常见的原因是
    + TFJob规范无效(参见上面)
    + TFJob operator 没有运行
+ 检查 pod 的事件以确保它们是规划好的。
  + 获取事件的方法有很多;如果你的豆荚还不到1小时大，你也可以
  
     ```yaml
    kubectl -n ${NAMESPACE} describe pods ${POD_NAME}
    ```
  + 输出的底部应该包含如下事件：

 	```yaml
Events:
  Type    Reason                 Age   From                                                  Message
  ----    ------                 ----  ----                                                  -------
  Normal  Scheduled              18s   default-scheduler                                     Successfully assigned tfjob2-ps-0 to gke-jl-kf-v0-2-2-default-pool-347936c1-1qkt
  Normal  SuccessfulMountVolume  17s   kubelet, gke-jl-kf-v0-2-2-default-pool-347936c1-1qkt  MountVolume.SetUp succeeded for volume "default-token-h8rnv"
  Normal  Pulled                 17s   kubelet, gke-jl-kf-v0-2-2-default-pool-347936c1-1qkt  Container image "gcr.io/kubeflow/tf-benchmarks-cpu:v20171202-bdab599-dirty-284af3" already present on machine
  Normal  Created                17s   kubelet, gke-jl-kf-v0-2-2-default-pool-347936c1-1qkt  Created container
  Normal  Started                16s   kubelet, gke-jl-kf-v0-2-2-default-pool-347936c1-1qkt  Started container
  ```
  + 阻止容器启动的一些常见问题是
    + 没有足够的资源来调度 pod
    + pod 尝试挂载一个不存在或不可用的卷(或 secret)
    + docker 镜像 不存在或无法访问(例如，由于许可问题)
     
+ 如果容器启动;按照上一节的说明检查容器的日志。