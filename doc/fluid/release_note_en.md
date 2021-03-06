 # Release Notes

##  Important Updates
This version focuses on enhancement of the framework functions, includes improving the inference deployment capability, releasing PLSC for super-large-scale classification training task, and optimizing the parameter server mode. In addition, the compilation options, compilation dependence and code library are fully cleaned up and optimized. The model library is optimized by adjusting the structure and adding dynamic graph models. The development kits and utility components are upgraded.

### Training Framework
 - Adds AMP (Automatic Mixed Precision) interfaces and control flow interfaces.
 - Optimizes the tensor using method and GPU memory allocation strategy.
 - Supports Nvidia DALI GPU data preprocessing library.
 - Optimizes the functions and performance of basic Ops
 - Enhances the functions of dynamic graph models, including performance improvement and supporting new APIs which can converts the data independent dynamic graph model into static graph model.
 - Improves the user experience of debug functions.

### Inference Deployment
- Paddle Serving
      - Optimizes the Python API.
      - Supports new programming languages API, such as R and Go.
      - Enhanced the quantitative capability.
- Paddle Lite
      - Supports deploying the model generated by the post-training quantization method without calibration data.
      - Enhanced the OpenCL capability.
      - Supports Kunlun XPU.
- Paddle Slim
      - Optimizes the pruning, quantization, distillation and NAS (Network Architecture Search) API for adapting the model library.
      - Supports large-scale knowledge distillation framework called Pantheon.

### Distributed Training
- Unified the implementation mode of the semi-asynchronous, fully asynchronous and GEO modes in parameter server mode. The back-end is unified into communicator. The front-end interface is unified into fleet. Select different mode by configuring the fleet strategy.
- Releases the PLSC for super-large-scale classification training task.

### Model Construction
- Releases the text-so-speech model library called Parakeet, including several leading-edge text-to-speech algorithms.
- Adds 14 image classification pre-training models in PaddleCV, for enriching the 3D and tracking direction models.
- Supports Jieba word segmentation in PaddleNLP.
- Adds a multi-task model called MMoE in PaddleRec.
- Adds more dynamic graph models.
- Adjusts and optimizes the structure of model library.

### Development Kits
- Optimizes the PaddleDetection and PaddleSeg by adding a large number of models as well as pre-training models, enhancing the training speed and accuracy of typical models, and strengthens the model compression and deployment capabilities.
- Releases the recommended sorting system called ElasticRec, can be deployed via K8S and support streaming training and online forecast services.

### Utility Components
- Adds 52 pre-training models to enrich the models up to 100+, as well as improves the function experience.
- Upgrades the kernel of PALM, opens API, and supports more task types.
- Adds an open dataset in PaddleFL (Federated learning framework).
- Upgrades the versions of PARL (Deep reinforcement learning framework) and PGL (Graph learning framework) . Opens more algorithm and supports more functions.

## Training Framework
- API
    - Adds AMP (Automatic Mixed Precision) APIs, which can convert a network training mode into mixed accuracy mode in a general way, and ensuring the accuracy fluctuation within the normal range.
    - Adds control flow OPs, such as while_loop, cond, case and switch_case. It is recommended to use the new APIs for much easier to use. The following functions are supported:
        - Supports using python callable as the control condition or executive objects.
        - Supports using different losses or optimizers in different branches of the control flow.
        - Supports using CPU data or GPU data in condition of the control flow
    - Supports using the variable lists as parameters for some APIs, while these APIs only supported  string lists as the `parameter_list` or `no_grad_set`. Do not need to obtain the `name` attribute of variables when using the following APIs:
        - fluid.backward.append_backward(loss, parameter_list=None, no_grad_set=None, callbacks=None)
        - fluid.backward.gradients(targets, inputs, target_gradients=None, no_grad_set=None)
        - The minimize methods of optimizers, such as Adam's minimize: minimize(loss, startup_program=None, parameter_list=None, no_grad_set=None, grad_clip=None)
