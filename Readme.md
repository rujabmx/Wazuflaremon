# Arquitectura Multicloud Segura para Wazuh con Cloudflare Zero Trust

Este repositorio contiene la documentación, diagramas y configuraciones necesarias para exponer de forma segura el **Wazuh Dashboard** y conectar aproximadamente **200 agentes distribuidos en arquitecturas Multicloud (Huawei Cloud, AWS y Azure)** utilizando la red privada de **Cloudflare Zero Trust** de manera 100% gratuita.

---

## 📌 Arquitectura General

El **Wazuh Manager** reside centralizado en **Huawei Cloud**. Los agentes de otras nubes públicas se comunican con él a través de túneles de infraestructura de Cloudflare (`cloudflared`) actuando como Gateways de subred, evitando la apertura de puertos (`1514`/`1515`) a internet general.

```text
[ EQUIPO DE TI ] ──( Autenticación Access )──> [ Cloudflare Edge ] ──> [ Wazuh Dashboard (Puerto 5601) ]
                                                       │
                                            ( Cloudflare Zero Trust )
                                                       │
         ┌─────────────────────────────────────────────┴─────────────────────────────────────────────┐
         ▼                                             ▼                                             ▼
┌──────────────────────────────┐              ┌──────────────────────────────┐              ┌──────────────────────────────┐
│        HUAWEI CLOUD          │              │             AWS              │              │            AZURE             │
│ (Red Privada Nativa)         │              │ (VPC Route Table redirection)│              │ (UDR / Route Table Forward)  │
│                              │              │                              │              │                              │
│  [ Wazuh Manager / Tunnel ]  │              │    [ Micro EC2 Gateway ]     │              │     [ Micro VM Gateway ]     │
│       (IP: 10.10.0.5)        │              │        (cloudflared)         │              │        (cloudflared)         │
│              ▲               │              │              ▲               │              │              ▲               │
│              │               │              │              │               │              │              │               │
│      [ ~70 Agentes ]         │              │       [ ~70 Agentes ]        │              │       [ ~60 Agentes ]        │
└──────────────────────────────┘              └──────────────────────────────┘              └──────────────────────────────┘
```

---

## 🛠️ Guía de Despliegue por Componente

### 1. Servidor Central: Wazuh Manager (Huawei Cloud)

1. Instala el agente de Cloudflare (`cloudflared`) en el servidor del Wazuh Manager.
2. Desde el panel de Cloudflare Zero Trust, crea un túnel y asócialo al servidor.
3. **Exposición del Dashboard:** Añade un *Public Hostname* (ej. `://tuempresa.com`) apuntando localmente a `https://localhost:5601`.
4. **Habilitación de Red Privada:** En la pestaña *Private Networks* del túnel, declara la IP privada del propio manager: `10.10.0.5/32`.
5. **Protección Web:** Ve a *Access > Applications* y crea una regla que restrinja el subdominio `://tuempresa.com` únicamente a correos corporativos autorizados mediante un código OTP (One-Time PIN).

---

### 2. Configuración de Gateways Remotos (AWS & Azure)

Despliega una microinstancia Linux dedicada en la VPC de AWS y en la VNet de Azure. En ambas, instala `cloudflared` y vincula cada una a un túnel independiente en Cloudflare. 

Habilita el reenvío de paquetes IP en el sistema operativo de ambas microinstancias ejecutando:
```bash
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

#### Configuración Específica en AWS:
1. Selecciona la instancia EC2 de Cloudflare.
2. Ve a **Acciones > Redes > Cambiar comprobación de origen/destino** y selecciona **Desactivar**.
3. Ve a la **Route Table** de la VPC de tus servidores y añade la siguiente ruta:
   * **Destination:** `10.10.0.5/32`
   * **Target:** `Instance` -> *(Selecciona la EC2 de Cloudflare)*

#### Configuración Específica en Azure:
1. Ve a la Interfaz de Red (NIC) de la VM de Cloudflare.
2. Ve a **Configuraciones de IP**, activa la casilla **IP Forwarding** (Reenvío de IP) y guarda los cambios.
3. Crea una **Tabla de Rutas (UDR)** en Azure y añade una regla:
   * **Destination address prefix:** `10.10.0.5/32`
   * **Next hop type:** `Virtual Appliance`
   * **Next hop address:** *(IP privada local de la VM de Cloudflare)*
4. Asocia esta Tabla de Rutas a la subred donde viven tus servidores de Azure.

---

### 3. Configuración de los 200 Agentes Wazuh

En los 200 servidores de producción (sin importar la nube en la que se encuentren), el archivo de configuración del agente debe apuntar transparentemente a la IP privada central:

```xml
<!-- /var/ossec/etc/ossec.conf o C:\Program Files (x86)\ossec-agent\ossec.conf -->
<ossec_config>
  <client>
    <server>
      <address>10.10.0.5</address>
      <port>1514</port>
      <protocol>tcp</protocol>
    </server>
  </client>
</ossec_config>
```

---

## 💰 Costo del Ciclo de Vida de la Infraestructura

| Servicio | Cantidad / Configuración | Costo Mensual |
| :--- | :--- | :--- |
| **Cloudflare DNS & Dominios** | Administración y delegación de registros | **0.00 USD** |
| **Cloudflare Tunnels** | Conexión e interconexión de redes privadas | **0.00 USD** |
| **Cloudflare Access** | Autenticación previa al Dashboard (Hasta 50 usuarios) | **0.00 USD** |
| **Infraestructura de Red** | Túneles de infraestructura (*site-to-site*) ilimitados | **0.00 USD** |
| **TOTAL CLOUDFLARE** | **Arquitectura Zero Trust Completa** | **0.00 USD** |

*Nota: Solo se contemplan los costos operativos estándar de cómputo de las nubes públicas (Huawei Cloud, AWS y Azure) por el alojamiento de las instancias de servidores y pasarelas de red.*

---
🛡️ **Mantenido por el Equipo de Seguridad e Infraestructura de TI.**
