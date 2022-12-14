KIND_NAME ?= ansible
SELDON_CORE_DIR ?= ${HOME}/work/seldon-core

LATEST_TAG ?= latest

IMAGE_NAME ?= rafalskolasinski/mock-model
IMAGE_VERSION ?= ${LATEST_TAG}

BASE_IMAGE_NAME ?= seldonio/seldon-core-s2i-python38
BASE_IMAGE_VERSION ?= 1.14.1

BASE_IMAGE ?= ${BASE_IMAGE_NAME}:${BASE_IMAGE_VERSION}


ISTIO_PORT ?= 8003
HTTP_SERVICE_PORT ?= 9000
GRPC_SERVICE_PORT ?= 5000
METRICS_SERVICE_PORT ?= 6000

SELDON_LOG_LEVEL ?= INFO
SELDON_DEBUG ?= false

WORKERS ?= 1


FLAGS = SELDON_LOG_LEVEL=${SELDON_LOG_LEVEL}
FLAGS += PREDICTIVE_UNIT_HTTP_SERVICE_PORT=${HTTP_SERVICE_PORT}
FLAGS += PREDICTIVE_UNIT_GRPC_SERVICE_PORT=${GRPC_SERVICE_PORT}
FLAGS += PREDICTIVE_UNIT_METRICS_SERVICE_PORT=${METRICS_SERVICE_PORT}
FLAGS += INCLUDE_METRICS_IN_CLIENT_RESPONSE=false
FLAGS += SELDON_DEBUG=${SELDON_DEBUG}


PAYLOAD ?= '{"meta": {"tags": {"some": "tag"}}, "data": {"names": ["a", "b", "c"], "ndarray": [[1, 2, 3]]}}'


############################ LOCAL TESTING ###############################

serve:
	${FLAGS} seldon-core-microservice Model --service-type MODEL --workers ${WORKERS} --threads 5

local-rest:
	curl -v -s -H 'Content-Type: application/json' \
		-d ${PAYLOAD} \
		http://localhost:${HTTP_SERVICE_PORT}/api/v1.0/predictions | jq .

local-grpc:
	cd ${SELDON_CORE_DIR}/executor/proto && grpcurl -v \
		-d ${PAYLOAD} \
		-plaintext -proto ./prediction.proto  0.0.0.0:${GRPC_SERVICE_PORT} seldon.protos.Seldon/Predict

local-metrics:
	curl -s localhost:${METRICS_SERVICE_PORT}/metrics


##################### IMAGE BUILD / PUSH  / LOAD ###########################

image:
	docker build --build-arg BASE_IMAGE=${BASE_IMAGE} -t ${IMAGE_NAME}:${LATEST_TAG} .

push: image
	docker tag ${IMAGE_NAME}:${LATEST_TAG} ${IMAGE_NAME}:${IMAGE_VERSION}
	docker push ${IMAGE_NAME}:${IMAGE_VERSION}

kind: image
	kind load docker-image ${IMAGE_NAME}:${IMAGE_VERSION} --name ${KIND_NAME}

docker-serve:
	docker rm -f mock-model || echo "Already removed"
	docker run --rm -it --name "mock-model" \
		-p 127.0.0.1:${HTTP_SERVICE_PORT}:9000 \
		-p 127.0.0.1:${GRPC_SERVICE_PORT}:5000 \
		${IMAGE_NAME}:${LATEST_TAG}


############################## K8s TESTING #################################

deploy:
	kubectl apply -f manifest.yaml

remove:
	kubectl delete -f manifest.yaml

rest:
	curl -v -s -H 'Content-Type: application/json' \
		-d ${PAYLOAD} \
		http://localhost:${ISTIO_PORT}/seldon/seldon/seldon-mock-model/api/v1.0/predictions  | jq .


grpc:
	cd ${SELDON_CORE_DIR}/executor/proto && grpcurl -v \
		-d ${PAYLOAD} \
		-rpc-header seldon:seldon-mock-model -rpc-header namespace:seldon \
		-plaintext -proto ./prediction.proto \
		0.0.0.0:8003 seldon.protos.Seldon/Predict

metrics:
	curl -v -s http://localhost:${ISTIO_PORT}/seldon/seldon/seldon-mock-model/prometheus
