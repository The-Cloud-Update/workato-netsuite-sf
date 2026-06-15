# Integración NetSuite ↔ Salesforce vía Workato
## Documento de Setup y Requerimientos

**Fecha:** Junio 2026  
**Propósito:** Guía de requerimientos y pasos para establecer la integración entre NetSuite y Salesforce utilizando Workato como middleware iPaaS.

---

## 1. Visión General

Workato actúa como capa de integración (iPaaS) entre NetSuite (ERP/financiero) y Salesforce (CRM/ventas). La integración más común es el flujo **Lead-to-Cash**: desde que un vendedor cierra un deal en Salesforce hasta que el pedido, factura y pago se registran en NetSuite, con visibilidad de vuelta en Salesforce.

```
Salesforce (CRM)  ←——  Workato Recipes  ——→  NetSuite (ERP)
  Oportunidades              ↕                Sales Orders
  Quotes                  Triggers           Invoices
  Accounts               Mappings            Customers
  Contacts               Error Handling      Payments
```

---

## 2. Prerrequisitos Generales

### 2.1 Licencias y Accesos Necesarios

| Sistema   | Requerimiento |
|-----------|--------------|
| Workato   | Suscripción activa con conectores NetSuite y Salesforce habilitados |
| NetSuite  | Cuenta con rol de Administrador para configuración inicial |
| Salesforce | Cuenta con perfil System Administrator |
| Ambos     | Entornos Sandbox para pruebas antes de producción |

### 2.2 Roles en el Equipo

| Rol | Responsabilidad |
|-----|----------------|
| Administrador NetSuite | Configura roles de integración, tokens y permisos |
| Administrador Salesforce | Configura el Connected App, perfiles y permisos de API |
| Desarrollador/Implementador Workato | Crea y configura las recipes |
| Analista de Negocio | Define el mapeo de campos y reglas de sincronización |

---

## 3. Setup en NetSuite

### 3.1 Habilitar Features de Integración

Navegar a: `Setup > Company > Enable Features > SuiteCloud`

Activar los siguientes checkboxes:

- [x] Client SuiteScript
- [x] Server SuiteScript
- [x] SOAP Web Services
- [x] REST Web Services
- [x] Token-Based Authentication (TBA)
- [x] OAuth 2.0

### 3.2 Crear un Integration Record

`Setup > Integration > Manage Integrations > New`

| Campo | Valor |
|-------|-------|
| Name | `Workato Integration` |
| State | `Enabled` |
| Token-Based Authentication | ✅ Activado |
| TBA Authorization Flow | ❌ Desactivado |
| Authorization Code Grant | ❌ Desactivado |

> **IMPORTANTE**: Guardar y copiar el **Consumer Key** y **Consumer Secret** inmediatamente. NetSuite solo los muestra una vez.

### 3.3 Crear el Rol de Integración

`Setup > Users/Roles > Manage Roles > New`

Asignar nivel **Full** a los siguientes permisos:

**Permisos de Setup:**
- Log in using Access Tokens
- Log in using OAuth 2.0 Access Tokens
- REST Web Services
- SOAP Web Services
- Set Up Company
- SuiteScript
- User Access Tokens
- Set Up SOAP Web Services

**Permisos de Listas:**
- Custom Body Fields
- Custom Address Form
- Deleted Records

**Permisos de Transacciones** (según objetos a sincronizar):

| Objeto | Nivel mínimo |
|--------|-------------|
| Sales Orders | View / Create / Edit |
| Invoices | View / Create |
| Customers | View / Create / Edit |
| Estimates/Quotes | View / Create |

### 3.4 Crear el Usuario de Integración

`Setup > Users/Roles > Manage Users`

- Crear usuario dedicado (ej. `workato-integration@empresa.com`)
- Asignarle el Rol de Integración creado en el paso 3.3
- **No usar una cuenta de usuario humano** — esto garantiza que el acceso no se interrumpa si alguien deja la empresa

### 3.5 Generar el Access Token

`Setup > Users/Roles > Access Tokens > New`

| Campo | Valor |
|-------|-------|
| Application Name | El Integration Record del paso 3.2 |
| User | El usuario de integración del paso 3.4 |
| Role | El rol de integración del paso 3.3 |

> **IMPORTANTE**: Guardar el **Token ID** y **Token Secret** inmediatamente. Solo se muestran una vez.

### 3.6 Obtener el Account ID de NetSuite

Navegar a: `Setup > Integration > SOAP Web Services Preferences`

- Formato producción: `1234567`
- Formato sandbox: `1234567_SB1`

---

## 4. Setup en Salesforce

### 4.1 Permisos del Usuario de Integración

Crear un usuario dedicado con un perfil que incluya:

