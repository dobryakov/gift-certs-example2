# Mermaid диаграммы (Mermaid CLI)

```sh
docker run --rm -u $(id -u):$(id -g) \
  -v ./specs:/data \
  minlag/mermaid-cli \
  -i /data/<diagram>.mmd \
  -o /data/<diagram>.png \
  -s 2
```

- Используется для генерации PNG-диаграмм (`architecture.png`, `erd-gift-certificates.png`, `sequence-purchase.png`, `flowchart-return.png`) из соответствующих файлов `.mmd`.
- Флаг `-s 2` увеличивает масштаб и обеспечивает высокое разрешение; при необходимости отдачи под печать можно поднять до `-s 3`.
- Важно запускать из корня репозитория; замените `<diagram>` на имя исходного файла (без расширения).
