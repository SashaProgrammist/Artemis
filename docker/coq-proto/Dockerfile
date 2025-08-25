# Этап 1: Сборка и настройка окружения
# Используем официальный образ Coq 8.20, который основан на Debian 12 "Bookworm"
FROM coqorg/coq:8.20 AS builder

# Переключаемся на пользователя root для установки системных пакетов
USER root

# Определяем версии зависимостей, где это надёжно работает (pip, opam)
ARG OCAML_SWITCH=4.14.1
ARG DUNE_VERSION=3.15.3
ARG SERAPI_VERSION=8.20.0+0.20.0
ARG PIP_VERSION=24.0
ARG SETUPTOOLS_VERSION=69.5.1
ARG WHEEL_VERSION=0.43.0
ARG JUPYTERLAB_VERSION=4.2.1

# Устанавливаем базовые утилиты
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
      python3 python3-venv python3-pip git build-essential ca-certificates m4 pkg-config curl unzip \
 && rm -rf /var/lib/apt/lists/*

# Создаём системный venv для изоляции Python-пакетов
ENV VENV_PATH=/opt/venv
RUN python3 -m venv $VENV_PATH
ENV PATH="$VENV_PATH/bin:$PATH"

# Обновляем pip и базовые пакеты ВНУТРИ venv
RUN $VENV_PATH/bin/python3 -m pip install --no-cache-dir --upgrade \
    pip==${PIP_VERSION} \
    setuptools==${SETUPTOOLS_VERSION} \
    wheel==${WHEEL_VERSION}

# Инициализируем opam и создаём switch для OCaml
RUN opam init --disable-sandboxing --yes && \
    (opam switch create ${OCAML_SWITCH} --yes || opam switch create 4.14.0 --yes)

# Используем SHELL с окружением opam для последующих команд
SHELL ["/bin/bash", "-lc"]

# Устанавливаем необходимые opam-пакеты, используя синтаксис `пакет.версия`
RUN eval $(opam env) && \
    opam update -y && \
    opam install --yes dune.${DUNE_VERSION} coq-serapi.${SERAPI_VERSION}

# Создаём скрипт для активации окружения opam
RUN mkdir -p /etc/profile.d && \
    echo 'eval $(opam env)' > /etc/profile.d/opam.sh

# Устанавливаем JupyterLab ВНУТРИ venv
RUN $VENV_PATH/bin/python3 -m pip install --no-cache-dir jupyterlab==${JUPYTERLAB_VERSION}


# Этап 2: Создание чистого финального образа
FROM coqorg/coq:8.20

# Временно переключаемся на root для операций с системными файлами
USER root

# Копируем системные пакеты, установленные на предыдущем этапе
COPY --from=builder /usr /usr
COPY --from=builder /lib /lib
COPY --from=builder /lib64 /lib64

# Копируем настроенное окружение opam, Python venv и скрипт активации
COPY --from=builder /root/.opam /root/.opam
COPY --from=builder /opt/venv /opt/venv
COPY --from=builder /etc/profile.d/opam.sh /etc/profile.d/opam.sh

# Устанавливаем переменные окружения для Python и opam
ENV VENV_PATH=/opt/venv
ENV PATH="$VENV_PATH/bin:/root/.opam/default/bin:$PATH"

# Создаём символическую ссылку на sertop от имени root
RUN ln -s /root/.opam/default/bin/sertop /usr/local/bin/sertop

# Возвращаемся к непривилегированному пользователю `coq` для безопасности
USER coq

# Устанавливаем рабочую директорию
WORKDIR /workspace

# Открываем порт для JupyterLab
EXPOSE 8888

# Команда по умолчанию для запуска JupyterLab
# `--allow-root` здесь фактически не нужен, так как мы запускаемся от `coq`,
# но не мешает работе.
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--no-browser", "--allow-root", "--NotebookApp.token=''"]