| Permiso | Requerimiento |
|---------|--------------|
| API Enabled | ✅ Obligatorio |
| View All Data / Modify All Data | ✅ Recomendado para integración completa |
| Customize Application | ✅ Para triggers en tiempo real |
| Manage Data Integrations | ✅ Para operaciones bulk |
| View Setup and Configuration | ✅ Para operaciones bulk |
| Use Any API Client | ✅ Si el org tiene API Access Control habilitado |

### 4.2 Instalar el Connected App de Workato

**Opción A (Recomendada — vigente desde Sep 17, 2025):**  
Un Salesforce Admin instala el Workato Connected App desde AppExchange y lo autoriza en `Setup > Connected Apps > OAuth Usage`.

**Opción B:**  
Asignar el permiso `Approve Uninstalled Connected Apps` al usuario de integración.

> **Nota**: A partir del 17 de septiembre de 2025, Salesforce exige autorización explícita del Connected App para todas las nuevas conexiones.

### 4.3 Configurar la Conexión OAuth 2.0

Al crear la conexión en Workato:

1. Seleccionar tipo de auth: `OAuth 2.0 (Authorization Code Grant)`
2. Indicar si es Sandbox o Production
3. Configurar Custom Domain si el org usa My Domain (ej. `empresa.my.salesforce.com`)
4. Clic en **Connect** → autenticar con el usuario de integración

**Verificar que el Connected App NO tenga activado:**
- Require Proof Key for Code Exchange (PKCE) — Workato no soporta este flujo OAuth

### 4.4 Field-Level Security

Para cada objeto sincronizado (Account, Opportunity, Contact, Quote, etc.):
- Dar **visibilidad** a todos los campos que Workato leerá
- Dar **editar** a todos los campos que Workato escribirá

---

## 5. Setup en Workato

### 5.1 Crear la Conexión a NetSuite

`Workato > Connections > New Connection > NetSuite`

| Campo | Valor |
|-------|-------|
| Connection Name | `NetSuite Production` |
| Account ID | Obtenido en paso 3.6 |
| Token ID | Obtenido en paso 3.5 |
| Token Secret | Obtenido en paso 3.5 |
| Timezone | Timezone del org de NetSuite |
| Consumer Key | Obtenido en paso 3.2 (sección Advanced) |
| Consumer Secret | Obtenido en paso 3.2 (sección Advanced) |

### 5.2 Crear la Conexión a Salesforce

`Workato > Connections > New Connection > Salesforce`

| Campo | Valor |
|-------|-------|
| Connection Name | `Salesforce Production` |
| Auth Type | OAuth 2.0 |
| Environment | Production / Sandbox |
| Custom Domain | Si el org usa My Domain |

Clic en **Connect** y autenticar con el usuario de integración de Salesforce.

### 5.3 Verificar Conexiones

Ambas conexiones deben mostrar estado **Connected** (verde) antes de crear recipes.

---

## 6. Flujos de Sincronización y Recipes

### 6.1 Flujo Principal: Lead-to-Cash

```
[SF] Opportunity → CLOSED WON
         ↓ Trigger Workato
[NS] Buscar/Crear Customer
         ↓
[NS] Crear Sales Order con líneas de producto
         ↓
[SF] Actualizar Opportunity con NS Order ID
         ↓
[NS] Finance genera Invoice
         ↓ Trigger Workato
[SF] Crear/actualizar Invoice record en SF
         ↓
[NS] Registrar Payment
         ↓ Trigger Workato
[SF] Actualizar con estado de pago
```

### 6.2 Recipes del Acelerador Workato (Prebuilt)

| Recipe | Trigger | Acción |
|--------|---------|--------|
| Opportunity → Sales Order | Opp. Closed Won en SF | Crear Sales Order en NS |
| Quote → Estimate | Quote aprobado en SF | Crear Estimate en NS con shipping/tax |
| NS Invoice → SF | Invoice creado en NS | Crear/actualizar registro en SF |
| Customer Sync | New/Updated Account en SF | Crear/actualizar Customer en NS |
| Contact Sync | New/Updated Contact en SF | Sincronizar a NS |
| Payment Sync | Payment registrado en NS | Actualizar SF con estado de pago |
| Financial 360 | On-demand desde SF | Pull de invoices, balances, credit memos desde NS |

### 6.3 Mapeo de Campos Clave

| Salesforce | NetSuite | Notas |
|-----------|---------|-------|
| Account.Name | Customer.companyName | Sistema de record: SF |
| Account.Id | Customer.externalId | ID de SF almacenado como externo en NS |
| Opportunity.Name | SalesOrder.memo | |
| Opportunity.Amount | SalesOrder.total | Considerar impuestos y descuentos |
| OpportunityLineItem | SalesOrder.item[ ] | Requiere mapeo Product → NS Item |
| Contact.Email | Contact.email | |
| Quote.ExpirationDate | Estimate.dueDate | |
| Quote.TotalPrice | Estimate.total | |

---

