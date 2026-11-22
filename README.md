# lab-redes-wireshark2
# lab-redes-wireshark2

## 🛡️ Análisis de Tráfico TCP y Gestión de Firewall con UFW y Wireshark

Laboratorio práctico de seguridad en redes donde se configura el firewall **UFW (Uncomplicated Firewall)** en **Ubuntu Server** para restringir y posteriormente habilitar tráfico HTTP (puerto `80/tcp`), manteniendo el acceso administrativo SSH (puerto `22/tcp`). El comportamiento de los paquetes a nivel de red se captura e inspecciona desde **Kali Linux** utilizando **Wireshark**.

---

### 🖥️ Entorno del Laboratorio

| Rol | Sistema Operativo | Dirección IP | Interfaz |
| :--- | :--- | :--- | :--- |
| **Servidor (Objetivo)** | Ubuntu Server (VirtualBox) | `192.168.56.10` | Red Interna / Host-Only |
| **Cliente / Analizador** | Kali Linux | `192.168.56.20` | `eth0` |

---

### 📋 Desarrollo del Laboratorio

#### 1. Configuración Inicial de UFW y Bloqueo del Puerto 80 (HTTP)

En **Ubuntu Server**, se restablece la configuración del firewall a sus valores por defecto, se definen las políticas base (denegar todo el tráfico entrante y permitir el saliente), se asegura el acceso remoto por SSH (`22/tcp`) y se bloquea explícitamente el puerto web (`80/tcp`).

    # Restablecer reglas anteriores
    sudo ufw reset

    # Establecer políticas por defecto
    sudo ufw default deny incoming
    sudo ufw default allow outgoing

    # Permitir acceso administrativo SSH y bloquear servicio web HTTP
    sudo ufw allow 22/tcp
    sudo ufw deny 80/tcp

    # Activar firewall y verificar estado
    sudo ufw enable
    sudo ufw status verbose

**Evidencia en Ubuntu Server:**

![Configuración inicial de UFW](VirtualBox_ubuntu%20server_22_11_2026_02_12_06.png)

---

#### 2. Captura de Tráfico Bloqueado en Wireshark (Kali Linux)

Desde **Kali Linux** (`192.168.56.20`), se inicia una captura en la interfaz `eth0` aplicando el filtro de visualización:

    host 192.168.56.10 and port 80

Al intentar establecer una conexión HTTP hacia `192.168.56.10`:
1. **Resolución ARP:** El cliente pregunta por la dirección MAC de `192.168.56.10` y obtiene respuesta (`08:00:27:2f:bc:97`).
2. **Intento de Three-Way Handshake fallido:** Kali envía el paquete inicial `TCP [SYN]` desde el puerto origen `42946` hacia el puerto destino `80`.
3. **Descarte silencioso (DROP):** Debido a la regla `DENY IN` de UFW, el servidor descarta los paquetes sin enviar respuesta (`RST` o `SYN, ACK`). Como resultado, Kali realiza múltiples retransmisiones (`[TCP Retransmission]`) hasta agotar el tiempo de espera (*timeout*).

**Evidencia en Wireshark (Puerto Bloqueado):**

![Captura de retransmisiones TCP SYN en Wireshark](Captura%20de%20pantalla%202026-11-22%20022412.png)

---

#### 3. Desbloqueo del Puerto 80/tcp en Ubuntu Server

Una vez comprobado el bloqueo a nivel de paquetes, se elimina la regla de denegación y se habilita el tráfico entrante hacia el puerto `80/tcp`.

    # Eliminar regla de bloqueo y permitir puerto 80/tcp
    sudo ufw delete deny 80/tcp
    sudo ufw allow 80/tcp

    # Verificar las nuevas reglas activas
    sudo ufw status verbose

**Evidencia en Ubuntu Server:**

![Regla actualizada permitiendo puerto 80](Captura%20de%20pantalla%202026-11-22%20022831.png)

---

#### 4. Verificación de Tráfico Permitido en Wireshark

Con el puerto `80/tcp` abierto, se repite la petición web desde Kali Linux hacia `192.168.56.10` manteniendo el mismo filtro en Wireshark:

1. **Three-Way Handshake exitoso:**
   * Paquete 1: `192.168.56.20 → 192.168.56.10` `[SYN]`
   * Paquete 2: `192.168.56.10 → 192.168.56.20` `[SYN, ACK]`
   * Paquete 3: `192.168.56.20 → 192.168.56.10` `[ACK]`
2. **Intercambio HTTP:** Se envía la petición `GET / HTTP/1.1` y el servidor responde con `HTTP/1.1 200 OK (text/html)` entregando el contenido de la página.
3. **Cierre de conexión:** Se finaliza la sesión TCP de forma ordenada mediante intercambio de banderas `[FIN, ACK]` y `[ACK]`.

**Evidencia en Wireshark (Tráfico Permitido):**

![Captura de handshake TCP y respuesta HTTP 200 OK](Captura%20de%20pantalla%202026-11-22%20023056.png)

---

### 🔍 Conclusiones

* **Comportamiento de `DENY` en UFW:** A diferencia de `REJECT` (que devuelve un paquete `TCP RST` o `ICMP Port Unreachable`), la acción `DENY` descarta silenciosamente los paquetes entrantes (`DROP`), provocando que el cliente intente retransmitir el paquete `SYN` repetidamente.
* **Importancia del orden y acceso administrativo:** Mantener la regla `ALLOW IN` en el puerto `22/tcp` evita perder la conexión remota al activar el firewall con una política por defecto de denegación entrante (`default deny incoming`).



