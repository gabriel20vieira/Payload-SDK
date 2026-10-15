# CLAUDE.md

Este ficheiro dá contexto ao Claude Code (e a qualquer pessoa nova no projeto) sobre o repositório.

## Objetivo

Correr modelos **YOLO custom** em inferência no edge, no **DJI Manifold 3**, montado num drone DJI, usando o **DJI Payload SDK (PSDK)** para obter vídeo e telemetria e devolver resultados ao piloto ou à aplicação.

A prioridade é a **qualidade da deteção** em condições reais de voo (objetos pequenos, altitude, blur de movimento, luz variável). A latência tem de chegar para tempo real, mas não se troca precisão por FPS sem medir primeiro.

## Estado atual

- O repositório está vazio: sem commits, e o `HEAD` aponta para um branch inválido. O primeiro commit deve ir para `main`.
- A estrutura de diretórios abaixo é a **proposta**. Atualiza este ficheiro quando ela mudar.

## Hardware alvo: DJI Manifold 3

Confirmado na página oficial da DJI (https://enterprise.dji.com/manifold-3/specs):

- Até 100 TOPS (INT8, sparse: é o número de melhor caso, não o real), 16 GB LPDDR5, 256 GB SSD.
- 33 W nominais, IP55, -20 °C a 50 °C.
- Aeronaves suportadas: Matrice 400, Matrice 4 Series, Matrice 4D Series.
- Ligação por E-Port V2 (no M400 usar E1–E3, de preferência E3) e USB-C 3.2 Gen 1.

Ainda **por confirmar no dispositivo** (os resellers não concordam):

- O módulo NVIDIA exato (Jetson Orin NX ou AGX Orin). Verifica com `cat /etc/nv_tegra_release` e `jtop`.
- As versões de JetPack, TensorRT, CUDA e cuDNN, e a versão do PSDK suportada.
- Se há DLA disponível e quantos núcleos.

Quando confirmares estes valores, escreve-os aqui. Não assumas versões: os engines TensorRT dependem da versão exata.

## Arquitetura (pipeline)

```
Câmara/gimbal DJI ──PSDK liveview (H.264/H.265)──► decode por HW (NVDEC / GStreamer nvv4l2decoder)
        │                                              │
        └─ telemetria PSDK (atitude, altitude, GPS,    ▼
           zoom, ângulo do gimbal)              pré-processamento GPU (letterbox, normalização)
                                                       │
                                                       ▼
                                                TensorRT engine (YOLO custom)
                                                       │
                                                       ▼
                                   pós-processamento (decode, NMS) ► tracking ► geolocalização
                                                       │
                                                       ▼
                                   saída: widgets PSDK / transmissão de dados / log / overlay
```

Regras da pipeline:

- **Sem cópias CPU↔GPU desnecessárias.** O decode, o pré-processamento e a inferência devem ficar em memória GPU/NVMM sempre que possível.
- **Captura, inferência e saída correm em threads separadas**, com filas limitadas e política de descartar frames antigos. Nunca acumular latência.
- **Cada frame leva timestamp e a telemetria correspondente.** A geolocalização e o tracking dependem desta sincronização.
- **Os callbacks do PSDK não podem bloquear.** Copiar os dados e devolver o controlo logo.

## Estrutura proposta

```
training/        # treino e avaliação no PC/servidor (Python, Ultralytics)
  configs/       # data.yaml, hiperparâmetros
  scripts/       # train, val, export ONNX
export/          # ONNX versionados + metadata (classes, imgsz, versão do modelo)
edge/            # aplicação no Manifold 3 (C++17, CMake)
  src/
    psdk/        # init PSDK, liveview, telemetria, widgets
    video/       # decode por HW
    infer/       # wrapper TensorRT, pré/pós-processamento CUDA
    track/       # tracker (ByteTrack ou equivalente)
  tools/         # build de engines, benchmark, avaliação no dispositivo
third_party/     # PSDK e outras dependências fixadas por versão
datasets/        # NÃO versionado (ver .gitignore); só manifestos/splits
docs/
```

## Fluxo do modelo

1. **Treino** fora do drone (Ultralytics YOLO). Guardar dataset, commit, `imgsz` e a seed de cada run.
2. **Exportar para ONNX** com `imgsz` fixo e opset compatível com o TensorRT do dispositivo. Por defeito sem NMS no grafo, a menos que se confirme que o plugin do TensorRT o suporta.
3. **Construir o engine TensorRT no próprio Manifold 3.** Os engines não são portáveis entre GPUs nem entre versões do TensorRT. Nunca versionar `.engine`; versionar o ONNX e o comando de build.
4. **Validar no dispositivo:** correr o mesmo split de validação com o engine e comparar o mAP com o do PC. Uma queda maior que ~1 ponto de mAP50-95 é um bug (pré-processamento diferente, calibração má) até prova em contrário.

## Como ganhar qualidade (alavancas, por ordem de impacto típico)

1. **Dados do domínio:** imagens aéreas reais da mesma câmara, altitudes, ângulos de gimbal e condições do uso final. Hard negatives. É isto que mais pesa.
2. **Resolução de entrada:** objetos pequenos vistos do ar precisam de `imgsz` maior (960–1280), ou de **tiling** (estilo SAHI) sobre o frame em resolução nativa, juntando depois as deteções com NMS. Medir o custo em FPS.
3. **Paridade de pré-processamento:** o letterbox, o padding (114), a ordem RGB/BGR e a normalização têm de ser bit-a-bit iguais aos do treino. Há um teste que compara as saídas Python e C++ para a mesma imagem.
4. **Precisão numérica:** FP16 é o padrão seguro. INT8 só com calibração feita sobre imagens representativas (≥500, do domínio) e com mAP validado no dispositivo. Se o INT8 perder qualidade, voltar a FP16 ou usar precisão mista em camadas sensíveis.
5. **Tamanho do modelo:** preferir um modelo maior (s/m) a FP16 em vez de um nano, desde que caiba no orçamento de latência.
6. **Coerência temporal:** um tracker estabiliza as deteções e reduz falsos positivos de frame único. Filtrar por persistência ou confiança acumulada.
7. **Uso da telemetria:** a altitude e o zoom dão a escala esperada do objeto, o que permite filtrar caixas impossíveis e escolher o tile ou o zoom certo.
8. **Thresholds:** afinar `conf` e `iou` por classe com base na curva PR do split de validação, não a olho.

Qualquer alteração que afete a qualidade tem de vir acompanhada de números: mAP no dispositivo, FPS e latência ponta-a-ponta (do frame recebido ao resultado enviado).

## Orçamento de desempenho

- Medir com `jtop` / `tegrastats`: utilização GPU/DLA, temperatura, potência.
- Fixar o modo de energia (`nvpmodel`) e documentar qual se usa. Os benchmarks só são comparáveis no mesmo modo.
- Vigiar o throttling térmico com o drone parado no chão (sem fluxo de ar) em comparação com o voo.
- Objetivo inicial (ajustar): ≥15 FPS no stream principal, latência ponta-a-ponta < 150 ms.

## Convenções de código

- **Edge:** C++17, CMake, RAII para recursos CUDA/TensorRT/PSDK. Sem `new`/`delete` soltos. Erros do PSDK (`T_DjiReturnCode`) são sempre verificados e registados.
- **Treino/tools:** Python 3, type hints, scripts reprodutíveis por linha de comando (sem notebooks como fonte de verdade).
- A configuração (caminhos de modelos, thresholds, classes, tiling) fica em ficheiros YAML/JSON, não hardcoded.
- Os logs têm timestamps monotónicos e incluem o ID do modelo carregado.

## Segurança e credenciais

- As credenciais da app DJI (App ID, App Key, App License, Developer Account) **nunca vão para o git**. Ficam num ficheiro local ignorado (ex.: `edge/config/dji_app_info.local.h`) com um template `*.example` versionado.
- Não versionar datasets, pesos `.pt` grandes nem engines. Usar armazenamento externo ou Git LFS e referenciar por hash.
- Código que mexe em controlo de voo ou no gimbal exige revisão explícita. Por defeito a aplicação é **só de perceção** e não comanda a aeronave.

## Comandos

_Por preencher quando existir código._ Secções esperadas:

- Treino: `python training/scripts/train.py --cfg ...`
- Export: `python training/scripts/export_onnx.py --weights ... --imgsz ...`
- Build do engine (no Manifold 3): `trtexec --onnx=... --saveEngine=... --fp16`
- Build do edge: `cmake -S edge -B build && cmake --build build -j`
- Avaliação no dispositivo: `edge/tools/eval_engine ...`

## Referências

- DJI Payload SDK: https://developer.dji.com/doc/payload-sdk-tutorial/en/
- Especificações do Manifold 3: https://enterprise.dji.com/manifold-3/specs
- Ultralytics (export/TensorRT): https://docs.ultralytics.com/
- NVIDIA TensorRT: https://docs.nvidia.com/deeplearning/tensorrt/
