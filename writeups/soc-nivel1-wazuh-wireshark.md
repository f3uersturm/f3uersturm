# Lab SOC Nivel 1: SIEM con Wazuh + Análisis de tráfico con Wireshark

> [!TIP]
> 📄 Este write-up está resumido en texto. La [documentación original completa](./Analista_SOC_2024.pdf) incluye más de 20 capturas de pantalla paso a paso de todo el proceso.

**Contexto:** BootCamp Analista SOC Nivel 1 (DOJO), 2024. Home lab armado y documentado de forma independiente para cumplir los requisitos del curso: entorno virtualizado, implementación de un SIEM (Wazuh) y análisis de tráfico de red (Wireshark) con identificación de una muestra maliciosa real.

## 1. Entorno virtualizado

Ambiente de laboratorio con VirtualBox, pensado como espacio controlado y aislado para las prácticas siguientes.

- **Kali Linux** como sistema operativo base — elegido por experiencia previa (uso desde 2015) por sobre Ubuntu, que resultaba más intuitivo pero no era el entorno con el que venía trabajando.
- **Windows 11** (imagen de evaluación) como máquina víctima/endpoint a monitorear.
- Descarga e instalación de VirtualBox + Extension Pack, imágenes de Ubuntu y Windows desde fuentes oficiales (osboxes.org, Microsoft Developer).

**Incidente real durante el armado:** al intentar importar el `.ova` de Windows sobre Kali como sistema anfitrión, la VM abortó por falta de memoria física disponible — la partición de intercambio (swap) había quedado subdimensionada en la instalación de Kali. Solución aplicada: migrar Windows 11 a sistema base (anfitrión) y correr Kali como una VM dentro de él, en vez de al revés. Las VMs de Kali, Windows y Wazuh quedaron funcionando sobre ese esquema, con red NAT compartida (`NatNetwork5`) para que se reconocieran entre sí.

**Lección de fondo:** la memoria de intercambio insuficiente en el host puede tumbar una importación de VM aunque el disco tenga espacio de sobra — vale la pena revisar el swap, no solo el almacenamiento, antes de armar un lab con múltiples VMs pesadas.

## 2. Implementación de Wazuh (SIEM)

- Despliegue de Wazuh v4.9.1 (OVA) como VM independiente, con las credenciales por defecto (`wazuh-user`).
- Validación de la IP interna del servidor con `ip addr` (interfaz `eth0`).
- Acceso al panel web (`https://<ip>/app/login`, credenciales `admin`/`admin`).
- **Despliegue de agente en el endpoint Windows:** desde *Server Management → Endpoints Summary → Deploy new agent*, generación del comando de instalación (PowerShell, con la IP del manager embebida) y arranque del servicio (`NET START WazuhSvc`).
- Verificación de conexión: el agente aparece como `active` en el panel, con su IP y sistema operativo correctamente identificados.
- Exploración de **Threat Intelligence → Threat Hunting**: dashboard con evolución de alertas por nivel, técnicas MITRE ATT&CK más frecuentes, y eventos por agente — incluida la regla `504` (desconexión de agente), inspeccionada en detalle junto con su mapeo a compliance (GDPR, HIPAA, PCI-DSS, TSC).

## 3. Análisis de tráfico con Wireshark

Ejercicio de análisis forense sobre capturas `.pcap` provistas por la cátedra, siguiendo una metodología deliberada (no aplicar filtros a ciegas sin antes entender qué se está buscando):

1. **Configuración previa:** agregado de columnas de puerto origen/destino para facilitar la lectura.
2. **Protocol Hierarchy** (*Statistics → Protocol Hierarchy*): visión general de qué protocolos dominan el tráfico — presencia de paquetes *Malformed* (12, posible indicio de problemas de red o ataque) y tráfico cifrado (TLS) identificado y descartado como no analizable en texto plano.
3. **Resolved Addresses**: identificación de fabricantes/dominios asociados a las MACs e IPs presentes.
4. **Conversations**: ordenado por tamaño de paquete para localizar transferencias de archivo grandes — señal típica de posible malware.
5. **Extracción de objetos HTTP** (*File → Export Objects → HTTP*): entre los objetos transferidos aparece un archivo llamado `neuro.msi`.
6. **Verificación de firma de archivo:** en vez de confiar en la extensión `.msi`, se extrajo el encabezado hexadecimal real (`xxd neuro.msi | more`) y se comparó contra la firma oficial de un instalador de Microsoft (`D0 CF 11 E0 A1 B1 1A E1`, referencia: [gary Kessler's file signature table](https://www.garykessler.net/library/file_sigs.html)) — coincidencia exacta, confirmando que el archivo era efectivamente un instalador y no un ejecutable disfrazado con otra extensión.
7. **Confirmación en VirusTotal:** el hash del archivo fue marcado como malicioso por 40 de 66 motores antivirus, con etiquetas consistentes entre proveedores (`Trojan.Win32.Sheller`, `Trojan.Win64.Brutel`, `Trojan.Agent.Latrodectus`) — indicando una familia de troyano dropper conocida.

## Conclusión

El laboratorio cubre el ciclo completo de un analista SOC nivel 1: desde levantar la infraestructura de monitoreo (SIEM + agente) hasta ejecutar un análisis de tráfico metódico que termina en la identificación positiva de una muestra maliciosa real, validada por dos vías independientes (firma de archivo + motores AV) en vez de una sola fuente.

---
*Documentado originalmente en 2024 como parte del BootCamp Analista SOC Nivel 1 (DOJO); reescrito para este portfolio.*