- Basic Functions Optimization
    - Supports configuring tensor data with numpy float16 data types, and no need to convert to unit16 type first.
    - Supports using minus sign to express the tensor's opposite.
    - GPU memory Allocation Strategy:
        - Changes the default policy to `AutoGrowth`. In this policy, the GPU memory is applied on demand when not affecting the training speed. While it`s difficult to start another task on the same GPU in the GPU memory pre-allocation strategy before. This change can avoid this problem.
        - Adjusts the GPU memory allocation for multi-card tasks: Set the GPU memory allocators on different GPU cards to the `Lazy` initialization mode. If a card is not used, the GPU memory will not be applied for this card. While the GPU memory OOM problem could be caused by running tasks on idle GPU cards without setting CUDA_VISIBLE_DEVICES, when GPU memory is occupied on other GPU cards. This change can avoid this problem.
    - OP Function Upgrade
        - elu: This activation function supports the calculation of second-order gradients.
        - Prroi_pool: The parameter `rois` supports the `Tensor` or `LoDTensor` type.
        - Conv2d, pool2d, batch_norm, lrn: supports using the MKL-DNN library to perform gradient calculation of these OPs.
        - argsort: Supports descending. A new parameter `descending` is added, default value is `False`.
- Basic Performance Optimization
    - DALI Preprocessing Acceleration
        - Supports the Nvidia DALI GPU data preprocessing library, which can be used to accelerate the preprocessing speed of data such as images, videos, and speeches.
    - Automatic Mixed Precision Training Optimization
        - Implements the following optimization strategies to increase the training throughput of the ResNet50 model, along with the DALI data preprocessing module. The mixed accuracy training throughput of a single V100 card is increased from 600+ images/s to 1,000+ images/s. The throughput of 8 cards for a single machine is increased to 7,840 image/s. The throughput of 32 cards for 4 machines is increased to 28,594 images/s.
            - Supports NHWC data layout inputs for some OPs such as batch_norm, conv2d. Accelerates fp16 calculation speed by using Tensor Core technology.
            - Fusing some op patterns in the model, such as batch_norm and relu, based on the IR Pass mechanism.
            - Optimizes kernel of some elementwise OPs, such as add, mul.
    - Optimize the `RecomputeOptimizer` to enable bigger batchsize. The batchsize of Bert-large model increases by 533.62% while using the `RecomputeOptimizer`.
    - OP Performance Optimization
        - Implements the fusion operator called `fuse_emb_seq_pool` of `embedding` and `sequence_pool`. Optimizes the `murmurhash3_x64_128` in `bloom_filter`. These optimization increases the training speed of some NLP models.
        - Optimizes the GPU performance of `mean op`. When a data of 32 *32 *8 *8 tensor is input, the forward calculation speed is increased by 2.7 times.
        - Optimizes OPs of `assign` and `lod_reset`, to avoid nnecessary GPU memory copy and data transform.
        - Optimizes the kernel implementation of stack OP. The performance of a single card of GPU in the XLnet/Ernie model is improved by 4.1%.
- Dynamic Graph
    - Function Optimization
        - Removes the `name_scope` parameter in `Layers` to make it easier to inherit and call.
        - Removes the `block` parameter in the `to_variable` to simplify the use of the API.
        - Removes the `build_once` as for the the problem that model parameters depend on data. So that `Layers` can get all the parameter tables when implementing the `init` execution. It`s convenient for saving and loading, parameter initialization, parameter debugging, and parameter optimization.
        - Optimizes the automatic pruning function facilitate user networking and reduce the reverse calculation amount.
        - Supports `SelectedRows` operation so that the Embedding layer supports sparse update of a single card.
        - Adds functions such as ParameterList, LayerList, and Sequencial, as for the problem that the framework lacks containers. It`s more convenient for networking with these functions.
        - Supports functions such as named_sublayers and named_parameters to facilitate programming.
        - Supports the `Linear lr warmup decay` strategy.
    - Performance Optimization
        - Optimizes the interaction of python with c++, GradMaker, OperatorBase, and allocator. The performance is improved by 270% for the LSTM-based language model task on the P40 machine.
        - Removes the redundant codes for performance problems caused by calling dead codes of `optimized_guard`. The performance of optimizers such as SGD and Adam is improved by 5% to 8% for or the Transformer model (batch_size=64) on the P40 machine.
        - Optimizes asynchronous DataLoader of the dynamic graph. For the Mnist, ResNet and other CV models , the single card training speed is improved by more than 40% on the P40 machine.
        - Adds numpy bridge function, to support sharing the underlying data between Tensor and ndarray in CPU mode. This can avoid the copy problem of numpy input when creating variables, and improve efficiency.
        - Optimizes the GPU memory by the forward variable space strategy, which can delete the Tensor Buffer not required in reverse calculation in advance. The maximum batch size is increased by more than 20%-30% in some models such as ResNet.
         - To reduce the performance impact caused by adding extra `scale_op` to update the beta parameter in `AdamOptimizer`.  Iintegrate the updating logic of `beta` into `adam_op` to reduce the cost of calling op kernel.  The performance  of  is improved by 9.67%  on the P40 machine.
    - Dynamic Graph Deployment
        - Supports the `TracedLayer` interface to convert the dynamic graph model into the static graph.
- Debugging Analysis
    - Optimizes the error message. Classifies the framework error messages and optimizes the message descriptions for more convenient to solve the problem according to the messages.
    - Optimizes the performance analysis profile function.
        - Enhances the functions and accuracy of the profile. Supports profile options at different levels. The call relation of events can be recorded in the profile data and printed.
    - Optimizes the checking and debugging functions of `nan inf` which is enabled through `FLAGS_check_nan_inf`. The performance, function, and output information are all greatly improved.
        - In terms of speed, the v100 test ResNet50 model has a performance improvement of about 1000 times compared with the original utility components, and maintains an over 80% efficiency for normal training.
        - In terms of function, the support for fp16 is added and environment variables can be set to skip the inspection of op, op_role, and op_var to facilitate the debugging of the fp16 model.
        - The output information is detailed and accurate. Besides wrong op and tensor names, the quantity of wrong nan, inf, and normal numerical values are printed to facilitate debugging.
- Releases the lightweight installation package `paddlepaddle-tiny` for CPU training and forecast, supporting installed on Windows/Linux/Mac OS and python27/python35/python36/python37.
    - Supports the following compile functions: no avx, no ml, no gpu, no unittest.
    - Remove the slim and some dataset.
    - Reduce the Linux package size from 90M to 37M. Reduce the Windows package size from50.8 M to 9.6M. Reduce the MAC package size from 59M to 19.8M.
    - Reduce the number of installation requirement dependencies from 15 to 7.

## Inference Deployment
- Server-side Inference Library
    - Python API
        - Supports reading and writing model from the memory to meet the model encryption requirements.
        - The Scale operator is no longer added at the end of the inference model.
        - Adds ZeroCopy API, which is basically the same as the C++ APIs. Supports using numpy.ndarray as the input and output. It`s convenient for Python scenario.
        - Adds several interfaces in AnalysisConfig to completely cover the C++ inference functions, including removing pass and disabling inference glog.
    - Support for Other Programming Languages
        - Add inference API of R and Go, and the related usage methods and examples are added.
    - Provides the corresponding header file of ProtoBuf to facilitate users to analyzing structure of models.
    - For a inference library with TRT compilation, the TensorRT library is not provided from thrid_party any more and needs to be downloaded by users at https://developer.nvidia.com/tensorrt.
    - Functional Enhancement:
        - Supports access Paddle Lite by submap mode, and ResNet50 has been verified.
        - Supports the MKL-DNN FC INT8 kernel.
        - Supports Ernie model in Paddle-TensorRT. For the Ernie model (seq length = 128) on the T4 card, the delay of fp16 inference is 3.6 ms, which is faster than the fp32 inference by 37%.
        - Quantization: the single-threaded performance  and the multi-threaded performance  are improved by 2.79 times and 1.79 times for ERNIE INT8 on the second-generation Xeon scalable platform 6271 respectively, while the  Ernie INT8 model  has only slight decline precision compared with the FP32 model.