## 7. Consideraciones de Arquitectura

### 7.1 Sistema de Record por Entidad

| Entidad | Sistema de Record | Dirección |
|---------|------------------|-----------|
| Accounts / Customers | Salesforce | SF → NS |
| Contacts | Salesforce | SF → NS |
| Products / Items | NetSuite | NS → SF |
| Precios / Price Books | NetSuite | NS → SF |
| Sales Orders | NetSuite | NS es fuente de verdad |
| Invoices | NetSuite | NS → SF (read-only en SF) |
| Payments | NetSuite | NS → SF (read-only en SF) |

### 7.2 Manejo de Errores

Configurar en Workato:

| Configuración | Recomendación |
|--------------|--------------|
| Error notifications | Email + Slack cuando una recipe falle |
| Retry logic | Auto-retry 3x para errores transitorios (timeout, rate limits) |
| Dead letter handling | Recipe separada para procesar registros fallidos |
| Job history | Habilitar audit trail y retener por mínimo 30 días |
| Alertas de volumen | Alerta si el job count cae inesperadamente |

### 7.3 Sincronización Inicial (Backfill)

Para la primera carga de datos históricos:
1. Exportar en bulk desde SF (Accounts, Contacts)
2. Procesar en lotes de 200-500 registros para respetar límites de API
3. Validar external IDs cruzados antes de activar recipes en tiempo real
4. Ejecutar en horario de bajo tráfico

---

## 8. Entornos y Estrategia de Deployment

### 8.1 Entornos

```
Desarrollo     →    Staging / UAT    →    Producción
NS Sandbox          NS Sandbox             NS Production
SF Sandbox          SF Full Sandbox        SF Production
Workato Dev         Workato Test           Workato Prod
```

### 8.2 Orden de Activación

1. Configurar conexiones en Sandbox de ambos sistemas
2. Construir y probar recipes en Workato Dev
3. UAT con datos reales de staging
4. Aprobación de stakeholders (Finance + Sales)
5. Go-live en producción con monitoreo intensivo la primera semana
6. Activar todas las recipes de sincronización bidireccional

---

## 9. Estimación de Esfuerzo

| Fase | Actividad | Estimado |
|------|-----------|---------|
| 1 | Setup técnico NS + SF + Workato | 2–3 días |
| 2 | Mapeo de campos y reglas de negocio | 3–5 días |
| 3 | Construcción de recipes en Workato | 5–10 días |
| 4 | Testing en sandbox (UAT) | 5–7 días |
| 5 | Backfill histórico y go-live | 2–3 días |
| **Total** | | **~3–5 semanas** |

---

## 10. Checklist de Setup

### NetSuite
- [ ] SuiteCloud features habilitados (Web Services, TBA, OAuth 2.0)
- [ ] Integration Record creado — Consumer Key y Secret guardados de forma segura
- [ ] Rol de integración creado con permisos correctos
- [ ] Usuario de integración dedicado creado con rol asignado
- [ ] Access Token generado — Token ID y Secret guardados de forma segura
- [ ] Account ID anotado

### Salesforce
- [ ] Usuario de integración con API Enabled
- [ ] Workato Connected App instalado y autorizado en el org
- [ ] Field-Level Security configurada para todos los objetos a sincronizar
- [ ] PKCE desactivado en el Connected App
- [ ] Sandbox disponible para pruebas

### Workato
- [ ] Conexión a NetSuite verificada (Connected)
- [ ] Conexión a Salesforce verificada (Connected)
- [ ] Acelerador NetSuite-Salesforce instalado (opcional pero recomendado)
- [ ] Recipes configuradas y probadas en sandbox
- [ ] Error notifications configuradas (Email + Slack)
- [ ] Monitoreo y alertas de jobs activadas

### Negocio / Datos
- [ ] Sistema de record definido por entidad
- [ ] Mapeo de campos documentado y aprobado por Finance y Sales
- [ ] Reglas de duplicados definidas
- [ ] Estrategia de backfill histórico aprobada
- [ ] Plan de pruebas UAT firmado

---

## 11. Recursos y Documentación Oficial

- [Workato Docs: NetSuite Connection Setup](https://docs.workato.com/connectors/netsuite/netsuite-connect.html)
- [Workato Docs: NetSuite REST Connector](https://docs.workato.com/connectors/netsuite-rest/connect.html)
- [Workato Docs: Salesforce Connector](https://docs.workato.com/connectors/salesforce.html)
- [Workato Docs: Salesforce Custom OAuth Profile](https://docs.workato.com/connectors/salesforce/custom-oauth.html)
- [Workato Accelerator: NetSuite-Salesforce](https://www.workato.com/accelerators/netsuite-salesforce-accelerator)
- [Guía de integración SF-NetSuite (Workato Blog)](https://www.workato.com/the-connector/salesforce-netsuite-integration-guide/)
