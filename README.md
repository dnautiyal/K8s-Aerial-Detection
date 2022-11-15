# FlaskMLops

- docker build -t aerial-detection-webapp .
- docker run -e AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id) -e AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key) -p 8080:8080 aerial-detection-webapp
- docker build -t aerial-detection-inference .
- docker run -e AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id) -e AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key) -p 8080:8080 aerial-detection-inference

## to run from command line 
- uvicorn inference_service:app --reload


## to stop the container
docker stop <container_name>

## doing it thru Docker-Compose
- git pull https://github.com/dnautiyal/FlaskMLops.git
- docker-compose -f docker-compose.yaml up --build
- docker-compose down 



# +++++++++++++++++++++++++++++++++++++++++++++++++++++

## TensorRT engine
- conda activate pytorch
- git clone https://github.com/schwenkd/aerial-detection-mlops.git
- cd aerial-detection-mlops
- cd yolov7
- aws s3 cp s3://aerial-detection-mlops4/model/Visdrone/Yolov7/20221026/exp11/weights/best.pt ae-yolov7-best.pt
### install onnx-simplifier not listed in general yolov7 requirements.txt
- pip3 install onnx-simplifier
### Pytorch Yolov7 -> ONNX with grid, EfficientNMS plugin and dynamic batch size
- python export.py --weights ./ae-yolov7-best.pt --grid --end2end --dynamic-batch --simplify --topk-all 100 --iou-thres 0.65 --conf-thres 0.35 --img-size 960 960
### ONNX -> TensorRT with trtexec and docker ... for CUDA version 11.6 you need nvcr.io/nvidia/tensorrt:22.02-py3
- docker run -it --rm --gpus=all nvcr.io/nvidia/tensorrt:22.02-py3
### Copy onnx -> container:
- docker cp ae-yolov7-best.onnx <container-id>:/workspace/
### Export with FP16 precision, min batch 1, opt batch 8 and max batch 8
- ./tensorrt/bin/trtexec --onnx=ae-yolov7-best.onnx --minShapes=images:1x3x960x960 --optShapes=images:8x3x960x960 --maxShapes=images:8x3x960x960 --fp16 --workspace=4096 --saveEngine=ae-yolov7-best-fp16-1x8x8.engine --timingCacheFile=timing.cache
### Test engine
- ./tensorrt/bin/trtexec --loadEngine=ae-yolov7-best-fp16-1x8x8.engine
### Copy engine -> host:
- docker cp <container-id>:/workspace/ae-yolov7-best-fp16-1x8x8.engine .
### copy everthing to s3
- aws s3 cp ae-yolov7-best.pt s3://aerial-detection-mlops4/model/Visdrone/Yolov7/best/ae-yolov7-best.pt
- aws s3 cp ae-yolov7-best.onnx s3://aerial-detection-mlops4/model/Visdrone/Yolov7/best/ae-yolov7-best.onnx
- aws s3 cp ae-yolov7-best-fp16-1x8x8.engine s3://aerial-detection-mlops4/model/Visdrone/Yolov7/best/ae-yolov7-best-fp16-1x8x8.engine

### +++++++++++++++++++++++++++++++++++++++++++++++++++++++
## Build and Configure Triton Model Repository
### Create folder structure
- mkdir -p triton-deploy/models/yolov7/1/
- aws s3 cp s3://aerial-detection-mlops4/model/Visdrone/Yolov7/best/ae-yolov7-best-fp16-1x8x8.engine .
- touch triton-deploy/models/yolov7/config.pbtxt
- vim triton-deploy/models/yolov7/config.pbtxt
## write following content to the file
### name: "yolov7-visdrone-finetuned"
### platform: "tensorrt_plan"
### max_batch_size: 8
### dynamic_batching { }
#
### Place model
- mv ae-yolov7-best-fp16-1x8x8.engine triton-deploy/models/yolov7/1/model.plan
- aws s3 sync triton-deploy s3://aerial-detection-mlops4/model/Visdrone/Yolov7/triton-deploy

