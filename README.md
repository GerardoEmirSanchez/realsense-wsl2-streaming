# Guía de Instalación, Configuración y Streaming de Intel RealSense en WSL2

Guía paso a paso para limpiar el entorno, reiniciar a cero y replicar todo el proceso de instalación y configuración de cámaras Intel RealSense (serie D400) en WSL2 con visualización web local.

---

## Fase 0: Limpieza total para prueba desde cero
Pasos para limpiar el entorno de trabajo actual y regresar al estado inicial sin reinstalar Ubuntu.

### 1. En la terminal de Ubuntu (WSL)
```bash
# Desactivar entorno si está activo
deactivate 2>/dev/null

# Detener cualquier proceso de Python residual
killall -9 python python3 2>/dev/null

# Eliminar el entorno virtual, scripts y capturas generadas
cd ~
rm -rf vision_env capture
rm -f s1_realsense_pixeles.py visor_web_camara.py
```

### 2. En Windows PowerShell (Administrador)
```powershell
# Apagar WSL por completo para liberar hardware y memoria
wsl --shutdown
```

---

## Fase 1: Configuración en Windows (Host)

### 1. Instalar usbipd-win
En **PowerShell (como Administrador)**:
```powershell
winget install --interactive --exact dorssel.usbipd-win
```

### 2. Enlazar la cámara Intel RealSense a WSL
Con la cámara conectada a un puerto USB 3.0:

1. Lista los dispositivos USB conectados para localizar el `BUSID` de la RealSense:
   ```powershell
   usbipd list
   ```
2. Comparte el puerto del dispositivo con WSL (solo se requiere una vez):
   ```powershell
   usbipd bind --busid <TU-BUSID>
   ```
3. Conectar el dispositivo a la instancia activa de WSL:
   ```powershell
   usbipd attach --wsl --busid <TU-BUSID>
   ```

---

## Fase 2: Configuración en Ubuntu (WSL2)

Toca la tecla `Windows` y escribe `Ubuntu` y ábrelo como **administrador**

En la terminal de **Ubuntu**:

### 1. Dependencias del sistema y soporte USB
```bash
sudo apt update && sudo apt install -y usbutils python3-pip python3-venv libgl1 libglib2.0-0
```

### 2. Verificar detección de hardware
```bash
lsusb
```
*Debe listar: `Intel Corp. Intel(R) RealSense(TM) Depth Camera`.*

### 3. Permisos de hardware
```bash
sudo usermod -aG video,plugdev $USER
sudo chmod 666 /dev/video*
sudo chmod -R 777 /dev/bus/usb/
```

### 4. Entorno virtual de Python
```bash
# Crear entorno virtual limpio
python3 -m venv ~/vision_env

# Activar el entorno virtual
source ~/vision_env/bin/activate

# Actualizar pip e instalar librerías necesarias
pip install --upgrade pip
pip install opencv-python numpy pyrealsense2 flask
```

---

## Fase 3: Script de Captura y Streaming (`visor_web_camara.py`)

Con el entorno virtual activo (`vision_env`), genera el script ejecutando el siguiente bloque en la terminal:

```bash
cat << 'EOF' > ~/visor_web_camara.py
from flask import Flask, Response
import cv2
import numpy as np
import os
import pyrealsense2 as rs

app = Flask(__name__)
os.makedirs("capture", exist_ok=True)

# 1. Configuración del sensor Intel RealSense
pipe = rs.pipeline()
cfg = rs.config()
cfg.enable_stream(rs.stream.color, 640, 480, rs.format.bgr8, 30)
pipe.start(cfg)

def flujo_video():
    while True:
        frames = pipe.wait_for_frames()
        color = frames.get_color_frame()
        if not color:
            continue

        # Convertir buffer nativo a ndarray de NumPy
        frame = np.asanyarray(color.get_data())
        h, w, _ = frame.shape

        # Elementos didácticos: Centro óptico y ROI del cajón
        y_c, x_c = h // 2, w // 2
        cv2.circle(frame, (x_c, y_c), 6, (0, 0, 255), -1)
        
        y1, y2 = int(h * 0.2), int(h * 0.8)
        x1, x2 = int(w * 0.2), int(w * 0.8)
        cv2.rectangle(frame, (x1, y1), (x2, y2), (0, 255, 0), 2)
        
        cv2.putText(frame, f"Centro Optico ({x_c},{y_c})", (x_c + 10, y_c), 
                    cv2.FONT_HERSHEY_SIMPLEX, 0.5, (0, 0, 255), 1)

        # Persistencia continua del último frame (Muestra A6)
        cv2.imwrite("capture/captura_cajon_000.jpg", frame)

        # Codificación JPEG para transmisión HTTP multipart
        ok, buffer = cv2.imencode('.jpg', frame)
        if not ok:
            continue

        yield (b'--frame\r\n'
               b'Content-Type: image/jpeg\r\n\r\n' + buffer.tobytes() + b'\r\n')

@app.route('/')
def video_feed():
    return Response(flujo_video(), mimetype='multipart/x-mixed-replace; boundary=frame')

if __name__ == '__main__':
    print("\n=======================================================")
    print(" Transmision activa. Abre tu navegador en Windows y entra a:")
    print(" http://localhost:5000")
    print("=======================================================\n")
    app.run(host='0.0.0.0', port=5000, threaded=True)
EOF
```

---

## Fase 4: Ejecución y Validación

### 1. Iniciar servidor
```bash
python ~/visor_web_camara.py
```
En la terminal debe verse el mensaje:
`* Running on [http://127.0.0.1:5000](http://127.0.0.1:5000)`

### 2. Visualización en navegador
Abre Chrome, Edge o Firefox en Windows e ingresa a:
```text
http://localhost:5000
```

### 3. Verificar captura en Windows
Detén el servidor con `Ctrl + C` y abre la carpeta de capturas:
```bash
explorer.exe capture
```
Abre `captura_cajon_000.jpg` con el visor de imágenes de Windows y confirma que mide $640 \times 480$ píxeles.

