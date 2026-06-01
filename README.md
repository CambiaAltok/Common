# CambiaAltok
Es una aplicacion web para el envio de divisas de un tipo de moneda a otra mediante un intermediario. El software es construido a medida del cliente divida basicamente en dos fases. 
El presente documento es tecnico, describiendo las bases sobre las que se esta contruyendo el software.

# Stack Tecnológico

- **Frontend:** Angular (v20) SPA — Separado en dos portales (Usuario Final y Administración/Gerencia).
- **Backend Core:** .NET Core con arquitectura limpia e implementación del patrón **CQRS**.
- **ORM / Acceso a Datos:** Entity Framework Core (Comandos) y Dapper / Consultas directas optimizadas (Lecturas).
- **Autenticación:** Servidor OAuth 2.0 independiente (Emisión de Bearer Tokens JWT).
- **Reportes:** Api para generacion y mantenimiento de reportes
- **Bases de Datos:** SQL Server (2 Instancias/Bases aisladas: Aplicación y Autenticación).
- **Almacenamiento:** API de Storage dedicada para la carga y gestión de archivos.
- **Hosting:** MonsterAsp (Ambientes de producción y desarrollo).

# Flujo de Datos y Seguridad

1. **Autenticación:** Las SPAs de Angular interactúan con el **Sitio de Autenticación**. Tras validar credenciales contra la **BD de Autenticación (SQL)**, se retorna un token JWT que se adjunta en la cabecera Authorization: Bearer <token> de cada petición HTTP.
1. **Segregación de Consultas y Comandos (CQRS):** \* Las operaciones de escritura (mutaciones de estado, creación de transacciones) se procesan a través de comandos utilizando **Entity Framework Core**.
   1. Las operaciones de lectura masiva o listados consumen vistas u consultas optimizadas para evitar bloqueos y mejorar los tiempos de respuesta.
1. **Persistencia de Archivos:** La **API de Storage** valida los privilegios del token Bearer antes de guardar documentos de identidad o comprobantes directamente en el sistema de archivos de MonsterAsp y otro proveedor.


# Pipeline de CI/CD (GitHub Actions)

El despliegue está completamente automatizado desde este repositorio hacia **MonsterAsp** mediante workflows estructurados por ambientes.

# Estrategia de Ramificación y Despliegue

- **Rama dev:** Despliega de forma automática al ambiente de **Development** (Pruebas y QA).
- **Rama Prod:** Despliega de forma automática al ambiente de **Production** (Entorno vivo de cara al usuario).
- La rama **Dev** es asignada por defecto como la principal en todos los repositorios

# Matriz de Infraestructura y Dominios

|**Componente**|**Development**|**Production MonsterAsp**|` `**Production**|
| :- | :- | :- | :- |
|**Angular SPA (Cliente)**|cambiaaltok-dev.premiumasp.net|cambiaaltok.premiumasp.net|www.cambiaAltok.com|
|**Angular SPA (Admin)**|cambiaaltok-admin-dev.premiumasp.net|cambiaaltok-admin.premiumasp.net|admin.cambiaAltok.com|
|**Sitio Autenticación**|cambiaaltok-auth.premiumasp.net|cambiaaltok-auth.premiumasp.net|auth.cambiaAltok.com|
|**API Principal (.NET)**|cambiaaltok-api-dev.premiumasp.net|cambiaaltok-api-dev.premiumasp.net|api.cambiaAltok.com|
|**API Storage**|cambiaaltok-storage-api-dev.premiumasp.net|cambiaaltok-storage-api.premiumasp.net|storage.cambiaAltok.com|
|**API Reports**|cambiaaltok-reports-api-dev.premiumasp.net|cambiaaltok-reports-api.premiumasp.net|reports.cambiaAltok.com|
|**BD Autenticación (SQL)**|**db54305**|**db54305**|**db54305**|
|**BD Aplicación (SQL)**|**db53882**|**db53848**|**db53848**|

**Angular SPA (Cliente)** es publico y al autenticarse el usuario podra realizar las transacciones necesarias. 