- Mobile/Embedded End-side [Paddle Lite](https://github.com/PaddlePaddle/Paddle-Lite)
    - Releases the version v2.3.
    - Upgrades the functions of Model_optimize_tool.
    - Supports "The post-training quantization method without calibration data". The model storage space can be reduced by 2 to 4 times.  
    - OpenCL: The migration of 30 Image2D Kernels are finished and 14 Ops are covered.
    - Strenthens the capability with FPGA, NPU. Supports Kunlun XPU for inference.
    - Releases a new official website document. Adds the document of "post-training quantization method without calibration data"  
- [Paddle Serving](https://github.com/PaddlePaddle/Serving):
    - Releases the forecast service of remote text vector representation of the bert-type semantic understanding model.
    - Release the paddle-gpu-serving WHL package. Supports pip installation and Python codes.
    - Supports 13 semantic understanding models in Paddlehub. Supports the single-machine multi-card mode. The forecast speed is 869.56 samples per second using the Ernie_tiny model, when the average sample length is 7 under a single P4 GPU.
- [PaddleSlim](https://github.com/PaddlePaddle/PaddleSlim):
    - Moves PaddleSlim to independent repo.
    - Refactors pruning, quantization, distillation and NAS API. Provide more low-level APIs for developer.
        - Quantization:
            - Adds post training quantization strategy based on KL divergence. Supports quantization of the embedding layer.
            - Supports quantization for  MKL-DNN-FC layer based on QAT.
            - Adds post training quantization that support 30 kinds of operators. Supports spartial operators to skip quantization.
            - Supports skipping some operators in training aware strategy
        - Pruning: Refactors and enhances the code of pruning to support more kinds of networks.
        - NAS:
            - Supports NAS based on simulated annealing. Provides more predefined search spaces and support custom search spaces.
            - Adds one-shot algorithm for NAS. The speed of search is 20 times faster than that of the previous version.
    - Releases the large-scale scalable knowledge distillation framework called Pantheon.
        - Achieves full decoupling between student and teacher models and among teacher models. They can independently run on different physical devices respectively to make full use of computing resources.
        -  Supports the multi-device large-scale inference of the teacher model in the single node. The acceleration ratio is tested to be linear on BERT-like complex models.
        - Supports knowledge transmission between teacher and student models running on any two physical devices in the same Internet environment. By using TCP/IP protocol for communication in online distillation mode.
        - Unifies API interfaces for online and offline distillation modes, enabling different teacher models operating in different distillation modes.
        - The merging of knowledge and the batch reorganization of knowledge data are completed automatically on the student side to facilitate the knowledge fusion of the multi-teacher models.
    - Model Zoo:
        - Releases benchmark of image classification model such as ResNet50, MobileNet.
        - Adapts PaddleDetection library and release benchmark of YOLOv3 models with different backbone.
        - Adapts PaddleSeg library and release benchmark of Deepabv3+ models with  different backbone.
    - Refines Document:
        - Refines documents of API. Adds some QuickStart tutorials and advanced tutorials. Adds model zoo docu which contain models for image classification, object detection,  semantic segmentation. Translates all documents to English.

## Distributed
- Parameter Server Mode:
    - Reduces the memory usage greatly during training. On 100 million embedding training tasks, the Trainer-side memory can be reduced by 90%.
    - Reduces the memory usage of distributed saving and loading models greatly. The Pserver-side memory peak value can be minimized to $1/N $ of the original value, where N is the number of Pserver nodes.
    - Optimizes the dense parameter communication in GEO mode.
    - Supports distributed AUC index calculation.
    - Adds distributed barrier functions.
    - Adds Semi-asynchronous modes in Communicator.
    - Supports semi-asynchronous modes of the `TrainFromDataset` training interface.
    - Adds `DistributedStrategy` in `Fleet` to improve the convenient usage. Integrates the current distributed related flags.
    - Supports single-program multi-loss training modes in `Fleet pslib` to optimize the training performance.
    - Supports k8s environment in 100 billion sparse mode.
- [Large-scale classification library PLSC](https://github.com/PaddlePaddle/PLSC): It supports the large-scale classification problem that data parallel cannot solve due to the limitation of video memory capacity.
    - Supports three built-in models such as ResNet50, ResNet101, and ResNet152. Supports User-defined models. Under the single-machine eight-V100 GPU configuration environment, the ResNet50 model has a million-class training speed of 2,122.56 images/s, which is 1.3 times faster than that of the standard ResNet50 model.
    - Releases a `plsc-serving whl` package for model online forecast service. It can forecast the image semantic vector representation of the face recognition model. Supports making a forecast using a user-trained model. The forecast speed of the ResNet50 model (batch size=256) is 523.47 images/s under a single V100 GPU.
    - Releases the pre-training models based on the ResNet50 network and the MS1M-ArcFace dataset: https://plsc.bj.bcebos.com/pretrained_model/resnet50_distarcface_ms1mv2.tar.gz.
- Releases the benchmark for ResNet50 mixed precision training (single-card, multi-card, and multi-machine)

## Basic Model Library
- [Models repo github](https://github.com/PaddlePaddle/models)
- PaddleNLP
    - Seq2seq supports training modes such as RL and GAN in the static-graph of Paddle.
    - A training model for word segmentation and part-of-speech tagging is released.  With the knowledge distillation framework Pantheon, the F1 score of this model  on the own dataset is improved 1% over that of PaddleNLP LAC.  This model is merged into the jieba repo, with adding a flag use_paddle to enable deep learning model mode. In addition, the paddle version detection and rollback mechanism is added in jieba to ensure user experience.
    - Adds dynamic graph model implementations for these models: word2vec, senta, transformer, Bert, seq2seq, and LAC.
- PaddleSpeech
  - Releases text-to-speech toolkit Parakeet (Paddle PARAllel text-to-speech toolkit).
        - Implements the standard workflow for data preprocessing, training, and synthesis of the TTS models.
        - Provides the out-of-the-box pre-processing implementation of typical datasets.
        - Provides the commonly-used model components in the TTS field to  facilitate the model implementation.
        - Reseases the TTS models DeepVoice3, ClarinNet, TransformerTTS, FastSpeech, WaveNet, and WaveFlow.
- PaddleCV
    - Image Classification:
        - Adds 14 pre-training models including SENet-vd, Res2Net, and HRNet series of models:
        - Supports accelerating data preprocessing by using DALI. On the ImageNet training, 1.5 times (ResNet50) to more than 3 times (ShuffleNet) the acceleration is obtained and the GPU utilization is greatly improved.
    - 3D Vision:
        - Releases PointNet++, PointRCNN models.  
    - Tracking Model Library:
         - Releases SiamFC and ATOM models,
    - Add dynamic graph model implementations for the following models: MobileNet-v1/v2, YOLOv3, FasterRCNN, MaskRCNN, video classification TSM model, and video motion positioning BMN model.
- PaddleRec
    - Releases a multi-task model called MMoE for the recommended field. It can be applied to large-scale multi-task joint training in the industrial circles.
    - Adds dynamic graph model implementations for the following models: gru4rec, deepfm.

## End-To-End Development Kits
- [PaddleDetection](https://github.com/PaddlePaddle/PaddleDetection)
    - The precision of the YOLOv3 model is further improved. The precision for the COCO data reaches 43.2%, an absolute increase of 1.4% compared with the previous version.
    - Add the following model implementations and pre-training models:
        - Add the best single model CascadeCARCNN-FPN-Dcnv2-Nonlocal ResNet200-vd in the Google AI Open Images 2019-Object Detction competition is added.  Releases a pre-training model of this algorithm based on Objects365 data.
        - Add a series of CBResNet, Res2Net, and HRNet pre-training models.
      - Adds a LibraRCNN algorithm and the pre-training models.
      - Add GIoU, DIoU, and CIoU loss-based pre-training models in the FasterRCNN R50 FPN model. Without reducing the inference speed, the precision for the COCO data is improved by 1.1%, 0.9%, and 1.3% respectively.
  - Adds Modules:
      - Backbone network: CBResNet, Res2Net, and HRNet are added.
      - Loss modules: GIoU loss, DIoU loss, and CIoU loss are added. Libra loss and YOLOv3 loss support a fine-grained op combination.
      - Postprocessing modules: The softnms and DIOU nms modules are added.
      - Regular module: A DropBlock module is added.
  - Functional Optimization and Improvement:
      - YOLOv3 data preprocessing is accelerated. The overall training speeds up by 40%.
      - The data preprocessing logic is optimized.
      - The benchmark data for face detection inference is added.
      - Inference examples under the Paddle inference library Python API are added.
  - Detection Model Compression:
      - Pruning: A MobileNet-YOLOv3 uningtailoring solution and model are released, with FLOPs - 69.6%, mAP + 1.4% for the VOC dataset, and FLOPS - 28.8%, mAP + 0.9% for the COCO dataset. A ResNet50vd-dcn-YOLOv3 pruning solution and model are released, with FLOPs - 18.4%, mAP + 0.8% for the COCO dataset.
      - Distillation: A MobileNet-YOLOv3 distillation solution and model are released, with mAP + 2.8% for the VOC data and mAP + 2.1% for the COCO data.
      - Quantization: YOLOv3 and BlazeFace quantitative models are released.
      - Pruning + Distillation: A MobileNet-YOLOv3 pruning + distillation solution and model are released, with FLOPS - 69.6%, inference speedup 64.5% under the GPU, mAP - 0.3 % for the COCO dataset. A ResNet50vd-dcn-YOLOv3 pruning + distillation solution and model are released, with FLOPS - 43.7%, inference speedup 24.0% under the GPU, mAP + 0.6 % based on the COCO data.
      - Search: A complete search solution for the open source BalzeFace-nas.
  - Inference Deployment:
      - The support of the Paddle inference library for TensorRT and FP16 precision is adapted.  
  - Documents:
      - Adds the documents for introducing the data preprocessing module and a document for implementing the user-defined data Readers.
      - Adds the documents about how to add an algorithm model.
      - Documents are deployed to the website: https://paddledetection.readthedocs.io/zh/latest/
- [PaddleSeg](https://github.com/PaddlePaddle/PaddleSeg)
    - Adds Models
        - LaneNet model applicable to lane segmentation scenarios.
        - Lightweight Fast-SCNN model applicable to high performance scenarios.
      - HRNet semantic segmentation model applicable to high-precision scenarios.
  - Releases multiple PaddleSlim-based model compression solutions:
      - Fast-SCNN tailoring solution and model on Cityscapes dataset.
      - Deeplabv3p-Xception and Deeplabv3p-MobilenetV2 distillation solutions on Cityscapes dataset.
      - Deeplabv3p-MobilenetV2 search solution on Cityscapes dataset.
      - Deeplabv3p-Mobilenet quantitative solution and model on Cityscapes dataset.  
  - Enhance the deployment capability
      - Adds the lightweight deployment of Python.
      - The TensorRT acceleration support for FP16 and Int8 quantitative models is added.
      - Adds the tutorials for human portraits segmentation Paddle-Lite mobile deployment of DeepLabv3p-MobileNetV2
      - Optimizes the Model exportation step. Supports GPU implementation of image preprocessing and post processing. The performance is improved by 10%-20%.
      - Provides the benchmark for the prediction performance of U-Net, ICNet, PSPNet, DeepLabv3+, and other models for images of different sizes to facilitate users to select models based on performance.  
  - Experience Optimization
      - Adds a learning rate function called warmup. Supports using with different learning rate decay strategies to improve fine-tuning stability.
      - Adds the function of automatically saving an optimal mIoU model.
      - The document logic is comprehensively optimized. An AIStudio practical tutorial on industrial scenarios such as industrial quality inspection and fundus screening is provided.
     - Marked imaged can be saved in pseudo-color image format to improve their preview experience.
- [ElasticRec](https://github.com/PaddlePaddle/ElasticRec)
    - An ElasticRec recommended sorting system is released. It is deployed through K8S. Streaming training and online inference service are supported.

## Utility Components
- [PaddleHub](https://github.com/PaddlePaddle/PaddleHub)
    - 52 new pre-trained models are added. Currently, the total number of pre-training models is 100+:
        - Semantic models: Five semantic models such as RoBERTa_wwm, BERT_wwm, and ERNIE-Tiny are added.
        - Text classification: Three anti-porned models are added.
        - Image classification: A total of 36 image classification models such as ResNext-WSL and EfficientNet are added.
        - Object detection: Five detection models such as pedestrian detection and vehicle detection are added.
        - Key point detection: Two models for key point detection of face and body posture are added.
        - Face mask detection: Two PyramidBox-Lite-based face mask detection models are added.  
        - Universal face detection: Four universal Face detection models such as Ultra Light Fast Generic Face Detector and PyramidBox-Lite are added.
    - Function:
        - Bert Service, a text vector representation service based on Paddle Serving is added.
        - Task flexibility is enhanced. An hook mechanism supports the loading of user-defined codes is added.  
        - Code results are optimized. The command line execution speed is increased by 50%.
        - Dataset and Reader are refactored, The quantity of adaptive user-defined dataset codes is reduced by 60%.  
        - The AutoFinetune interface is optimized. Multi-experiment visualization effect display is supportsed.
    - Experience Optimization
        - The logic is fully optimized. Rich AIStudio tutorial contents are added.
        - The landing page of the official website has been fully upgraded to provide the function of quick online experience and tutorial guidance.
- Multi-task learning framework [PALM](https://github.com/PaddlePaddle/PALM)
  - Python3 and Windows are supported.
  - Release APIs and the multi-task learning kernel are upgraded.
    - Support independent task saver.
  - Continuous training and inference are supported, Dataset files can be switched over freely under a single execution.  
  - Supports model customization.
  - The multi-task learning kernel is refactored and fix some bugs.
- Upgrade multi-task learning ability.
  - Support independent settings of batch size and sequence length across tasks.
  - Fix inconsistent problem of the tasks on GPUs.
  - The multi-task learning scheduling and termination strategies are optimized to generally improve the model generalization ability.
- Upgrade the ability and types of pre-defined tasks.
  - Upgrade matching task. Add pairwise learning and multiple categories support.
  - The support for machine reading comprehension tasks is enhanced. User controllable preprocessing hyper-parameters are added.
  - The support for sequence labeling tasks is added.
- The large-scale training/inference capability is strengthened.
  - Add automatic multi-gpus inference.
  - Refactor asynchronous reader. Support dynamic padding length for multi-task learning running on multiple-gpus.
- A module for the management and downloading of pre-training models is added.
  - The management and downloading of pre-training models such as BERT, ERNIE, and RoBERTa are supported.
  - A RoBERTa Chinese pre-training model is added Releases the version v1.3.
- Federated Learning [PaddleFL](https://github.com/PaddlePaddle/PaddleFL):
    - According to the added components, the original samples are modified in example and the femnist_demo and submitter_demo examples are added
    - Fl_distribute_transpiler is optimized to add the support of FedAvg strategy for the adam optimizer.
    - SecAgg strategy (Secure Aggregation) is added to achieve secure parameter aggregation.
    - The scheduler and submitter functions are added: The scheduler is used to control whether the trainer participates in update during training. The submitter is used to complete the function of submitting paddleFL tasks in the MPI clus
    - A LEAF dataset federated learning open dataset is added. An API is added to set a benchmark. Classical datasets in the image classification, emotion analysis, character inference, and other fields , such as MNIST and Sentiment140, are supported.
- Deep Reinforcement Learning Framework [PARL](https://github.com/PaddlePaddle/PARL)
  - Version v1.3 is released.
  - The support for the Multi-Agent RL algorithm including MADDPG is added.
  - The support for multi-card training is added. An example of a multi-card DQN algorithm is released.
  - SOTA algorithms TD3 and SAC in the open source continuous control field.
  - Implementation and training solution for the open source NeurIPS2019 reforcement learning challenge champion model. Trained models are open (Consideration can be given to open class)
- Paddle Graph Learning Framework [PGL](https://github.com/PaddlePaddle/PGL)
  -  Version v1.1 is released:
    - The support for the authoritative graph learning database OGB is added. Three types of tasks including nodepropered, linkpred, and graphpropered are fully supported. A SOTA baseline is released.C  Decouples the forecast library from third_party. Refactors 28 third-party-dependent compilation codes to facilitate the unified management of external dependencies.
    - A graph solution PGL-Rec and a knowledge graph embedding algorithm set PGL-KE are released.
    - An improvement on ease of use is made. A high-order API of PGL is released.  
    - Other upgrade points: Sampling of a multi-process graph is optimized and a GraphSAGE kind of models is accelerated by three times. Lod Tensor-based Graph Batch and Graph Pooling operators are added. Models including distributed heterogeneous task graph algorithm, GraphZoom, and PinSage are added for Model Zoo.

## Code Reconstruction and Upgrade
- Compilation
    - A compilation thus improving the code quality.
      C  Fixes the codes corresponding to the warnings of -Wno-error=sign-compare (at a total of more than 100 points). An error will be reported for all subsequent warnings of this kind during compilation, option WITH_NCCL is added. Single-card users can display and specify WITH_NCCL=OFF to accelerate compilation.
    - A compilation option WITH_TP_CACHE is added to cache third-party source codes to avoid repeated downloading. Windows users can set it to ON to speed up compilation and improve compilation stability.
    - The `CUDA_ARCH_NAME` default value is set to `Auto` (`All` indicates compiling all GPU architectures and `Auto` indicates compiling only the current machine GPU architecture). For developers, a lot of compilation time is saved using `Auto` than using `All`, thus improving development efficiency.
    - Redundant links and products and needless file copying are reduced, thus speeding up the compilation in Windows.
- External Dependency Library
    - MKL-DNN is upgraded to the latest Version 1.1.
    - The inference library is decoupled from `third_party` and 28 third-party-dependent compilation codes are refactored to facilitate the unified management of external dependencies.
    - Two third-party-dependent private code repository, one unnecessary external dependency, and 2000+ lines of unnecessary codes under the patch are removed to improve the code repository quality.
- Code Cleanup, Refactoring, and Optimization
    - The unnecessary `contrib/float16` directory is removed. The unnecessary snappy/snappystream dependency under the BRPC is deleted.
    - `loss.py` and `sequence_lod.py` are split out of `python/paddle/fluid/layers/nn.py` according to the API functions, thus reducing the code quantity of `nn.py` and facilitating reading.
    - The codes corresponding to the warnings of `-Wno-error=sign-compare` (at a total of more than 100 points) are fixed. An error will be reported for all subsequent warnings of this kind during compilation, thus improving the code quality.
    - `WarningLnk4006/WarningLnk4221` (There are about 300) compiled by Windows MSVC is removed to improve the code repository quality.
    - The quantity of reduce_op, expand_op, and expand_as_op templates is reduced to accelerate GPU compilation and reduce whl package space by 70 M.
    - The pybind function of every OP is automatically generated under the dynamic graph using codes and directly called in layers to improve the dynamic graph performance and reduce the coupling degree with the static graph.

## Bug Fixes
- Fix the problem of MKL-DNN error when PaddleDetection-based Faster-RCNN uses the Python API to make a inference.
- Fix the problem of training suspension in the GPU implementation of sum op because some Tensors are not initialized.
- Fix the problem of precision loss when the value in fill_constant is set to a large integer.
- Fix the problem of precision inconsistency of softmax_with_cross_entropy_op with regard to the CUDA.
- Fix the problem that when a clone program is fixed, the stop_gradient attribute in the program can not be copied to a new program.
- Fix the problem of precision loss of elementwise_pow op with regard to integers.
- Fixed the problem that some GFLAGSs cannot perform specifying outside the inference library.
- Fix the problem of random inference core caused by some passes in Analysistor multithreading. (fc_gru_fuse_pass, seqconv_eltadd_relu_fuse_pass, attention_lstm_fuse_pass, embedding_fc_lstm_fuse_pass, fc_lstm_fuse_pass, seq_concat_fc_fuse_pass)
- Fix the error that specifying a GPU in the same process using AnalysisConfig does not take effect after NativePredictor is used to specify the use of CPU inference.
- Fix the bug of compilation error in the case of -DWITH_MKL=OFF on Windows.
- Fix the bug that tuple (Variable) cannot be input into the py_func OP; Add an code example of how to write Python OP.
- Fix the problem of the sigmoid cudnn kernel being called as the tanh cudnn kernel by mistake.
- Fix some bugs related to reshape and Conv2D depthwisecoin dynamic graph mode; fix the problem of some parameters in the network having no gradient, causing the bug of program crash.
- Fix the bug of running error of GradientClip in parameter server mode.
- Fix the problem of memory leak in full asynchronous mode  of the parameter server.
