# Mapara do Maroto — Visualizador Multi-Câmera

> **🚧 WIP (Work in Progress) — Projeto em desenvolvimento.**
>
> Funcionalidades, interfaces e documentação podem mudar. A implementação atual é um protótipo de visualização e integração, com limitações descritas neste documento.

O **Mapara do Maroto** é uma aplicação para visualizar posições de pessoas e esqueletos humanos sobre o mapa de um ambiente. Um servidor Python recebe anotações de detecção por AMQP, processa essas informações e disponibiliza o estado da cena por uma API HTTP. No navegador, uma interface em JavaScript desenha o mapa e as pessoas com perspectiva e permite navegar pela cena.

A aplicação suporta duas formas de representar pessoas:

- **Projeção 2D → chão:** transforma um ponto de referência da imagem de cada câmera em uma posição no plano do chão, usando a calibração da câmera.
- **Esqueletos 3D:** recebe articulações já reconstruídas por um serviço externo e desenha suas conexões no espaço. Esse modo pode coexistir com a projeção 2D de outras câmeras.

Este repositório contém o visualizador e os arquivos locais de mapa e calibração. Os serviços de captura, detecção de pessoas e reconstrução de esqueletos 3D são dependências externas.

## Sumário

- [Objetivo e escopo](#objetivo-e-escopo)
- [Arquitetura e fluxo de dados](#arquitetura-e-fluxo-de-dados)
- [Fundamentação teórica](#fundamentação-teórica)
- [Implementação técnica](#implementação-técnica)
- [Estrutura do projeto](#estrutura-do-projeto)
- [Requisitos e instalação](#requisitos-e-instalação)
- [Execução e configuração](#execução-e-configuração)
- [Uso da interface](#uso-da-interface)
- [API HTTP](#api-http)
- [Adaptação para outro ambiente](#adaptação-para-outro-ambiente)
- [Diagnóstico de problemas](#diagnóstico-de-problemas)
- [Limitações e evolução](#limitações-e-evolução)
- [Possíveis aplicações](#possíveis-aplicações)

## Objetivo e escopo

O objetivo é reunir informações de diferentes câmeras em um referencial espacial comum, permitindo observar a distribuição de pessoas no ambiente e inspecionar resultados de um sistema de visão computacional.

A versão atual oferece:

- Leitura de paredes a partir de um arquivo NumPy `.npz`.
- Leitura de calibrações individuais de câmera.
- Consumo de mensagens `ObjectAnnotations` por tópicos AMQP.
- Projeção de detecções 2D no plano mundial `Z = 0`.
- Exibição opcional de esqueletos 3D recebidos pelo broker.
- API com o último estado disponível e expiração de dados antigos.
- Visualização interativa com rotação, deslocamento e zoom.

A atualização depende da chegada das mensagens e das consultas periódicas do navegador. Não há garantia de latência nem de sincronização entre as câmeras.

## Arquitetura e fluxo de dados

```mermaid
flowchart LR
    D["Serviços externos de detecção 2D"] --> B["Broker AMQP"]
    S["Serviço externo de reconstrução 3D"] --> B
    C["Calibrações .npz"] --> P["Servidor Python / FastAPI"]
    M["Mapa .npz"] --> P
    B -->|ObjectAnnotations| P
    P --> E["Último estado em memória"]
    E --> A["GET /api/state"]
    A -->|"Consulta a cada 100 ms"| W["JavaScript / Canvas 2D"]
```

1. O programa carrega o mapa e as calibrações exigidas pelo modo escolhido.
2. O FastAPI inicia uma thread para consumir mensagens do broker.
3. Detecções 2D são convertidas em pontos no chão; anotações 3D são convertidas em listas de articulações.
4. O último resultado de cada câmera e do tópico 3D é armazenado em memória.
5. A API monta uma resposta com o mapa e os dados ainda válidos.
6. O navegador consulta essa resposta e redesenha a cena continuamente.

O mapa é carregado diretamente do disco. Apesar do título `Map.BA1` usado na interface, o visualizador não publica nem assina um tópico de mapa chamado `Map.BA1`.

## Fundamentação teórica

### Referenciais e calibração

O processamento utiliza três referenciais:

| Referencial | Coordenadas | Uso |
| --- | --- | --- |
| Imagem | `(u, v)`, em pixels | Posição dos pontos detectados na imagem. |
| Câmera | `(Xc, Yc, Zc)` | Direção de observação e geometria da câmera. |
| Mundo | `(X, Y, Z)` | Referencial compartilhado pelo mapa e pelas pessoas. |

No modelo de câmera pinhole, a matriz intrínseca `K` relaciona coordenadas da câmera com pixels. A matriz extrínseca `rt = [R | t]` representa, na convenção esperada pelo código, a transformação do mundo para a câmera:

$$
\mathbf{p}_C = R_{CW}\mathbf{p}_W + \mathbf{t}_{CW}
$$

O programa amplia `rt` para uma matriz homogênea `4 × 4` e calcula sua inversa:

$$
T_{WC} = T_{CW}^{-1}
= \begin{bmatrix}R_{WC} & \mathbf{t}_{WC} \\ 0 & 1\end{bmatrix}
$$

Assim, `R_WC` e `t_WC` permitem transformar uma direção ou posição da câmera para o mundo. Todas as calibrações, o mapa e os esqueletos externos precisam usar a mesma origem, orientação dos eixos e escala. O visualizador não converte unidades automaticamente.

### Escolha do ponto de referência da pessoa

Uma detecção de pessoa ocupa uma região da imagem, mas a projeção no chão precisa de um ponto que aproxime sua posição de apoio. A função `reference_pixel()` combina:

- **Coordenada horizontal `u`:** média das coordenadas horizontais dos keypoints com IDs `12` e `13`, tratados pelo código como quadris.
- **Coordenada vertical `v`:** maior coordenada vertical dos keypoints com IDs `16` e `17`, tratados como tornozelos.

Como a coordenada vertical da imagem cresce para baixo, o maior `v` corresponde ao tornozelo mais baixo na imagem. Esse ponto é uma aproximação: sua coordenada horizontal e vertical podem vir de articulações diferentes.

Na ausência dessas articulações, o código usa o centro horizontal e a base da caixa delimitadora. A região deve conter pelo menos dois vértices; objetos sem essa região são ignorados, mesmo que possuam keypoints. O produtor deve fornecer os vértices na convenção esperada: primeiro canto superior esquerdo, segundo canto inferior direito.

### Correção de distorção e interseção com o chão

A função `pixel_to_ground()` corrige a distorção do pixel com `cv2.undistortPoints()`, usando `K`, os coeficientes de distorção e a matriz intrínseca corrigida `nK`.

Se o pixel corrigido é $\tilde{\mathbf{p}} = [\tilde{u},\tilde{v},1]^T$, a direção do raio no referencial da câmera é:

$$
\mathbf{r}_C = nK^{-1}\tilde{\mathbf{p}}
$$

Esse raio, expresso no mundo, pode ser parametrizado por:

$$
\mathbf{p}_W(\lambda)
= \mathbf{t}_{WC} + \lambda R_{WC}\mathbf{r}_C
$$

A hipótese geométrica central é que a posição da pessoa pertence ao chão plano `Z = 0`. Impondo essa condição:

$$
\lambda
= -\frac{t_{WC,z}}{(R_{WC}\mathbf{r}_C)_z}
$$

A posição resultante é retornada como `(X, Y, 0)`. Raios com denominador próximo de zero e resultados não finitos são descartados.

Essa projeção não recupera o corpo em 3D. Ela estima uma posição no chão a partir de uma imagem e de uma superfície conhecida. Erros de calibração, oclusão dos pés, detecções imprecisas ou pisos fora do plano adotado afetam a estimativa.

### Esqueletos 3D e visualização em perspectiva

No modo 3D, a reconstrução já ocorreu em um sistema externo, identificado nos comentários como `skeleton_tracker_main`. O visualizador recebe os keypoints com coordenadas mundiais `(X, Y, Z)` e desenha articulações e conexões entre elas. Ele não realiza triangulação nem associação entre vistas.

O navegador utiliza o contexto **2D** do Canvas para desenhar uma cena com perspectiva. Uma câmera virtual orbital define posição, direção de observação e vetores horizontal e vertical. Os pontos são transformados para esse referencial e projetados na tela, com tamanho aparente inversamente proporcional à profundidade. Pontos atrás ou muito próximos da câmera virtual são descartados.

As paredes são polilinhas no plano do chão, sem altura ou superfície sólida. Os marcadores do modo 2D são figuras de altura fixa, usadas apenas para representar a localização; não são medidas do corpo da pessoa.

## Implementação técnica

### Servidor e estado compartilhado

O arquivo [map_viewer_multicamera.py](map_viewer_multicamera.py) concentra a configuração, a leitura dos arquivos, o consumo AMQP e a API. Suas principais funções são:

| Função | Responsabilidade |
| --- | --- |
| `load_map()` | Lê a chave `lista` e converte as paredes em pontos `[x, y, 0]`. |
| `load_calibration()` | Lê os parâmetros da câmera e inverte a transformação extrínseca. |
| `reference_pixel()` | Seleciona o pixel de referência de cada pessoa. |
| `pixel_to_ground()` | Corrige a distorção e intersecta o raio com `Z = 0`. |
| `extract_ground_points()` | Converte os objetos de um frame em posições no chão. |
| `extract_skeletons_3d()` | Extrai IDs, rótulos, confiança e articulações 3D finitas. |
| `consume_sources()` | Assina tópicos e atualiza o último estado recebido. |
| `get_state()` | Agrupa os dados válidos para a resposta HTTP. |
| `main()` | Processa argumentos, carrega os arquivos e inicia o Uvicorn. |

Um `threading.Lock` coordena o acesso ao estado compartilhado entre o consumidor e as rotas HTTP. Cada mensagem substitui o resultado anterior de sua fonte; não há banco de dados, histórico de trajetórias nem reprodução de sessões.

O tempo de validade é medido desde o recebimento da mensagem no servidor, usando `time.time()`. O valor `--point-ttl` controla quais dados entram na resposta da API. Uma fonte pode estar ativa e enviar uma lista vazia de pessoas.

O frontend também compara os timestamps recebidos com o relógio do navegador e aplica um limite fixo de **1,5 segundo**. Portanto, aumentar `--point-ttl` não aumenta automaticamente o tempo de exibição. Diferenças entre os relógios do cliente e do servidor podem afetar essa verificação.

### Integração AMQP

As mensagens devem ser compatíveis com `is_msgs.image_pb2.ObjectAnnotations`. A conexão e as assinaturas utilizam `is_wire.core.Channel` e `Subscription`.

| Fonte | Tópico padrão | Conteúdo esperado |
| --- | --- | --- |
| Câmera individual | `SkeletonDetector.<id>.Detection` | Objetos com região e keypoints em pixels. |
| Reconstrução 3D | `SkeletonDetector.3D.Annotations` | Objetos com keypoints em coordenadas mundiais. |

**Sem `--use-3d-topic`:** todas as câmeras selecionadas são processadas individualmente e precisam de calibração local.

**Com `--use-3d-topic`:** as câmeras `1`, `2`, `3` e `4` deixam de ser assinadas individualmente. Sua representação é substituída pelo tópico 3D. Câmeras selecionadas fora desse conjunto continuam no fluxo 2D → chão e precisam de calibração.

O grupo `{1, 2, 3, 4}` está definido no código por `MULTIVIEW_CAMERA_IDS`. O tópico 3D é assinado sempre que a flag está ativa, mesmo que nenhuma dessas quatro câmeras tenha sido passada em `--camera-ids`; a lista de câmeras não filtra o conteúdo desse tópico.

### Formatos dos arquivos locais

**Mapa:** o arquivo `.npz` deve conter a chave `lista`, com uma sequência de paredes. Cada parede é uma sequência de pontos que tenham pelo menos coordenadas `x` e `y`. O programa mantém paredes com dois ou mais pontos e força `z = 0`.

O arquivo incluído, `map-coords/map_B3A1.npz`, contém quatro polilinhas. O carregamento usa `allow_pickle=True`, necessário para o formato de objetos desse arquivo; utilize mapas de origem confiável.

**Calibração:** para uma câmera com ID `N`, o nome esperado é `calib_rtN.npz`.

| Chave | Formato esperado | Significado |
| --- | --- | --- |
| `K` | Matriz `3 × 3` | Parâmetros intrínsecos originais. |
| `nK` | Matriz `3 × 3` | Parâmetros intrínsecos após a correção de distorção. |
| `rt` | Matriz `3 × 4` | Transformação extrínseca mundo → câmera. |
| `dist_coeffs` ou `dist` | Coeficientes aceitos pelo OpenCV | Distorção da lente; `dist_coeffs` tem prioridade. |

Se nenhuma chave de distorção estiver presente, são usados cinco coeficientes nulos. Os arquivos incluídos usam `dist` com formato `1 × 5`. Alguns também possuem `roi`, `escala`, `w` e `h`, mas o visualizador não utiliza essas chaves para ajustar as coordenadas. O produtor das detecções deve fornecer pixels compatíveis com a calibração e com a resolução usada nela.

### Frontend

[templates/index.html](templates/index.html) contém a página, o Canvas e os estilos. [static/map_viewer.js](static/map_viewer.js) implementa a câmera virtual, as operações vetoriais, o desenho e os controles.

O JavaScript consulta `/api/state` a cada **100 ms**, enquanto `requestAnimationFrame()` controla o desenho. Essas frequências são independentes: redesenhar a cena não significa que chegou uma nova detecção.

Os esqueletos usam uma tabela fixa de conexões, `skeletonEdges`, com IDs de articulações a partir de `1`. Essa convenção precisa corresponder ao produtor 3D e é distinta dos IDs usados para selecionar quadris e tornozelos no fluxo 2D. A cor do esqueleto é derivada de seu ID, exibido próximo à cabeça ou à primeira articulação disponível.

## Estrutura do projeto

```text
Mapara_do_maroto/
├── README.md                    # Documentação teórica e técnica
├── README.txt                   # Instruções iniciais de organização
├── map_viewer_multicamera.py     # Servidor, geometria e consumidor AMQP
├── calib_files/
│   ├── calib_rt1.npz
│   ├── calib_rt2.npz
│   ├── calib_rt3.npz
│   ├── calib_rt4.npz
│   ├── calib_rt6.npz
│   ├── calib_rt7.npz
│   └── calib_rt8.npz
├── map-coords/
│   └── map_B3A1.npz             # Paredes do ambiente
├── templates/
│   └── index.html
└── static/
    └── map_viewer.js
```

Não há calibração da câmera `5` no repositório. Para utilizá-la no fluxo 2D, é necessário fornecer `calib_rt5.npz`.

## Requisitos e instalação

- Python **3.10 ou superior**, conforme os requisitos dos pacotes de mensageria identificados no ambiente de desenvolvimento. A leitura dos arquivos e a CLI foram verificadas com Python `3.12.3`.
- NumPy e OpenCV para os arquivos de calibração e as operações geométricas.
- FastAPI e Uvicorn para o servidor HTTP.
- Pacotes que forneçam os módulos `is_wire` e `is_msgs` compatíveis com os produtores de mensagens.
- Navegador com suporte a Canvas, `fetch()` e Pointer Events.
- Broker AMQP acessível e serviços externos publicando nos tópicos configurados, para visualizar pessoas.

No ambiente inspecionado, os módulos de mensageria são fornecidos por **`is-wire-sea==2.0.1`** e **`is-msgs-sea==1.3.1`**. Os nomes dos pacotes instaláveis diferem dos nomes usados nos imports.

A partir da raiz do projeto, em Linux/macOS:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install numpy opencv-python fastapi uvicorn \
  is-wire-sea==2.0.1 is-msgs-sea==1.3.1
```

Caso os pacotes de mensageria sejam distribuídos por um índice privado ou pelo laboratório, use a fonte de instalação correspondente. Esta versão não possui `requirements.txt` nem um arquivo de versões travadas; o comando acima reúne as dependências identificadas, sem constituir uma validação de instalação em um ambiente vazio.

Verifique os imports e a CLI:

```bash
python -c 'import cv2, numpy, fastapi, uvicorn; from is_wire.core import Channel, Subscription; from is_msgs.image_pb2 import ObjectAnnotations; print("Imports OK")'
python map_viewer_multicamera.py --help
```

Não há etapa de compilação do frontend nem dependência de Node.js para executar a aplicação.

## Execução e configuração

Execute o script a partir da raiz do projeto, pois os caminhos padrão de mapa e calibração são relativos ao diretório de execução.

### Modo 2D: uma ou várias câmeras

Uma câmera, com os parâmetros padrão:

```bash
python map_viewer_multicamera.py
```

Quatro câmeras com projeção individual no chão:

```bash
python map_viewer_multicamera.py --camera-ids 1 2 3 4
```

### Modo 3D e modo misto

Representação das câmeras `1–4` pelo tópico de esqueletos 3D:

```bash
python map_viewer_multicamera.py --camera-ids 1 2 3 4 --use-3d-topic
```

Esqueletos 3D mais projeções individuais das câmeras `6`, `7` e `8`:

```bash
python map_viewer_multicamera.py \
  --camera-ids 1 2 3 4 6 7 8 \
  --use-3d-topic
```

No modo 3D puro, as calibrações de `1–4` não são carregadas pelo visualizador; elas devem ser usadas no sistema externo responsável pela reconstrução.

### Broker e arquivos personalizados

Exemplo com broker local, mapa e porta explícitos:

```bash
python map_viewer_multicamera.py \
  --camera-ids 1 2 \
  --broker-uri 'amqp://guest:guest@localhost:5672' \
  --calib-dir ./calib_files \
  --map-file ./map-coords/map_B3A1.npz \
  --point-ttl 1.5 \
  --port 3001
```

Esse comando pressupõe que o broker local e os produtores já estejam disponíveis; o script não os inicia.

| Argumento | Padrão | Função |
| --- | --- | --- |
| `--camera-ids` | `1` | IDs das câmeras; repetições são removidas. |
| `--broker-uri` | `amqp://guest:guest@10.10.50.176:30000` | Endereço AMQP configurado no código para o ambiente original. |
| `--calib-dir` | `./calib_files` | Pasta das calibrações. |
| `--map-file` | `./map-coords/map_B3A1.npz` | Arquivo do mapa. |
| `--point-ttl` | `1.5` | Validade do último frame no servidor, em segundos. |
| `--use-3d-topic` | Desativado | Habilita o tópico 3D e substitui a projeção individual das câmeras `1–4`. |
| `--skeleton-3d-topic` | `SkeletonDetector.3D.Annotations` | Nome do tópico de anotações 3D. |
| `--port` | `3001` | Porta HTTP. |

O endereço AMQP padrão pertence à configuração original; altere-o para o broker do seu ambiente. Não é um serviço público disponibilizado por este repositório.

O servidor escuta em `0.0.0.0`. Para abrir no mesmo computador, acesse **http://localhost:3001**. Em outra máquina, use `http://<IP-do-servidor>:3001`, conforme a conectividade da rede. O script é o ponto de entrada necessário para carregar a configuração; iniciar apenas `uvicorn map_viewer_multicamera:app` não executa `main()`.

## Uso da interface

| Ação | Controle |
| --- | --- |
| Girar a câmera virtual | Arrastar com o botão esquerdo. |
| Deslocar a vista no plano do chão | Arrastar com o botão do meio. |
| Aproximar ou afastar | Rolar a roda do mouse. |

Ao receber o primeiro mapa não vazio, a interface ajusta o centro e a distância da câmera virtual. As linhas escuras representam paredes, a grade indica o plano do chão e o marcador `(0, 0)` indica a origem. Marcadores vermelhos representam as posições obtidas do fluxo 2D; articulações e linhas coloridas representam os esqueletos 3D.

## API HTTP

| Método | Rota | Resposta |
| --- | --- | --- |
| `GET` | `/` | Página do visualizador. |
| `GET` | `/static/map_viewer.js` | Código JavaScript servido como arquivo estático. |
| `GET` | `/api/map` | Objeto com a lista de paredes. |
| `GET` | `/api/state` | Mapa, pontos no chão e esqueletos 3D ainda válidos. |

Exemplo **ilustrativo** de `/api/state`, em modo misto:

```json
{
  "map": {
    "walls": [[[0.0, 0.0, 0.0], [5.0, 0.0, 0.0]]]
  },
  "ground_points": {
    "points": [{"id": 0, "camera_id": 6, "x": 2.0, "y": 1.0, "z": 0.0}],
    "received_at": 1791561600.0,
    "by_camera": {
      "6": {
        "points": [{"id": 0, "camera_id": 6, "x": 2.0, "y": 1.0, "z": 0.0}],
        "received_at": 1791561600.0
      }
    }
  },
  "skeletons_3d": {
    "enabled": true,
    "topic": "SkeletonDetector.3D.Annotations",
    "skeletons": [{
      "id": 1,
      "label": "human_3d",
      "score": 0.95,
      "keypoints": [{"id": 1, "x": 1.0, "y": 2.0, "z": 1.7, "score": 0.9}]
    }],
    "received_at": 1791561600.0
  }
}
```

- `ground_points.points` agrega os pontos das câmeras válidas; `by_camera` mantém a origem e o timestamp de cada câmera. As chaves de câmera no JSON são strings.
- No fluxo 2D, `id` é o índice do objeto dentro do frame, a partir de `0`. Ele não identifica uma pessoa de forma persistente nem permite associá-la entre câmeras.
- No fluxo 3D, o ID é o `obj.id` recebido, quando não zero, ou o índice do objeto mais `1`. Sua continuidade depende do produtor externo.
- `received_at` é um timestamp Unix em segundos, referente ao recebimento no servidor. No agregado 2D, representa o frame mais recente entre as fontes válidas.
- Câmeras expiradas deixam de aparecer em `by_camera`. No bloco 3D, a lista expira, mas o timestamp do último recebimento continua disponível.
- Com o modo 3D desativado, `enabled` é `false`, `topic` é `null` e `skeletons` é uma lista vazia.

Para inspecionar a API com o servidor em execução:

```bash
curl http://localhost:3001/api/map
curl http://localhost:3001/api/state
```

## Adaptação para outro ambiente

1. Prepare um mapa `.npz` com a chave `lista` e as paredes no referencial desejado.
2. Calibre cada câmera utilizada no fluxo 2D e salve os parâmetros no formato `calib_rt<ID>.npz`.
3. Verifique a convenção da extrínseca, a escala, a resolução da imagem e os IDs das articulações usados pelo detector.
4. Configure os produtores para publicar `ObjectAnnotations` nos tópicos correspondentes.
5. Se houver reconstrução 3D, alinhe seus resultados ao referencial do mapa. Para outro grupo de câmeras, ajuste `MULTIVIEW_CAMERA_IDS`; para outra topologia de articulações, ajuste `skeletonEdges`.
6. Inicie o visualizador com os caminhos, IDs e broker adequados e compare as posições exibidas com posições conhecidas do ambiente.

O repositório não inclui ferramentas para gerar mapas ou executar a calibração. A qualidade das posições exibidas depende desses processos externos.

## Diagnóstico de problemas

| Sintoma | Verificações |
| --- | --- |
| `ModuleNotFoundError` | Ambiente virtual ativo e pacotes que fornecem `cv2`, `is_wire` e `is_msgs` instalados. |
| Arquivo `.npz` não encontrado | Diretório de execução, caminhos configurados e existência de `calib_rt<ID>.npz`. |
| `KeyError` na leitura dos arquivos | Chave `lista` no mapa e chaves `K`, `nK`, `rt` na calibração. |
| Mapa aparece, mas pessoas não | Conexão AMQP, nomes dos tópicos, publicação de mensagens e modo 2D/3D selecionado. |
| Pessoas em posições incorretas | Extrínseca, distorção, escala, referencial e resolução das detecções. |
| Pessoas desaparecem rapidamente | Intervalo entre mensagens, TTL do servidor e limite de 1,5 segundo do frontend. |
| Esqueletos não aparecem | Flag `--use-3d-topic`, tópico configurado, coordenadas finitas e timestamps recentes. |
| Porta ocupada | Use outra porta com `--port`. |

O terminal registra a configuração e erros de recebimento. O console do navegador ajuda a identificar falhas HTTP ou de desenho. O carregamento do mapa pode funcionar mesmo quando o consumidor AMQP falha; isso não comprova que as detecções estão chegando.

## Limitações e evolução

As limitações atuais incluem:

- **Chão plano:** a projeção 2D assume `Z = 0`; escadas, rampas e diferentes níveis exigem outro modelo geométrico.
- **Ausência de fusão e tracking 2D:** detecções de câmeras diferentes são agregadas sem remover duplicatas nem preservar a identidade da pessoa entre frames.
- **Ausência de sincronização temporal:** cada fonte mantém seu último frame, sem alinhamento pelo instante de captura.
- **Confiança sem filtragem:** o fluxo 2D não aplica limiar aos scores dos keypoints; o fluxo 3D preserva os scores, mas não os usa para filtrar o desenho.
- **Validação geométrica parcial:** além dos casos degenerados e valores não finitos, não há filtro para interseções atrás da câmera ou fora da região do mapa.
- **Conexão AMQP sem recuperação explícita:** a conexão inicial ocorre fora do tratamento de erros do laço; sua falha pode encerrar a thread. Não há rotina implementada de reconexão.
- **Persistência e operação:** não há histórico, banco de dados, autenticação HTTP nem configuração de HTTPS na aplicação.
- **Tratamento de erros no frontend:** o JavaScript tenta escrever em um elemento `#status` que não está presente no HTML; o próprio tratamento de falhas de consulta pode gerar outra exceção.
- **Reprodutibilidade em desenvolvimento:** não há arquivo de dependências travadas nem suíte de testes no repositório. A integração completa exige broker e produtores externos.

Possíveis evoluções incluem unificar o TTL entre servidor e navegador, melhorar o estado de conexão na interface, implementar reconexão AMQP, adicionar testes com mensagens conhecidas, documentar versões de dependências, sincronizar fontes e incorporar associação de pessoas entre câmeras. Gravação de trajetórias, análise de ocupação e eventos por região também podem ser desenvolvidos sobre a API atual.

## Possíveis aplicações

As aplicações abaixo são possibilidades de uso e extensão. Recursos como trajetórias, mapas de calor, alertas e reprodução de áudio precisam de implementação adicional.

| Aplicação | Como o projeto pode contribuir | Extensões necessárias |
| --- | --- | --- |
| **Pesquisa em visão computacional** | Inspecionar projeções 2D, calibrações e esqueletos reconstruídos em relação ao mapa. | Métricas de erro, dados de referência e registro dos experimentos. |
| **Ensino de geometria computacional** | Demonstrar referenciais, parâmetros de câmera, interseção raio–plano e projeção em perspectiva. | Exemplos didáticos e fontes de dados simuladas. |
| **Observação da ocupação de espaços** | Visualizar onde aparecem detecções em laboratórios, salas ou áreas de circulação. | Remoção de duplicatas, contagem consistente e regiões de interesse. |
| **Análise de circulação e layout** | Usar posições para estudar percursos e concentração de pessoas. | Tracking persistente, histórico, estatísticas e mapas de calor. |
| **Áudio-tour e experiências interativas** | Usar a posição estimada para associar pessoas a pontos de interesse em uma visita. | Eventos por região, identificação da sessão e integração com reprodução de áudio. |
| **Interfaces para ambientes inteligentes** | Fornecer posições a aplicações que respondam à presença ou ao movimento. | Regras de eventos, integração com outros sistemas e avaliação da confiabilidade. |
| **Robótica e interação humano–robô** | Servir como painel de observação das posições humanas em um ambiente compartilhado. | Transformações de referenciais, integração com o sistema robótico e validação de latência e precisão. |
