# Build the manager binary
FROM golang:1.25 AS builder

WORKDIR /workspace

ARG TARGETPLATFORM
ARG TARGETARCH

ARG APISERVER_NETWORK_PROXY_VERSION=0.30.2
ARG KUBECTL_VERSION=v1.30.2
ARG ADDON_AGENT_IMAGE_NAME

# Build Apiserver-network-proxy binaries
RUN wget https://github.com/kubernetes-sigs/apiserver-network-proxy/archive/refs/tags/v${APISERVER_NETWORK_PROXY_VERSION}.tar.gz \
      && tar xzvf v${APISERVER_NETWORK_PROXY_VERSION}.tar.gz \
      && cd apiserver-network-proxy-${APISERVER_NETWORK_PROXY_VERSION} \
      && CGO_ENABLED=0 go build -o /workspace/proxy-server ./cmd/server/ \
      && CGO_ENABLED=0 go build -o /workspace/proxy-agent ./cmd/agent/ \
      && CGO_ENABLED=0 go build -o /workspace/proxy-test-client ./cmd/test-client/ \
      && CGO_ENABLED=0 go build -o /workspace/proxy-test-server ./cmd/test-server/ \
      && cd /workspace \
      && curl -LO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/${TARGETARCH}/kubectl" \
      && chmod a+x kubectl

COPY . .

# Build addons
RUN CGO_ENABLED=0 go build -a -o agent cmd/addon-agent/main.go
RUN CGO_ENABLED=0 go build -a \
        -ldflags="-X 'open-cluster-management.io/cluster-proxy/pkg/config.AgentImageName=${ADDON_AGENT_IMAGE_NAME}'" \
        -o manager cmd/addon-manager/main.go
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -a -o cluster-proxy cmd/cluster-proxy/main.go

# Use distroless as minimal base image to package the manager binary
# Refer to https://github.com/GoogleContainerTools/distroless for more details
FROM alpine:3.21

WORKDIR /
RUN apk add libc6-compat
COPY --from=builder /workspace/kubectl /workspace/proxy-server /workspace/proxy-agent /workspace/proxy-test-client /workspace/proxy-test-server ./
COPY --from=builder /workspace/agent /workspace/manager ./
COPY --from=builder /workspace/cluster-proxy ./
USER 65532:65532
