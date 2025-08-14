# 🐳 Docker 설정 가이드 (GCP Ubuntu + NVIDIA L4 GPU)

이 문서는 GCP 인스턴스(Ubuntu 22.04 LTS)에 NVIDIA L4 GPU가 있는 환경에서 NVIDIA 드라이버, Docker, NVIDIA Container Toolkit을 설치하는 전체 절차를 설명합니다.

---

## 1️⃣ NVIDIA 드라이버 설치
#### GCP의 Ubuntu 이미지에 내장된 자동 설치 기능을 사용하는 것이 가장 안정적이고 쉽습니다.
~~여기서는 535로 해놓긴 했는데 난 L4 기준 550 깔리긴 했음.<br/>보통 터미널에서 드라이버 목록 확인 후 권장 드라이버 자동 설치하는데 Gemini나 GPT한테 물어볼 것을 강력추천함~~

```bash
# 패키지 목록 업데이트
sudo apt update

# 권장 드라이버 자동 설치
sudo apt install ubuntu-drivers-common
sudo ubuntu-drivers autoinstall

# 변경사항 적용을 위한 재부팅 (필수)
sudo reboot
```

재부팅 후:

```bash
nvidia-smi
```

---

## 2️⃣ Docker 설치

```bash
# Docker 공식 GPG 키 설정
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Docker 저장소 추가
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Docker Engine 설치
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

sudo 없이 Docker 사용하도록 설정 (권장):
```bash
sudo usermod -aG docker $USER
```
  > Note: 이 설정을 적용하려면 SSH 접속을 완전히 끊었다가 다시 연결해야 합니다.


재로그인 후:

```bash
docker --version
```

---

## 3️⃣ NVIDIA Container Toolkit 설치 (nvidia-docker2)

```bash
# NVIDIA GPG 키 및 저장소 설정
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
&& curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# 툴킷 설치
sudo apt update
sudo apt install -y nvidia-container-toolkit

# 툴킷 설정을 적용하기 위해 Docker 재시작
sudo systemctl restart docker
```

### ✅ 최종 확인
모든 설치가 완료된 후, 아래 명령어를 실행하여 GPU를 사용하는 Docker 컨테이너가 정상적으로 작동하는지 최종 확인합니다.

```bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```
nvidia-smi 결과가 컨테이너 내부에서 정상적으로 출력되면, 모든 설정이 완벽하게 완료된 것입니다.


---

## 4️⃣ Dockerfile 예시 (PyTorch + CUDA)
> 굳이 안 해도 되긴 함

```dockerfile
FROM nvcr.io/nvidia/cuda:12.1.0-cudnn8-devel-ubuntu20.04

ENV DEBIAN_FRONTEND=noninteractive
ENV PATH="/opt/conda/bin:$PATH"

RUN apt-get update && apt-get install -y \
    git wget curl build-essential && rm -rf /var/lib/apt/lists/*

RUN wget https://repo.anaconda.com/miniconda/Miniconda3-py311_23.11.0-2-Linux-x86_64.sh -O miniconda.sh && \
    bash miniconda.sh -b -p /opt/conda && rm miniconda.sh
RUN /opt/conda/bin/conda install -y -c conda-forge mamba=1.4.2

RUN pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121

WORKDIR /workspace
RUN git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git
WORKDIR /workspace/YOUR_REPO

COPY requirements.txt .
RUN pip install -r requirements.txt
CMD ["python", "main.py"]
```

---

## 5️⃣ 실행 스크립트 예시 (run.sh)

```bash
#!/bin/bash
IMAGE="your-repo-image"
REPO="YOUR_REPO"

docker build -t $IMAGE .
docker run --rm -it --gpus all \
  -v $(pwd)/$REPO:/workspace/$REPO \
  -w /workspace/$REPO \
  $IMAGE
```

---

## 6️⃣ 실행 시 자주 쓰는 명령어

* 컨테이너 내부 진입: `docker run --rm -it --gpus all image bash`
* 실시간 nvidia-smi 확인: `watch -n 1 nvidia-smi`
* 마운트 경로 주의: GCP 내 경로는 대부분 `/home/ubuntu/...`

> 더 많은 명령어는 `DOCKER-COMMANDS.md`를 참조하세요.
