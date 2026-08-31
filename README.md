# 🦊 FoxGlove + MapAnything — Reconstrução 3D em Tempo Real

> Pipeline ROS 2 que conecta um nó de câmera, um nó de IA (MapAnything) e um nó de fusão para gerar uma nuvem de pontos 3D do ambiente, visualizada no FoxGlove Studio.

---

## 🚀 Sobre o Projeto

Este projeto integra o modelo **MapAnything** (Meta AI) a um pipeline ROS 2 para reconstrução 3D métrica em tempo real. Diferente de modelos de profundidade monocular "puros" (como o Depth Anything, que gera apenas mapas de profundidade *relativa*), o MapAnything estima diretamente **pose de câmera, profundidade métrica e o point cloud** a partir de uma ou mais imagens, com fusão multi-view embutida — o que o torna a escolha natural para este caso de uso.

**Arquitetura do pipeline:**

```
[Nó de Câmera] → imagem (sensor_msgs/Image)
      ↓
[Nó de IA — MapAnything] → profundidade métrica + pose
      ↓
[Nó de Fusão] → nuvem de pontos acumulada (sensor_msgs/PointCloud2)
      ↓
[FoxGlove Bridge] → visualização em tempo real (FoxGlove Studio)
```

---

## 📌 Pré-requisitos

* Ubuntu 26.04 LTS (Resolute Raccoon)
* ROS 2 Lyrical Luth (distro correspondente ao Ubuntu 26.04 — **não confundir com Jazzy**, que é para Ubuntu 24.04)
* Python 3.12+ (padrão do 26.04)
* GPU NVIDIA com drivers instalados (recomendado; PyTorch faz fallback para CPU se não houver)
* FoxGlove Studio instalado na máquina cliente (para visualização)

---

## 🔧 Como Instalar

### 1. Preparação do sistema e chaves do ROS 2

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y software-properties-common curl gnupg lsb-release build-essential python3-pip python3-venv git

sudo add-apt-repository universe -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

### 2. Instalação do ROS 2 Lyrical Luth

```bash
sudo apt update

# ATENÇÃO: bug conhecido no pacote sugerido 'hyperspec' (via rqt) quebra em Ubuntu 26.04.
# --no-install-suggests evita o problema.
sudo apt install -y --no-install-suggests ros-lyrical-desktop ros-dev-tools

# Configuração do ambiente bash
echo "source /opt/ros/lyrical/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

### 3. Instalação do FoxGlove Bridge e integradores ROS

```bash
sudo apt install -y \
  ros-lyrical-foxglove-bridge \
  ros-lyrical-cv-bridge \
  ros-lyrical-image-transport \
  ros-lyrical-camera-info-manager \
  ros-lyrical-pcl-conversions
```

`ros-lyrical-pcl-conversions` é usado para converter a saída do nó de fusão (profundidade + intrínsecos) em `sensor_msgs/PointCloud2`.

### 4. Instalação do PyTorch, MapAnything e dependências Python

Verifique antes qual driver NVIDIA está instalado (`nvidia-smi`) para escolher a wheel de CUDA correta — o índice abaixo (CUDA 12.8) é um bom padrão atual, mas confira em [pytorch.org/get-started](https://pytorch.org/get-started/locally/) se seu driver pede outra versão.

```bash
python3 -m pip install --upgrade pip

# PyTorch com suporte a CUDA 12.8 (ajuste o índice conforme seu driver)
pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128

# OpenCV e bibliotecas auxiliares de processamento
pip3 install opencv-python numpy matplotlib scipy

# MapAnything (Meta AI) — clonar e instalar em modo editável
git clone https://github.com/facebookresearch/map-anything.git
cd map-anything
pip3 install -e .
cd ..
```

> Se o repositório oficial do MapAnything exigir um checkpoint separado, baixe-o conforme as instruções do próprio repositório antes de rodar o nó de IA.

### 5. Verificação de GPU, CUDA e dependências

```bash
python3 -c "import torch, cv2; print(f'PyTorch: {torch.__version__}\nCUDA Disponível: {torch.cuda.is_available()}\nGPU: {torch.cuda.get_device_name(0) if torch.cuda.is_available() else \"Apenas CPU\"}\nOpenCV: {cv2.__version__}')"
```

Saída esperada (com GPU disponível):

```
PyTorch: 2.11.0
CUDA Disponível: True
GPU: NVIDIA GeForce RTX ....
OpenCV: 4.x.x
```

---

## 🧩 Estrutura do Workspace ROS 2

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src

# seus pacotes:
# - camera_node/      → publica sensor_msgs/Image + CameraInfo
# - mapanything_node/ → assina Image, roda inferência, publica profundidade/pose
# - fusion_node/       → assina profundidade+pose, acumula e publica PointCloud2

cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash
```

---

## ▶️ Executando o Pipeline

```bash
# Terminal 1 — nó da câmera
ros2 run camera_node camera_publisher

# Terminal 2 — nó de IA (MapAnything)
ros2 run mapanything_node depth_estimator

# Terminal 3 — nó de fusão (gera a nuvem de pontos)
ros2 run fusion_node point_cloud_fusion

# Terminal 4 — ponte para o FoxGlove Studio
ros2 launch foxglove_bridge foxglove_bridge_launch.xml
```

No FoxGlove Studio, conecte via `ws://localhost:8765` e adicione um painel **3D** assinando o tópico `PointCloud2` publicado pelo `fusion_node`.

---

## 🗺️ Roadmap

- [ ] Nó de câmera publicando `Image` + `CameraInfo`
- [ ] Nó MapAnything publicando profundidade métrica + pose
- [ ] Nó de fusão acumulando point cloud multi-frame
- [ ] Bridge FoxGlove funcional com visualização ao vivo
- [ ] Ajuste de performance (mixed precision / batch de frames)

---

## 📎 Notas

- Este projeto substitui a abordagem inicial baseada em Depth Anything: Depth Anything entrega apenas profundidade *relativa* por imagem, exigindo calibração manual de escala e back-projection próprio para virar nuvem de pontos. O MapAnything resolve isso nativamente (profundidade métrica + pose + fusão multi-view), encaixando melhor no objetivo de reconstrução 3D.
- Ubuntu 26.04 usa o distro **ROS 2 Lyrical Luth**, não Jazzy (que é para 24.04) nem Humble (22.04).
