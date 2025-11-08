## Mermaid диаграммы (Mermaid CLI)

```sh
docker run --rm -u $(id -u):$(id -g) \
  -v ./specs:/data \
  minlag/mermaid-cli \
  -i /data/<diagram>.mmd \
  -o /data/<diagram>.png
```

- Используется для генерации PNG-диаграмм (`architecture.png`, `erd.png`, `sequences.png`, `flow.png`) из соответствующих файлов `.mmd`.
- Важно запускать из корня репозитория; замените `<diagram>` на имя исходного файла.
