# Этап 1: Сборка Coq-окружения (без Python venv)
# Этот этап готовит только зависимости Coq и OCaml
FROM coqorg/coq:8.20 AS builder

USER root

# Версии для Coq/OCaml
ARG OCAML_SWITCH=4.14.1
ARG DUNE_VERSION=3.15.3
ARG SERAPI_VERSION=8.20.0+0.20.0

# Устанавливаем только build-essential, так как Python нам здесь не нужен
RUN apt-get update && apt-get install -y --no-install-recommends build-essential git curl ca-certificates

# Инициализируем opam и создаём switch
RUN opam init --disable-sandboxing --yes && \
    (opam switch create ${OCAML_SWITCH} --yes || opam switch create 4.14.0 --yes)

SHELL ["/bin/bash", "-lc"]

# Устанавливаем Coq/OCaml зависимости
RUN eval $(opam env) && \
    opam update -y && \
    opam install --yes dune.${DUNE_VERSION} coq-serapi.${SERAPI_VERSION}

# Создаём скрипт активации opam
RUN mkdir -p /etc/profile.d && \
    echo 'eval $(opam env)' > /etc/profile.d/opam.sh


# Этап 2: Создание финального образа с CUDA и всеми зависимостями
# Базируемся на образе от NVIDIA с CUDA 12.1
FROM nvidia/cuda:12.1.1-devel-ubuntu22.04

# Версии для Python-пакетов
ARG PIP_VERSION=24.0
ARG SETUPTOOLS_VERSION=69.5.1
ARG WHEEL_VERSION=0.43.0
ARG SEXPDATA_VERSION=1.0.2
ARG PYTORCH_VERSION=2.4.0
ARG TORCHVISION_VERSION=0.19.0
ARG TORCHAUDIO_VERSION=2.4.0
ARG JUPYTERLAB_VERSION=4.2.1

# Устанавливаем системные зависимости в Ubuntu
# libgmp-dev нужен для работы бинарных файлов OCaml
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
      python3.10-venv python3-pip git build-essential ca-certificates curl unzip libgmp-dev \
 && rm -rf /var/lib/apt/lists/*

# Копируем готовые артефакты Coq/OCaml из builder-а
COPY --from=builder /usr/local /usr/local
COPY --from=builder /root/.opam /root/.opam
COPY --from=builder /etc/profile.d/opam.sh /etc/profile.d/opam.sh

# Создаём venv НАПРЯМУЮ в финальном образе
ENV VENV_PATH=/opt/venv
RUN python3 -m venv $VENV_PATH

# Устанавливаем переменные окружения, включая venv в PATH
ENV PATH="$VENV_PATH/bin:/root/.opam/default/bin:$PATH"

# Устанавливаем ВСЕ Python-пакеты в один слой, используя --extra-index-url
RUN python3 -m pip install --no-cache-dir --upgrade \
    pip==${PIP_VERSION} \
    setuptools==${SETUPTOOLS_VERSION} \
    wheel==${WHEEL_VERSION} && \
    python3 -m pip install --no-cache-dir \
    sexpdata==${SEXPDATA_VERSION} \
    jupyterlab==${JUPYTERLAB_VERSION} \
    torch==${PYTORCH_VERSION} \
    torchvision==${TORCHVISION_VERSION} \
    torchaudio==${TORCHAUDIO_VERSION} \
    --extra-index-url https://download.pytorch.org/whl/cu121

# Создаём символическую ссылку на sertop
RUN ln -s /root/.opam/default/bin/sertop /usr/local/bin/sertop

WORKDIR /workspace
EXPOSE 8888

# Запускаем JupyterLab
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--no-browser", "--allow-root", "--NotebookApp.token=''"]