# Dockerfile для pycoq (рекомендованный)
FROM coqorg/coq:8.13

USER root

# базовые утилиты и dev-зависимости
RUN apt-get update \
 && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
      python3 python3-venv python3-pip git build-essential ca-certificates m4 pkg-config curl unzip \
 && rm -rf /var/lib/apt/lists/*

# создаём системный venv (удобно для jupyter + pip-пакетов)
ENV VENV=/opt/venv
RUN python3 -m venv $VENV
ENV PATH="$VENV/bin:$PATH"

# pip: не ломаем системный python, но обновим pip в venv
RUN $VENV/bin/python -m pip install --no-cache-dir --upgrade pip setuptools wheel

# opam: инициализация без интерактивности; отключаем sandbox (в контейнере часто требуется)
RUN opam init --disable-sandboxing --yes || true

SHELL ["/bin/bash", "-lc"]

# Создаём switch OCaml (подберите версию при проблемах)
# если 4.14.1 недоступна в образе, замените на совместимую (проверить opam list --all-versions)
RUN opam switch create 4.14.1 --yes || opam switch create 4.14.0 --yes || true

# Загружаем окружение opam в текущий shell и ставим нужные пакеты
RUN eval $(opam env) \
 && opam update -y \
 && opam install --yes dune \
 && eval $(opam env) \
 && opam install --yes coq-serapi pythonlib ppx_python pyml || \
    (echo "opam install failed, try pinning versions or check network" && false)

# Создаём скрипт для автоматического загрузки opam env в логин-shells/container
RUN mkdir -p /etc/profile.d \
 && echo 'eval $(opam env)' > /etc/profile.d/opam.sh

# Убедимся, что sertop доступен в PATH (создаём symlink к стандартному месту)
RUN eval $(opam env) && ln -sf "$(opam var bin)/sertop" /usr/local/bin/sertop || true

# Клонируем pycoq и собираем
WORKDIR /opt
RUN git clone --recursive https://github.com/ejgallego/pycoq.git || (echo "git clone failed" && false)
WORKDIR /opt/pycoq

# Сборка pycoq из исходников. make install использует opam окружение.
# RUN eval $(opam env) \
#  && make install \
#  && dune build examples/test.py

# Устанавливаем Jupyter (опционально)
RUN $VENV/bin/python -m pip install --no-cache-dir jupyterlab

# Рабочая директория
WORKDIR /workspace
EXPOSE 8888

# Подстраховка: добавим opam bin в PATH runtime
ENV PATH="/root/.opam/default/bin:/root/.opam/4.14.1/bin:$PATH"

CMD ["jupyter","lab","--ip=0.0.0.0","--no-browser","--allow-root","--NotebookApp.token=''"]