# Docker Command Reference

## Mermaid диаграммы (Mermaid CLI)

```sh
docker run --rm -u $(id -u):$(id -g) \
  -v /home/ubuntu/gift-certs-example2/specs:/data \
  minlag/mermaid-cli \
  -i /data/<diagram>.mmd \
  -o /data/<diagram>.png
```

- Используется для генерации PNG-диаграмм (`architecture.png`, `erd.png`, `sequences.png`, `flow.png`) из соответствующих файлов `.mmd`.
- Важно запускать из корня репозитория; замените `<diagram>` на имя исходного файла.

## BPMN диаграммы (Node + bpmn-to-image)

```sh
docker run --rm \
  -e HOST_UID=$(id -u) \
  -e HOST_GID=$(id -g) \
  -v /home/ubuntu/gift-certs-example2:/workspace \
  -w /workspace \
  node:20 bash -lc "apt-get update >/dev/null && \
    apt-get install -y chromium libnss3 libatk1.0-0 libatk-bridge2.0-0 libcups2 libdrm2 libxkbcommon0 libxcomposite1 libxdamage1 libxfixes3 libxrandr2 libgbm1 libasound2 libpangocairo-1.0-0 libpango-1.0-0 >/dev/null && \
    echo -e '#!/bin/bash\nexec /usr/bin/chromium --no-sandbox "\$@"' > /usr/local/bin/chromium-no-sandbox && \
    chmod +x /usr/local/bin/chromium-no-sandbox && \
    su node -c 'cd /workspace && PUPPETEER_EXECUTABLE_PATH=/usr/local/bin/chromium-no-sandbox npx --yes bpmn-to-image@0.9.0 specs/bpmn/<diagram>.bpmn:specs/bpmn/<diagram>.png' && \
    chown $HOST_UID:$HOST_GID specs/bpmn/<diagram>.png"
```

- Позволяет конвертировать BPMN-файлы в PNG (`issue_activation.png`, `redeem_void.png`).
- Перед запуском убедитесь, что файл BPMN находится в `specs/bpmn` и замените `<diagram>` на нужное имя.
- Команда устанавливает зависимости Chromium внутри контейнера и использует `--no-sandbox`, чтобы избежать ограничений при запуске Puppeteer.
