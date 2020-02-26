# Readme for FACESWAP-ORION
>>best to view under vscode with markdown plugin  
>>distributed learning acceleration support by [virtaitech-orion](http://virtai.tech)
## 1 Overview
- To start with `faceswap-orion`, you will need to know the following concepts:
    |Concept|Description|docker-image|
    |-|-|-|
    |*`controller`*|A controller-docker to control jobs assignment |*orion-controller:zhouyu*|
    |*`server`*|A server-docher to handle physical GPUs, execute actual work|*orion-server:zhouyu*|
    |*`client`*|A client-docker with built-in modified `faceswap-orion` environment |*orion-faceswap:zhouyu*|
    Go to [User-Guide](https://github.com/virtaitech/orion/blob/master/doc/Orion-User-Guide.md) if you want more details about the three concepts.
- You will need to start both 3 docker to accually run `faceswap` with **`orion`**.
- Client is your workspace; and you can just leave Server&Controller alone, once you have setup the environment.
- **TIP**:  
    - check your GPU status; sometimes the code crushed because your client loss access to your GPUs.

## 2 Instruction

- a) setup [*`controller`*](README.orion.md##3.1) & [*`server`*](README.orion.md##3.2)
- b) start [*`client`*](README.orion.md##3.3); make sure you have mount your data into the docker.  
- c) run environment test, to make sure you have setup right;  
    within *`client`*, under `/root/`:  
    ```
    sh env_test.sh
    ```
- d) modify `/root/faceswap.example.sh`, and start your first `faceswap-orion` test:
    ```
    sh /root/faceswap.example.sh
    ```
    
## 3 Quick-start

### 3.1 Start *`controller`* docker
-  execute cmd:
    ```
    docker run -itd --net host orion-controller:zhouyu
    ```
    
- *`controller`* will run as a **daemon** and listen at `port 0.0.0.0:9123` as default
- check with
    ```
    ps -aux | grep controller
    ```
    you will see the controller running.
### 3.2 Start *`server`* docker
-  execute cmd:
    ```
    docker run -itd --rm \
            --runtime nvidia \
            --net host \
            --ipc host \
            --pid host \
            --privileged \
            orion-server:zhouyu
    ```
    - -d : run as daemon at background；
    - --net host : allow docker to use host-device network; reduce work for environment settings
    - --ipc host : required for shared memory mode；
    - --pid host : Currently *oriond* strongly required, ensure `nvml` lib can normally run within docker container;
    - --privileged : required for RDMA mode;

- *`server`* will report to *`controller`* at `localhost:9123` as default
- *`server`* will serve at `localhost:9960` as default
- check with
    ```
    ps -aux | grep orion-server
    ```
### 3.3 Start *`client`* docker
- now we can start to setup our workspace environment -- *`client`*
- first, **modify** file `run.orion.faceswap.sh`, make sure you have already mount local dir into this docker container.
    ```
    # [file][run.orion.faceswap.sh]
    sudo docker run -it --rm \
            --ipc host --net host --privileged \
            --name orion.faceswap \
            -v /PATH/TO/YOUR/data/:/root/data/ \
            -w /root/ \
            orion-faceswap:zhouyu
    ```
- start the client with
    ```
    sh run.orion.faceswap.sh
    ```
- now you will access into `client` at `root@host-device:/root$`
- check with 
    ```
    ls /home/
    ```
    you will found nothing in it
- As default, *`client`* will looking for *`controller`* at `localhost:9123` to require GPU resource; and use `ibverbs` mode to communicate with *`server`*
- modify `/etc/orion/client.conf` if you have to **change** your `network settings`

### 3.4 Check Environment
- assume you have gone through [3.1]()-[3.3](); let's check your environment setup!
- assume you are now inside *`client`* docker, when you 
- run the environment test
    - go to `/root/` folder
        ```
        cd /root/
        ```
    - naive network test
        ```
        python3 demo.py
        ```
    - advanced environment test
        ```
        sh env_test.sh
        ```
    if you can't go through the test, go back to [3.1](), [3.2](), [3.3](), to ensure you have set up the orion environment correctly.  
    Be aware of your network settings~ make sure the ip is accessable! 
