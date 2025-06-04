FROM python:3.11-slim

# 기본 패키지 설치
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*

# JupyterLab 설치
RUN pip install --upgrade pip && \
    pip install jupyterlab

# Jupyter 실행용 포트
EXPOSE 8888

# 기본 작업 디렉토리
WORKDIR /workspace

# JupyterLab 실행 커맨드
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--port=8888", "--no-browser", "--allow-root"]
