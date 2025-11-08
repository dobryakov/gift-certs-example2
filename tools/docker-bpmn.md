# BPMN диаграммы (auto-layout + PNG)

```sh
# собрать образ (однократно)
docker build -f tools/Dockerfile.bpmn-renderer -t local/bpmn-renderer .

# сконвертировать диаграмму в PNG
docker run --rm \
  -v "$PWD":/workspace \
  -w /workspace \
  local/bpmn-renderer \
  specs/bpmn/<diagram>.bpmn:specs/bpmn/<diagram>.png
```