- when you pass all the environment test, `congratuation`, you are ready to go !!!
### 3.4 play with `faceswap-orion`
- assume you have go through [3.4 check environment]()
- now let's start faceswap-orion
- make sure you are now inside a *`client`*, which has passed environment check `env_test.sh`
- go to `/root/`
    ```
    cd /root/
    ```
- modify `faceswap.example.sh`, to set your data dir
    ```
    export ORION_VGPU=8
    export ORION_GMEM=30000
    ./orion.executor \
        --ngpu ${ORION_VGPU}\
        --pycall "faceswap-orion/faceswap.py \
            train \
            ...
            -bs 16 \
            -it 5000 \
            -s 100 \ 
            -ss 1000 \
            ...
            "
    ```
- execute cmd
    ```
    sh faceswap.example.sh
    ```
- now you can rest and wait for the results.
- once you can see `VIRTAI: TOTAL TIME` and ` [INFO] Releasing Orion resource ...`, the script is ending. go ahead and check your results.
    ```
    [1,0]<stdout>:02/26/2020 04:05:43 INFO     VIRTAI: TOTAL TIME:4845.131490945816(sec); 1.0319637370716541 iter/sec; 132.09135834517173 img/sec
    ...
    [1,3]<stdout>:2020-02-26 04:05:45 [INFO] Releasing Orion resource ...
    [1,1]<stdout>:2020-02-26 04:05:45 [INFO] Releasing Orion resource ...
    [1,0]<stdout>:2020-02-26 04:05:45 [INFO] Releasing Orion resource ...
    [1,2]<stdout>:2020-02-26 04:05:45 [INFO] Releasing Orion resource ...
    [1,0]<stdout>:02/26/2020 04:05:50 INFO     [Saved models] - Average since last save: face_loss_A: 0.03285, face_loss_B: 0.03903
    [1,0]<stdout>:2020-02-26 04:05:51 [INFO] Releasing Orion resource ...
    Executing code
    ```
>> `ATTENTION`: 
>>    - faceswap-orion is a modified version of [deepfakes/faceswap](https://github.com/deepfakes/faceswap) by [Virtaitech](http://virtai.tech), wich support faceswap to run on our product **`orion`**. 
>>    - you will need to use `orion.executor` to start the code with orion support.
>>    - 
## 5 Need-to-Knows about `faceswap-orion`
### 5.1 Disable functionality of keyboard interaction
- since `faceswap-orion` is running with [MPI](https://www.open-mpi.org/), which has limited support for keyboard interaction during when we running multi-GPU. we disabled the keyboard interaction function provide by [deepfakes/faceswap](https://github.com/deepfakes/faceswap), which means you can't press 'S' or 'ENTER' to interact.
- set a smaller '-ss' value
### 5.2 Feature of *speed-monotor*
- we add *`speed-monitor`* to monitor the training speed, which will be print as INFO everytime when save-model is triggered.
- *`speed-monitor`* algorithm:  
    ||||
    |-|-|-|
    | img/sec = | img/iter * iter/sec =| img/batch * batch/iter * iter/sec  
    | img/batch = | batch_size  
    | batch/iter = | **`1`** |( actually **`3`** in faceswap )
### 5.3 Multi-Logs
- running orion-modified version `faceswap-orion` will allow all threads (for each GPU) be able to print log. that's why you will see a lot of logging info.
- So that, you will see one single log print `X` times, where X is the number of GPU(s) you use. 
- watch the front of some logging info, 
    ```
    Orion-faceswap.executor Activated
    [1,0]<stdout>:Setting Faceswap backend to NVIDIA
    [1,1]<stdout>:Setting Faceswap backend to NVIDIA
    [1,2]<stdout>:Setting Faceswap backend to NVIDIA
    [1,3]<stdout>:Setting Faceswap backend to NVIDIA
    ...
    ```
    here `[1,X]`, `X` is the GPU thread id, and you will know which log is comming from witch thread.
## 6 Contact
||email|
|-|-|
|Join Us| career@virtaitech.com |
|Business Corparation| business@virtaitech.com |
|Feedback|feedback@virtaitech.com |
|Tech Support| wanghuize@virtaitech.com |
![Scan Me](scan.jpg)