**Angular SPA (Admin)** es para la gestion de datos y transacciones por medio del intermediario y administrador del sistema. Debera tener riguroso control de acceso debido al gerenciamiento de datos sensibles. 


# Variables de Configuración

- **Backend (.NET):** Controlado por appsettings.Development.json y appsettings.Production.json.
- **Frontend (Angular):** Resuelto en tiempo de compilación utilizando la estructura de carpetas src/environments/.
- **Secretos de GitHub:** Las credenciales de conexión FTP/SFTP y APIs de MonsterAsp se gestionan de forma segura mediante los *Repository Secrets*

## 🗺️ Arquitectura del Sistema (Modelo C4 - Contenedores)
El sistema se compone de múltiples aplicaciones independientes comunicadas a través de protocolos HTTP/REST, distribuidas en el hosting **MonsterAsp**.

```text
+-----------------------------------------------------------------------------------------------------------------------------+
|                                                 ZONA DE CLIENTE (FRONTEND)                                                  |
|                                                                                                                             |
|  +-----------------------------+                                            +-----------------------------+                 |
|  |   Angular SPA (Usuario)     |                                            |    Angular SPA (Admin)      |                 |
|  |  Portal de Transacciones   |                                            |   Gerenciamiento de Datos   |                 |
|  +--------------+--------------+                                            +--------------+--------------+                 |
|                 |                                                                          |                                |
+-----------------|--------------------------------------------------------------------------|--------------------------------+
                  |                                                                          |
                  | [1] Solicita Token                                                       | [1] Solicita Token
                  |                                                                          |
                  v                                                                          v
+-----------------------------------------------------------------------------------------------------------------------------+
|                                               ZONA DE SERVICIOS (MONSTERASP)                                                |
|                                                                                                                             |
|  +-----------------------------------------------------------------------------------------------------------------------+  |
|  |                                                Sitio de Autenticación                                                 |  |
|  |                                         [Identity Server / OAuth 2.0 API]                                             |  |
|  |                               Genera Bearer Tokens JWT tras validar credenciales.                                     |  |
|  +--------------------------------------------------------+--------------------------------------------------------------+  |
|                                                           |                                                                 |
|                                                           | [2] Lee/Escribe Usuarios                                        |
|                                                           v                                                                 |
|                                            +------------------------------+                                                 |
|                                            |   BD SQL Server (Auth)       |                                                 |
|                                            |   Esquema: Usuarios y Roles  |                                                 |
|                                            +------------------------------+                                                 |
|                                                                                                                             |
|                                                                                                                             |
|  [3] Peticiones HTTP Autenticadas (Bearer Token)                                                                            |
|  +-----------------------------------+------------------------------------+----------------------------+-----------------+  |
|  |                                   |                                    |                            |                 |  |
|  v                                   v                                    v                            v                 v  |
| +----------------------------+     +----------------------------+     +----------------------------+     +----------------+ |
| |     API Principal .NET     |     |     API Principal .NET     |     |      API de Reportes       |     |  API Storage   | |
| |    (Comandos - Write)      |     |    (Consultas - Read)      |     |     Generación de Docs     |     | (File Upload)  | |
| |  Entity Framework Core     |     |     Dapper / EF Core       |     |  Consultas Pesadas / OLAP  |     | Almacena docs  | |
| |  Mutaciones de Estado      |     |    Lecturas Optimizadas    |     |  Exportación de Datos      |     | e imágenes     | |
| +--------------+-------------+     +--------------+-------------+     +--------------+-------------+     +--------+-------+ |
|                |                                  |                                  |                            |         |
|                | [4] Escritura                    | [4] Lectura                      | [4] Lectura                | [4]     |
|                +-----------------+                |                                  |     de Datos               | Guarda  |
|                                  |                |                                  |                            v         |
|                                  v                v                                  v                    +----------------+ |
|                            +-----------------------------------------------------------+                  |   Sistema de   | |
|                            |                       BD SQL Server                       |                  |   Archivos /   | |
|                            |                       (Cambialtok)                        |                  |   Directorio   | |
|                            +-----------------------------------------------------------+                  +----------------+ |
+-----------------------------------------------------------------------------------------------------------------------------+

