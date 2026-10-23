# QRListo

**Transferencias simples y avisos de cobro para pequeños comercios argentinos.**

> **Documento de concepto y producto (nombre provisorio)**  
> **Mercado inicial:** Argentina · kioscos, almacenes, autoservicios, ferias y comercios con empleados.  
> **Actualizado:** 23 de octubre de 2026  
> **Estado:** propuesta para validar; no es una integración bancaria ya disponible.

## Contenido

1. [La idea en una frase](#1-la-idea-en-una-frase)
2. [El problema real](#2-el-problema-real)
3. [Usuarios y participantes](#3-usuarios-y-participantes)
4. [Flujo completo deseado](#4-flujo-completo-deseado)
5. [Experiencia de pago del cliente](#5-la-vuelta-de-rosca-tres-experiencias-para-quien-paga)
6. [QR fijo y dinámico](#6-dos-modalidades-para-el-comercio)
7. [Avisos de cobro a empleados](#7-la-segunda-mitad-avisar-al-empleado-sin-abrir-el-banco-del-dueño)
8. [Estados y conciliación](#8-estados-y-lógica-de-conciliación)
9. [Pantallas y experiencia](#9-pantallas-y-experiencia-de-usuario)
10. [Alcance del MVP](#10-alcance-mvp-y-qué-dejar-fuera)
11. [Arquitectura](#11-arquitectura-propuesta)
12. [Riesgos](#12-riesgos-y-condicionantes)
13. [Validación real](#13-plan-de-validación-real-antes-de-construir-demasiado)
14. [Roadmap](#14-roadmap-por-etapas)
15. [Modelo de negocio](#15-modelo-de-negocio-hipótesis-no-precios-comprobados)
16. [Decisiones recomendadas](#16-decisiones-recomendadas)
17. [Fuentes oficiales](#17-fuentes-oficiales-consultadas-y-documentación-técnica)

---

## 1. La idea en una frase

**Permitir que cualquier cliente transfiera al banco o billetera que el comercio elija con la menor fricción posible y que el empleado vea, en tiempo real, qué cobros se detectaron o acreditaron, sin acceder al home banking del dueño.**

No se pretende crear un banco, una billetera ni custodiar fondos. La aplicación facilita el destino de la transferencia, organiza las ventas y escucha señales de ingreso de dinero desde conectores autorizados.

### Propuesta de valor

- **Para quien compra:** no escribir a mano CBU de 22 dígitos ni memorizar alias; sin cuenta en nuestra aplicación ni registro previo.
- **Para quien atiende:** consultar un panel con importe, hora, origen si está disponible y estado del cobro, sin necesitar el celular ni las claves del dueño.
- **Para el dueño:** recibir transferencias en su cuenta existente, elegir quién ve cada operación y consultar el historial aun estando lejos del local.
- **Para el negocio:** reducir errores y discusiones por comprobantes, agilizar la caja y mejorar la conciliación. **No es una herramienta para ocultar operaciones ni evitar impuestos.**

## 2. El problema real

### Momento A: pagar demora demasiado

Situación habitual: el cliente llega al kiosco, le dictan un alias o le muestran un CBU, abre su banco, carga el destinatario, verifica el nombre, ingresa el monto y confirma. Es lento y pueden producirse errores de tipeo.

**Pero un QR que solamente abre una página para copiar el alias puede sumar pasos.** Escanear → abrir web → copiar → cambiar de app → pegar → ingresar monto → confirmar no garantiza ser más rápido que conocer un alias corto. Por eso el criterio de diseño no es “tener un QR”, sino **reducir tiempo y fricción total**.

### Momento B: el vendedor no sabe si llegó el dinero

La cuenta está a nombre del dueño; la atención la hace un empleado, a veces de noche. El cliente muestra una captura, envía un comprobante o espera a que el titular conteste. Un comprobante o captura **no prueba por sí solo** que los fondos estén acreditados en la cuenta receptora. Los clientes se impacientan, el empleado queda expuesto y el dueño recibe consultas a toda hora.

### Momento C: distintos bancos y billeteras

No buscamos un producto exclusivo para Galicia, Nación, Santander, BBVA, Mercado Pago, MODO ni ningún proveedor concreto. **El banco emisor del cliente es libre y el receptor pertenece al comercio.** La compatibilidad de los mecanismos de cobro/aviso varía por entidad y se resuelve mediante conectores; nunca se presume soporte universal.

## 3. Usuarios y participantes

| Rol | Qué hace | Qué NO necesita |
|---|---|---|
| Cliente | Ve el destino y transfiere desde su banco/billetera preferidos | Registrarse ni instalar QRListo |
| Empleado | Crea o ve el cobro, recibe señales y consulta estados | Contraseña bancaria ni acceso al saldo del dueño |
| Dueño/administrador | Configura cuenta, permisos, sucursales y fuentes de aviso | Estar despierto para autorizar cada visualización |
| Conector | Recibe email, notificación o evento financiero y lo normaliza | Poder iniciar transferencias por cuenta del cliente |

## 4. Flujo completo deseado

```text
DUEÑO (una vez)
  Registra comercio → CBU/CVU y alias → valida titular y configura recepción de avisos
  → imprime QR / activa QR de caja → asigna empleados

CLIENTE (cada compra)
  Ve el QR → usa el camino más rápido compatible con su banco
  → verifica el destinatario y el importe → autoriza la transferencia EN SU BANCO

BANCO O BILLETERA RECEPTORA
  Registra/acredita ingreso → potencialmente envía un evento bancario,
  un correo o una notificación Android (según lo que soporte)

QRListo
  Ingresa evento → verifica fuente, evita duplicados, concilia cuando es posible
  → actualiza caja vía WebSocket/SSE

EMPLEADO
  Ve «esperando», «aviso detectado», «acreditado» o «requiere revisión»
  → entrega el pedido cuando el protocolo de caja permite considerar pagada la venta
```

## 5. La vuelta de rosca: tres experiencias para quien paga

### Nivel 1 — Objetivo ideal: escanear **desde la app bancaria** y confirmar

**Experiencia objetivo:** abrir la app bancaria o billetera → escanear un QR de transferencia reconocido → verificar destinatario e importe → confirmar. **Sin copiar ni pegar.**

Este camino tiene la mejor promesa de velocidad. MODO documenta que desde su app se puede transferir eligiendo CBU/CVU/alias **o QR**. El BCRA regula QR de pagos con transferencia y describe cuentas comerciales identificadas con QR, pero esto **no demuestra que un código inventado por QRListo se interprete como transferencia común sin comisión en todos los bancos**.

Hay dos cuestiones a validar de forma separada:

1. **Formato/compatibilidad:** qué QR de cuenta, personal o comercial reconoce cada app y qué datos precarga.
2. **Condiciones del canal:** si se trata de transferencia común o de **pago con transferencia (PCT)**, que puede tener comisión al comercio y otras condiciones.

**Estrategia inicial:** permitir vincular o exhibir un **QR de transferencia emitido oficialmente por el banco o billetera**, cuando las condiciones del proveedor lo admitan, y comprobar en pruebas reales quién lo lee. No se asegura interoperabilidad universal ni se usa un QR personal en un contexto comercial sin revisar las condiciones del proveedor.

### Nivel 2 — Evolución: botón «Transferir con mi banco» (integración autorizada)

**Experiencia objetivo:** escanear QR del comercio con la cámara → elegir app compatible → abrir la pantalla de transferencia con destino (y, si la API lo permite, monto) precargados → verificar y autorizar.

En Android existen App Links y en iOS Universal Links, **pero solamente el proveedor de la aplicación bancaria puede implementar/autorizar el manejo de enlaces y parámetros de transferencia**. No existe una URL pública universal confirmada capaz de abrir cualquier banco argentino con CBU y monto precargados.

**Regla:** incluir el botón únicamente para integraciones oficiales verificadas; nunca simular que un enlace genérico completará el formulario del banco. Abrir solo la pantalla principal de una billetera no representa necesariamente una mejora.

### Nivel 3 — MVP universal: QR web + alias de un toque

**Experiencia disponible sin acuerdo bancario:**

1. El cliente escanea el QR con la cámara.
2. Se abre una **web mínima**, sin instalación, publicidad, cuenta ni formularios: comercio, titular, alias y botón grande **«Copiar alias»** (opcional **«Copiar CBU/CVU»**).
3. Abre su banco habitual, pega el alias, **verifica el nombre del destinatario**, ingresa el monto y confirma.

**Ajustes para reducir fricción:**

- Un solo botón primario «Copiar alias» y el CBU/CVU en «Otras opciones».
- Alias corto y legible, de preferencia el propio del comercio (no se inventa otro sin control de titularidad).
- Mostrar una sola cuenta predeterminada; evitar una lista de bancos que obligue a decidir antes de pagar. Cambiar de cuenta solo si el negocio lo necesita.
- Guardar el alias en el portapapeles **al tocar el botón**; no prometer copia silenciosa al cargar la página porque los navegadores la limitan.
- Confirmación visual «Alias copiado» y una instrucción de una línea: «Abrí tu banco → Transferir → Pegá el alias».
- Página ultraliviana, HTTPS, buena legibilidad, sin login, sin cookies de seguimiento innecesarias y con soporte para internet móvil lento.
- En QR dinámico de caja: visualizar importe y nombre comercial **para que el cliente no los olvide**, aunque ese importe **no se completa automáticamente en el banco** sin integración real.
- Se pueden **evaluar** tarjetas NFC para abrir la misma página con un toque; esto elimina el paso de enfocar la cámara en algunos teléfonos, pero **no soluciona el paso de abrir el banco** ni es universal.

**Punto honesto:** esta alternativa reduce tipeo y errores, pero **no está demostrado que ahorre tiempo neto**. Debe competir en pruebas contra un cartel con alias grande o un QR nativo de transferencia emitido por una entidad.

### Comparación de caminos

| Camino | Acciones aproximadas del cliente | Requiere tercero | Estado |
|---|---|---|---|
| Alias escrito en cartel | Abrir banco → escribir alias → monto → confirmar | No | Referencia base |
| QR web + copiar alias | Cámara → copiar → banco → pegar → monto → confirmar | No | **Construible ya** |
| QR desde lector de banco | Banco → escanear → verificar monto/destino → confirmar | Compatibilidad del QR | **Validar por app y canal** |
| Enlace a transferencia precargada | Cámara → abrir app compatible → verificar → confirmar | App/banco con enlace oficial | **Investigación e integraciones** |
| QR propio que inicia transferencias desde cualquier banco | Un escaneo → confirmar | Infraestructura e interoperabilidad financiera | **No asumir que existe públicamente** |

> **Hipótesis central de UX:** el mejor recorrido no es «escaneo con cámara y copia», sino **escaneo dentro del banco/billetera cuando el formato funciona**. La web de copia es un respaldo universal, no el destino final.

## 6. Dos modalidades para el comercio

### QR fijo impreso

- El comercio registra una cuenta receptora y obtiene un adhesivo, cartel o pantalla con QR persistente.
- Escanearlo abre la información de esa cuenta y ofrece los caminos compatibles.
- **No conoce el importe** de cada compra. El cliente lo ingresa en su banco; la caja puede crear paralelamente un cobro pendiente para intentar conciliar.
- Es el modo más barato y accesible para kioscos sin sistema de caja.

### QR dinámico por venta

- El empleado introduce **$8.500** en su panel y toca «Cobrar».
- Se crea una venta con identificador propio y un QR/URL de **sesión**.
- El cliente ve **$8.500**, el comercio y el destino de transferencia. Si la integración lo admite, el monto se envía a la app; en caso contrario, debe ingresarlo manualmente.
- El panel queda esperando eventos y intenta asociar el ingreso a esa venta.
- **Advertencia:** el identificador de sesión en una URL web **no viaja automáticamente en una transferencia bancaria convencional**; no garantiza trazabilidad sin referencia efectiva reconocida por el banco o proveedor.

## 7. La segunda mitad: avisar al empleado sin abrir el banco del dueño

La plataforma recibe **señales** desde distintos conectores. El dueño los vincula en la configuración; el empleado ve únicamente los datos de cobro necesarios. El dueño no tiene que estar despierto, salvo si se usa una fuente que depende de su teléfono y ese teléfono está apagado o sin conexión.

| Fuente | Funcionamiento | Ventaja | Limitación determinante |
|---|---|---|---|
| Reenvío selectivo de email bancario | Filtro reenvía avisos a casilla específica; un servicio como Postmark los convierte en webhook | No se conecta el home banking; funciona con teléfono apagado | **El banco debe enviar mails útiles y oportunos**. Correo no verifica fondos |
| Gmail API + Pub/Sub | Con OAuth autorizado, `users.watch` avisa cambios; backend busca correos | Sin reenvío manual en algunos casos | Permisos, verificación de app, posibles retrasos y eventos perdidos |
| Microsoft Graph (Outlook) | Suscripción a mensajes por webhook | Misma arquitectura para Outlook | Permisos, renovación y retrasos; no prueba acreditación |
| Notificaciones Android | App compañera autorizada lee ciertos avisos con `NotificationListenerService` | Cobertura experimental de varias apps bancarias | Depende de móvil encendido/conectado, formatos y restricciones; no es prueba de fondos |
| API de movimientos bancaria | Conector autorizado consulta créditos acreditados | Mejor evidencia de ingreso | Acceso, contratos y capacidad específicos por entidad |
| Webhook de banco/PSP | Proveedor autorizado envía evento de transferencia acreditada | Mejor ruta a confirmación automatizada | No funciona para cualquier CBU sin convenio; costos y requisitos |

**Ejemplo documental de integración financiera, no banco obligatorio:** BIND PSP documenta webhooks de transferencias entrantes CBU y CVU, con `COMPLETED`, importe y datos de contraparte. Este tipo de fuente es el modelo deseable para una confirmación fuerte, pero se obtiene mediante integración con el proveedor.

### Por qué NO basaría el producto en un MCP que revise el mail

Un **MCP** podría exponer herramientas para que un agente consulte avisos o genere reportes. Sin embargo, la recepción crítica de cobros exige procesos **siempre activos**, reintentos, almacenamiento de eventos, permisos y monitoreo. Lo implementaría como **webhooks/APIs + cola de eventos + backend**, dejando MCP como extensión opcional de consultas y automatización.

### Precaución: emails y notificaciones pueden fallar o falsificarse

Un correo de un supuesto banco puede estar manipulado, un mensaje reenviado puede perder señales de autenticación de origen, y los avisos pueden llegar tarde. **No convertir jamás un correo o notificación en «acreditado confirmado» por sí solo**, aunque coincida el monto.

## 8. Estados y lógica de conciliación

Los mensajes al cajero deben ser inequívocos:

| Estado | Cuándo se usa | Qué puede hacer el empleado |
|---|---|---|
| **Esperando transferencia** | Venta creada, sin señales suficientes | Esperar o cobrar por otro medio |
| **Aviso detectado · no verificado** | Email o push informa un aparente ingreso | Revisar, pero **no tratarlo como acreditación comprobada** |
| **Crédito acreditado · sin asociar** | Evento financiero auténtico confirma ingreso, pero no se sabe de qué venta es | Resolver asignación sin atribuirlo arbitrariamente |
| **Pago acreditado y conciliado** | Fuente financiera autorizada confirmó ingreso y se vinculó inequívocamente a una venta | Entregar según política de caja |
| **Requiere revisión** | Duplicados, mismo importe simultáneo, falta de datos, eventos sospechosos, discrepancia | Pedir ayuda/revisar con criterio; no inventar confirmación |
| **Cancelado/expirado** | Abandono o timeout de la venta | Evitar que un aviso posterior valide otra venta por error |

**Nunca** asociar dos ventas a un mismo movimiento. En particular, dos clientes que pagan $5.000 al mismo tiempo pueden producir una coincidencia de importe que **no identifica al pagador**. Se necesitan identificadores de movimiento, referencias bancarias si existen, destinatario, ventana temporal y origen, con reglas conservadoras. No se debe inventar una referencia de transferencia que la banca no transmite. Tampoco conviene usar centavos arbitrarios para “identificar” clientes como solución universal.

El comprobante del cliente puede ayudar a investigar, **pero no reemplaza** la consulta de cuenta o un evento verificable.

### Seguridad mínima de eventos

- Validar autenticidad de webhook y permisos del conector.
- Manejar **idempotencia**: varios avisos del mismo movimiento crean un solo crédito.
- Reintentos y recuperación de eventos faltantes, con auditoría de estado.
- Separar la recepción del evento financiero del evento interno «venta conciliada».
- Guardar trazabilidad de *origen del dato*, *hora del banco*, *hora de recepción* y *hora de notificación a caja*.
- Permisos por rol y comercio; no revelar saldo, claves bancarias o movimientos ajenos a empleados.

## 9. Pantallas y experiencia de usuario

### Cliente: página de un QR web

```text
┌───────────────────────────────────┐
│ KIOSCO DON PEDRO                 │
│ Transferí desde tu banco         │
│                                   │
│   Total informado: $8.500        │ ← solo si lo cargó la caja
│                                   │
│   Alias: kiosco.pedro.ventas      │
│                                   │
│   [  COPIAR ALIAS  ]             │ ← botón principal
│   [ Ver CBU / CVU ]               │
│                                   │
│ Abrí tu banco → Transferir → Pegá│
│ Verificá titular antes de pagar.  │
└───────────────────────────────────┘
```

Si existe una integración oficial probada, sustituir el botón principal por **«Transferir con [app compatible]»**, con copia de alias como respaldo. **Nunca mostrar un botón que simule ser el de una entidad bancaria.**

### Empleado: panel de caja

```text
┌─────────────────────────────────────────┐
│ Caja 1 · Kiosco Don Pedro              │
│ Nuevo cobro  [ $8.500 ]  [ Generar QR ] │
├─────────────────────────────────────────┤
│ Venta #104 · $8.500  🟠 Aviso detectado │
│ Venta #103 · $3.200  🟢 Acreditado      │
│ Venta #102 · $5.000  ⚪ Revisión        │
├─────────────────────────────────────────┤
│ Conector email: ACTIVO                  │
│ API financiera: NO CONFIGURADA         │
│ Último evento: 03:12:07                │
└─────────────────────────────────────────┘
```

Los colores son auxiliares: **todo estado siempre se escribe con texto**. Opcionales: alerta sonora cuando cambia a «acreditado» y una alerta distinta al llegar solo un aviso no verificado.

### Dueño: configuración

- Selecciona cuenta receptora (verificación de formato, titular y autorización; no basta verificar los 22 dígitos del CBU).
- Configura comercio/sucursal/caja y empleados con permisos limitados.
- Habilita conectores disponibles (email, Android opcional, API/webhook si tiene acceso).
- Prueba recepción de evento y ve su nivel de confianza.
- Consulta historial, filtros y conciliaciones pendientes; revoca acceso cuando quiera.

## 10. Alcance MVP y qué dejar fuera

### MVP de aprendizaje — sin falsas promesas de confirmación

1. Alta de comercio y cuenta; roles dueño/empleado.
2. QR web fijo a página pública que ofrece alias y CBU/CVU con **copiar en un toque**.
3. Modo cobro con monto mostrado y venta pendiente.
4. Panel de caja actualizado por WebSocket o SSE.
5. **Un conector de email autorizado**, preferiblemente con reenvío selectivo a Postmark/servicio equivalente, que cree eventos **«avisos detectados»**.
6. Historial, deduplicación, auditoría, desconexión y estado visible del conector.
7. Pruebas comparativas contra cartel de alias y QR bancario oficial.

**No anunciar en este MVP “acreditación bancaria automática verificada” si solo leemos correo o notificaciones.**

### MVP comercial para prometer «confirmado»

Requiere al menos **un conector financiero contratado y probado** que confirme créditos realmente asentados en una cuenta admitida. Ofrecer la función únicamente en comercios/cuentas compatibles, con monitoreo y procedimiento ante demoras.

### Fuera de alcance inicial

- Custodiar o mover dinero por cuenta del cliente.
- Solicitar claves, tokens o acceso al home banking del dueño.
- Transferencias automáticas sin autorización del titular pagador.
- Soporte supuesto para todos los bancos.
- Facturación fiscal automática y eliminación de impuestos.
- IA para «certificar» capturas de comprobantes como si fueran confirmación bancaria.
- Convertir cualquier QR de Mercado Pago en un CBU: puede no exponerlo (el dato es opcional según norma y formato).

## 11. Arquitectura propuesta

```text
             CLIENTE                                      COMERCIO
  QR fijo / QR dinámico / enlace                      Dueño / Empleado
              │                                           │
              ▼                                           ▼
     Web pública de pago                         Panel React + TypeScript
     (sin login del cliente)                               │
              │                                           │ WebSocket/SSE
              └───────────┐                 ┌─────────────┘
                          ▼                 ▼
                       API FastAPI (HTTPS)
                    Comercios · Cuentas · Ventas
                     Conciliación · Permisos
                              │
                       PostgreSQL + cola
                              ▲
               ┌──────────────┼────────────────┐
               │              │                │
         Email webhook     Android listener   API/Webhook PSP
         Gmail/Outlook      (opcional)        (verificable)
```

**Stack sugerido:** frontend React + TypeScript como PWA; backend Python + FastAPI; PostgreSQL; cola de trabajos (Redis + worker, o infraestructura equivalente); entrega a caja por SSE/WebSocket; lectores/generadores de QR con bibliotecas revisadas. Integraciones con contratos/adaptadores (`EmailEventAdapter`, `AndroidNotificationAdapter`, `BankWebhookAdapter`), sin mezclar sus niveles de confianza.

### Modelo mínimo de datos

- `merchants`, `branches`, `users`, `roles`.
- `receiving_accounts`: entidad, tipo, CBU/CVU, alias, titular, fecha de comprobación, vigencia.
- `payment_sessions`: monto, moneda, caja, creación, expiración, estado, referencia local.
- `source_connections`: conector, autorización, salud y última sincronización.
- `incoming_events`: fuente, identidad verificable del evento, momento, importe, datos disponibles, estado de autenticidad.
- `credits`: créditos confirmados por fuente financiera, id externo y cuenta destino.
- `reconciliations`: reglas aplicadas, venta asociada, certeza y revisión humana.
- `audit_log`: accesos/cambios/operaciones sensibles.

No registrar por defecto más datos personales de remitentes que los necesarios para conciliar. Cifrar información sensible en tránsito y reposo, y definir plazos de conservación acordes con fines legítimos y obligaciones aplicables.

## 12. Riesgos y condicionantes

| Riesgo | Mitigación |
|---|---|
| QR web más lento que alias escrito | Medir tiempos; priorizar QR de transferencia leído por banco compatible; mantener copia como respaldo |
| No existe universalidad de links bancarios | Matriz real de capacidades por app, OS y versión; solo integraciones oficiales |
| QR de cobro comercial vs transferencia común | Distinguir rieles, costos, términos y tratamiento contractual/fiscal; no venderlos como equivalentes |
| Banco no manda emails o llegan tarde | Mostrar salud del conector; no prometer SLA en método email; fallback operativo |
| Email spoofed o aviso Android falsificado | Clasificación «aviso no verificado»; protección de entrada; confirmación fuerte solo financiera |
| Dos pagos del mismo monto | No conciliar por monto solamente; referencias reales/ID externo o revisión |
| Empleado cobra en cuenta errónea | Mostrar nombre de comercio/titular; verificar destinatario en banco; controles de cambio de cuenta y antisuplantación de QR impresos |
| Alguien reemplaza el adhesivo QR | Dominio reconocible, controles periódicos del cartel, confirmación de destinatario y bloqueo/reimpresión ante sospecha |
| Pérdida de conectividad | Reintentos, colas y backfill cuando sea posible, estados honestos y procedimiento manual |
| Dependencia de integraciones privadas | Ofrecer fuentes compatibles por contrato y no prometer cobertura indiscriminada |
| Privacidad y regulación | Consentimiento claro, alcance mínimo, control de acceso, ley argentina de datos personales; revisión legal/regulatoria antes del despliegue |

**Fiscal:** las ventas siguen teniendo las obligaciones de registración, facturación e impuestos que correspondan. Cobrar por transferencia común no hace la venta “invisible”; comisiones, retenciones y reglas de cuentas deben analizarse por separado con asesoramiento contable. No se diseña para eludir obligaciones.

## 13. Plan de validación real (antes de construir demasiado)

### Experimento 1 — Fricción de compra (el más importante)

Reclutar compradores reales y comparar, con distintas apps bancarias y Android/iPhone:

- A. Alias grande y legible en cartel (línea base).
- B. QR web + copiar alias.
- C. QR de transferencia emitido por banco/billetera receptora, leído desde app compatible.
- D. Enlace que abre una app **si esa app admite oficialmente datos precargados**.

**Métricas:** segundos desde «quiero pagar» hasta confirmación en la app del cliente; cantidad de toques, cambio de aplicaciones, errores, abandono, dificultad percibida. **Criterio de decisión:** la alternativa elegida debe superar consistentemente a A, no limitarse a “tener QR”. Registrar compatibilidad exacta por app/versión/OS y si la operación es transferencia común o PCT.

### Experimento 2 — Detección de ingresos

Reclutar comercios con diferentes bancos receptores. Por cada cuenta, probar transferencias reales y autorizadas desde varios bancos emisores, horarios y montos:

- ¿El banco envía email por **cada** ingreso? ¿Hay importe, hora, cuenta, remitente e ID?
- ¿La notificación Android llega con el teléfono bloqueado, sin uso y en ahorro de batería?
- ¿Cuánto tarda desde la acreditación hasta el aviso recibido y hasta la visualización en caja?
- ¿Se pierde algún evento? ¿Qué sucede con redes inestables?
- ¿Qué proporción de eventos permite asociar una venta **sin ambigüedad**?
- ¿El banco/PSP ofrece contrato/API para créditos verificados?

**Métricas:** mediana, p95 y p99 de latencia por fuente; tasa de eventos recibidos; falsos positivos; créditos no conciliados; disponibilidad del conector. **No establecer un SLA público sin datos de campo.**

### Experimento 3 — Kiosco 24 horas

Simular empleado de noche, dueño ausente, dos compradores pagando igual monto, un email demorado y celular del dueño apagado. El sistema **debe diferenciar con claridad aviso recibido de dinero acreditado** y ofrecer un procedimiento de revisión, sin inventar confirmaciones.

### Condiciones para avanzar al producto de pago

- La experiencia de compra elegida mejora o al menos empata la velocidad de referencia sin errores nuevos.
- Existe al menos una fuente de acreditación verificable para la función «Pago acreditado» de los comercios que la contraten.
- Los eventos duplicados no duplican créditos y los ambiguos nunca se asignan automáticamente.
- El dueño puede configurar permisos para que un empleado use la caja sin conocer credenciales bancarias.
- Los costos reales de infraestructura y adquisición justifican el precio que el comercio está dispuesto a pagar.

## 14. Roadmap por etapas

| Etapa | Entregable | Qué valida |
|---|---|---|
| 0. Descubrimiento | Entrevistas + matriz de bancos/apps/QR + prueba de tiempos | Qué fricción se puede eliminar de verdad |
| 1. Prototipo UX | QR web, copia, QR dinámico ilustrativo, pruebas móviles | Que el cliente complete rápido la transferencia |
| 2. MVP de avisos | Comercios, roles, caja en vivo, reenvío de emails, estado «detectado» | Utilidad para empleados sin acceso bancario |
| 3. Verificación real | Al menos un webhook/API contratado, créditos autenticados y conciliación conservadora | Promesa comprobable de «acreditado» |
| 4. Compatibilidad | QR nativo de transferencia documentado, enlaces oficiales por proveedor, conectores adicionales | Menos pasos y más cobertura |
| 5. SaaS escalable | Multisucursal, reportes, facturación y soporte (si se define y regula) | Retención y disposición a pagar |

## 15. Modelo de negocio (hipótesis, no precios comprobados)

- **Plan inicial:** generación de QR, alias fácilmente copiable, panel de caja y avisos **no verificados** donde el banco los emita.
- **Plan avanzado:** conectores financieros autorizados, acreditaciones confirmadas, conciliación e historial multicaja.
- **Posibles clientes:** kioscos 24 horas, autoservicios, panaderías, carnicerías, ferias y locales donde el titular no está siempre en la caja.
- **Pricing a validar:** suscripción por comercio/sucursal o caja activa; evitar comisión por transferencia si no procesamos fondos y el modelo lo permite. Evaluar costos de webhooks, email, soporte y verificación.

### Diferenciación defendible

El QR y «copiar alias» por sí solos son fáciles de replicar y aportan valor limitado. **El diferencial es unificar la experiencia de transferencia con un panel multiempleado y una clasificación honesta de avisos y acreditaciones**, extendible a diferentes bancos sin exigir al empleado acceso a la cuenta del dueño.

## 16. Decisiones recomendadas

1. **Prioridad n.º 1: probar QR de transferencia nativos** desde MODO y varias apps bancarias, junto con sus condiciones de uso. Es la opción más prometedora para bajar realmente los clics.
2. **Prioridad n.º 2: lanzar el respaldo universal** con QR a una web liviana y botón «Copiar alias», pero validar si es mejor que un cartel legible.
3. **Prioridad n.º 3: recibir avisos** con email/reenvío o Android, **etiquetados como no verificados** y con medición de tiempos.
4. **Prioridad n.º 4: confirmar créditos de verdad** mediante conector autorizado para clientes elegibles y conciliar conservadoramente.
5. **Nunca prometer** «funciona en cualquier banco sin integración», «copia automáticamente al abrir», «QR siempre sin comisión», «notificación bancaria = pago acreditado» ni ventajas fiscales automáticas.

## 17. Fuentes oficiales consultadas y documentación técnica

Fuentes de referencia — las capacidades operativas y condiciones comerciales deben revisarse al implementar:

1. **BCRA — Comunicación A 7919:** especificaciones y posición 51 (CBU/CVU/alias opcional en QR regulados): https://www.bcra.gob.ar/Pdfs/comytexord/A7919.pdf
2. **BCRA — QR para pagos con transferencia:** https://www.bcra.gob.ar/noticias/se-crea-un-codigo-qr-para-simplificar-el-pago-por-transferencia/
3. **BCRA — Transferencias 3.0 / QR comercial:** https://www.bcra.gob.ar/noticias/transferencia-con-qr/
4. **BCRA — Alias CBU:** https://www.bcra.gob.ar/alias-cbu/
5. **MODO — Transferencias por QR/alias:** https://www.modo.com.ar/ayuda/preguntas-frecuentes/C%C3%B3mo-transferir-dinero
6. **Android — App Links:** https://developer.android.com/training/app-links/about
7. **Apple — Universal Links:** https://developer.apple.com/documentation/xcode/supporting-universal-links-in-your-app
8. **MDN — Clipboard API `writeText`:** https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/writeText
9. **Gmail — Push notifications:** https://developers.google.com/workspace/gmail/api/guides/push?hl=es-419
10. **Microsoft Graph — Notificaciones Outlook:** https://learn.microsoft.com/en-us/graph/outlook-change-notifications-overview
11. **Postmark — Inbound email parsing:** https://www.postmarkapp.com/developer/user-guide/inbound/parse-an-email
12. **Android — NotificationListenerService:** https://developer.android.com/reference/android/service/notification/NotificationListenerService.html
13. **BIND PSP — webhook transferencia entrante CBU:** https://psp.bind.com.ar/developers/apis/transfer-webhookentrantecbu
14. **BIND PSP — webhook transferencia entrante CVU:** https://psp.bind.com.ar/developers/apis/transfer-webhookentrantecvu
15. **ARCA — régimen de facturación / comprobantes:** https://www.arca.gob.ar/facturacion/regimen-general/comprobantes.asp

---

**Visión final:** *«Escaneá, transferí desde tu banco y seguí tu compra. El kiosco recibe la señal de cobro en caja sin pedirle al dueño que revise el teléfono».* Esta visión se implementa gradualmente: **primero simplificación del pago, luego avisos, finalmente acreditación verificada en los bancos/proveedores que lo permitan**.