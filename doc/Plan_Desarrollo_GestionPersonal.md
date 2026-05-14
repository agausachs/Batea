# Plan de Desarrollo
## Aplicación Web de Gestión de Necesidades de Personal

**.NET 8 · ASP.NET MVC · Clean Architecture · MediatR · AutoMapper · FluentValidation · EF Core · SQL Server**

| Campo | Valor |
|---|---|
| Versión | 1.0 |
| Fecha | Mayo 2026 |
| Stack | .NET 8 · ASP.NET MVC · Clean Architecture · MediatR · AutoMapper · FluentValidation · EF Core · SQL Server · ASP.NET Identity |
| Roles | Producción · RRHH · Nóminas · Supervisor |

---

## 1. Introducción y Alcance

Este documento define el plan de desarrollo completo para la implementación de la aplicación web de Gestión de Necesidades de Personal. El objetivo es proporcionar a la IA programadora todas las instrucciones necesarias para construir la solución desde cero, cubriendo arquitectura, modelos de datos, lógica de negocio, interfaz de usuario y configuración de despliegue.

### 1.1 Objetivos del Sistema

- Gestionar el ciclo de vida completo de una petición de personal, desde la detección de la necesidad hasta el inicio efectivo del nuevo empleado.
- Separar responsabilidades entre los tres departamentos involucrados: Producción, RRHH y Nóminas.
- Mantener trazabilidad completa de cada petición con auditoría de todas las acciones.
- Soportar múltiples contratas con sus propios centros de trabajo y usuarios.
- Proporcionar visibilidad de solo lectura a usuarios supervisores.

### 1.2 Roles del Sistema

| Rol | Nombre en Identity | Responsabilidades |
|---|---|---|
| Producción | `Production` | Crear y gestionar peticiones de personal. Ver el estado de sus peticiones. |
| RRHH | `HR` | Gestionar candidatos, revisiones médicas y asignación a peticiones. |
| Nóminas | `Payroll` | Establecer fecha prevista, registrar Alta SAP, Alta SS, firma de contrato o anulación. |
| Supervisor | `Supervisor` | Vista de solo lectura de todas las peticiones y su estado para las contratas asignadas. |

---

## 2. Arquitectura de la Solución

### 2.1 Estructura de Proyectos (Clean Architecture)

La solución se organiza en cuatro capas según los principios de Clean Architecture. El nombre de la solución será `PersonnelManagement`.

| Proyecto / Capa | Descripción |
|---|---|
| `PersonnelManagement.Domain` | Entidades, enumeraciones, value objects, eventos de dominio, interfaces de repositorio e interfaces base. No tiene dependencias externas. |
| `PersonnelManagement.Application` | Casos de uso con MediatR (Commands/Queries/Handlers), DTOs, validadores FluentValidation, perfiles AutoMapper, interfaces de servicios externos. |
| `PersonnelManagement.Infrastructure` | Implementaciones EF Core (DbContext, repositorios, Unit of Work), migraciones, servicios de correo, publicación de Domain Events. |
| `PersonnelManagement.Web` | Proyecto ASP.NET MVC 5. Controllers, Views (Razor), ViewModels, filtros, middleware, configuración DI, ASP.NET Identity. |

### 2.2 Patrones y Librerías

| Librería / Patrón | Uso |
|---|---|
| MediatR | Desacoplamiento entre Controllers y Application. Todos los casos de uso son Commands o Queries. Pipeline behaviors para validación y logging. |
| AutoMapper | Mapeo bidireccional entre Entities, DTOs y ViewModels. Perfiles separados por feature. |
| FluentValidation | Validación de todos los Commands y FormViewModels. Integrado en el pipeline de MediatR mediante `ValidationBehavior`. |
| EF Core | ORM principal. Code-First con migraciones. Dos DbContext: `ApplicationDbContext` (negocio) e `IdentityDbContext` (usuarios). |
| ASP.NET Identity | Autenticación y gestión de usuarios en esquema separado (`dbo_identity`) dentro del mismo SQL Server. Quitar el prefijo AspNet de los nombres de las tablas. |
| Serilog | Logging estructurado a fichero y consola. |
| Domain Events | Publicación interna de eventos al completar transacciones (MediatR `INotification`). |

### 2.3 Esquemas de Base de Datos

- Esquema `dbo`: todas las tablas de negocio de la aplicación.
- Esquema `identity`: tablas de ASP.NET Identity (`AspNetUsers`, `AspNetRoles`, etc.).
- Ambos esquemas en la misma base de datos SQL Server, configurables vía connection string.

---

## 3. Modelo de Dominio

### 3.1 Entidad Base — `FullAuditableBaseEntity`

Todas las entidades de negocio heredan de esta clase base:

```csharp
public abstract class FullAuditableBaseEntity
{
    public int Id { get; set; }
    public DateTime Created { get; set; }
    public string CreatedBy { get; set; }
    public DateTime? LastModified { get; set; }
    public string LastModifiedBy { get; set; }
    public DateTime? Deleted { get; set; }
    public string DeletedBy { get; set; }
    public bool IsDeleted => Deleted.HasValue;

    private readonly List<INotification> _domainEvents = new();
    public IReadOnlyCollection<INotification> DomainEvents => _domainEvents.AsReadOnly();
    public void AddDomainEvent(INotification e) => _domainEvents.Add(e);
    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

> El borrado es **siempre lógico** (soft delete). El DbContext aplica un global query filter `IsDeleted == false` en todas las entidades.

### 3.2 Enumeraciones del Dominio

#### `WorkShift` (Turno)
| Valor | Nombre | Descripción |
|---|---|---|
| 1 | `Morning` | Mañana |
| 2 | `Afternoon` | Tarde |
| 3 | `Night` | Noche |

#### `WorkRegime` (Correturno / Régimen)
| Valor | Nombre | Descripción |
|---|---|---|
| 1 | `FiveTwo` | 5/2 |
| 2 | `SixOne` | 6/1 |
| 3 | `SixZero` | 6/0 |
| 4 | `MondayFriday` | L-V |
| 99 | `Other` | Otro (campo de texto libre adicional) |

#### `RequestReason` (Motivo de la Petición)
| Valor | Nombre | Descripción |
|---|---|---|
| 1 | `IT` | Incapacidad Temporal |
| 2 | `Leave` | Permiso |
| 3 | `Vacation` | Vacaciones |
| 4 | `Extra` | Punta de trabajo / Nueva plaza |

#### `RequestStatus` (Estado Global de la Petición)
| Valor | Nombre | Descripción |
|---|---|---|
| 0 | `Draft` | Petición incompleta, aún editando |
| 1 | `Submitted` | Enviada, pendiente de RRHH |
| 2 | `CandidateAssigned` | RRHH ha asignado candidato |
| 3 | `MedicalReviewPending` | Pendiente de revisión médica |
| 4 | `MedicalReviewPassed` | Revisión médica apta |
| 5 | `MedicalReviewFailed` | Revisión médica no apta (candidato descartado) |
| 6 | `PayrollPending` | En manos de Nóminas |
| 7 | `HighDateSet` | Fecha prevista de alta establecida |
| 8 | `SAPHighDone` | Alta en SAP completada |
| 9 | `SSHighDone` | Alta en SS completada |
| 10 | `ContractSigned` | Contrato firmado, petición cerrada con éxito |
| 11 | `NoShow` | Candidato no se presenta |
| 12 | `SAPHighCancelled` | Anulación de alta SAP en curso |
| 13 | `SSHighCancelled` | Anulación de alta SS en curso |
| 14 | `Cancelled` | Petición anulada completamente |

#### `CandidateStatus` (Estado del Candidato)
| Valor | Nombre | Descripción |
|---|---|---|
| 1 | `Proposed` | Propuesto por RRHH |
| 2 | `MedicalReviewRequired` | Necesita revisión médica |
| 3 | `MedicalReviewPassed` | Apto |
| 4 | `MedicalReviewFailed` | No apto (descartado) |
| 5 | `Active` | Asignado y activo en la petición |
| 6 | `Discarded` | Descartado |

### 3.3 Entidades Principales

#### `Contrata`

| Campo | Tipo | Descripción |
|---|---|---|
| `Id` | `int PK` | Clave primaria |
| `Name` | `string(200)` | Nombre comercial de la contrata |
| `Code` | `string(20)` | Código corto (único) |
| `StartDate` | `DateTime` | Fecha de inicio del contrato |
| `EndDate` | `DateTime?` | Fecha de fin (`null` = indefinido). Si tiene fecha y es pasada, la contrata se considera inactiva. |
| `IsActive` *(computed)* | `bool` | `true` si `EndDate` es null o futura |
| `WorkCenters` | `ICollection<WorkCenter>` | Centros de trabajo de esta contrata |

#### `WorkCenter` (Centro de Trabajo)

| Campo | Tipo | Descripción |
|---|---|---|
| `Id` | `int PK` | Clave primaria |
| `ContrataId` | `int FK` | Contrata a la que pertenece |
| `Code` | `string(10)` | Código del centro (JM, PC, SP, FT, JC, CO, AB, CA…) |
| `Name` | `string(200)` | Nombre descriptivo del centro |

#### `StaffRequest` (Petición de Personal)

| Campo | Tipo | Descripción |
|---|---|---|
| `Id` | `int PK` | |
| `RequestNumber` | `string(20)` | Número autogenerado (ej: `PET-2026-0001`). Se asigna al pasar a `Submitted`. |
| `RequestDate` | `DateTime?` | Fecha de petición. Se asigna automáticamente al pasar a `Submitted`. |
| `ContrataId` | `int FK` | Contrata solicitante |
| `WorkCenterId` | `int FK` | Centro de trabajo donde cubrir la vacante |
| `WorkShift` | `enum WorkShift` | Turno requerido |
| `WorkRegime` | `enum WorkRegime` | Régimen / correturno |
| `WorkRegimeOther` | `string?` | Descripción libre si `WorkRegime = Other` |
| `Reason` | `enum RequestReason` | Motivo de la necesidad |
| `Status` | `enum RequestStatus` | Estado actual de la petición |
| `Notes` | `string?` | Notas adicionales de Producción |
| `RequestItems` | `ICollection<RequestItem>` | Personas a cubrir |
| `Candidates` | `ICollection<RequestCandidate>` | Candidatos propuestos por RRHH |
| `ExpectedStartDate` | `DateTime?` | Fecha de alta prevista (la establece Nóminas) |
| `SAPHighDate` | `DateTime?` | Fecha real del alta en SAP |
| `SSHighDate` | `DateTime?` | Fecha real del alta en SS |
| `ContractSignedDate` | `DateTime?` | Fecha de firma del contrato |
| `NoShowDate` | `DateTime?` | Fecha en que se confirmó no presentación |
| `SAPHighCancelledDate` | `DateTime?` | Fecha de anulación del alta SAP |
| `SSHighCancelledDate` | `DateTime?` | Fecha de anulación del alta SS |

#### `RequestItem` (Persona a cubrir dentro de la petición)

Una petición puede cubrir N personas (p.ej. múltiples vacaciones simultáneas). Cada `RequestItem` representa una persona cubierta.

| Campo | Tipo | Descripción |
|---|---|---|
| `Id` | `int PK` | |
| `StaffRequestId` | `int FK` | Petición a la que pertenece |
| `EmployeeId` | `string(50)?` | ID del trabajador en SAP/PAI/Dorlet. `null` si es nueva plaza. |
| `EmployeeName` | `string(200)?` | Nombre y apellidos (predeterminados del sistema externo). `null` si es nueva plaza. |
| `StartDate` | `DateTime` | Fecha de inicio de la cobertura / incorporación deseada |
| `EndDate` | `DateTime?` | Fecha de fin de la cobertura. `null` si es nueva plaza o se desconoce. |
| `IsNewPosition` | `bool` | `true` si es una nueva plaza (sin trabajador a cubrir) |

#### `RequestCandidate` (Candidato propuesto por RRHH)

| Campo | Tipo | Descripción |
|---|---|---|
| `Id` | `int PK` | |
| `StaffRequestId` | `int FK` | Petición asociada |
| `FullName` | `string(300)` | Nombre completo del candidato |
| `DocumentId` | `string(20)?` | DNI/NIE del candidato |
| `HasPreviousContract` | `bool` | ¿Ha trabajado antes en la empresa? |
| `LastMedicalReviewDate` | `DateTime?` | Fecha de la última revisión médica (si la tiene) |
| `MedicalReviewRequired` | `bool` | Calculado: `true` si no tiene RM vigente |
| `MedicalReviewDate` | `DateTime?` | Fecha en que se realizó la RM en este proceso |
| `MedicalReviewResult` | `bool?` | `null` = pendiente, `true` = apto, `false` = no apto |
| `Status` | `enum CandidateStatus` | Estado del candidato en el proceso |
| `Notes` | `string?` | Notas de RRHH sobre el candidato |
| `IsSelected` | `bool` | `true` = es el candidato finalmente elegido para la petición |

#### `MedicalReviewValidityConfig` (Configuración)

| Campo | Tipo | Descripción |
|---|---|---|
| `Id` | `int PK` | |
| `ValidityMonths` | `int` | Meses de validez de la revisión médica. Default: 12. Configurable por un administrador. |
| `UpdatedBy` | `string` | Usuario que actualizó el parámetro |
| `UpdatedAt` | `DateTime` | Fecha de la última actualización |

#### `ApplicationUser` (extiende `IdentityUser`)

| Campo | Tipo | Descripción |
|---|---|---|
| `FullName` | `string(300)` | Nombre completo del usuario |
| `IsActive` | `bool` | Activo/inactivo |
| `UserContratas` | `ICollection<UserContrata>` | Contratas asignadas al usuario |

#### `UserContrata` (tabla de relación M:N)

| Campo | Tipo | Descripción |
|---|---|---|
| `UserId` | `string FK` | FK hacia `ApplicationUser` |
| `ContrataId` | `int FK` | FK hacia `Contrata` |
| `AssignedDate` | `DateTime` | Fecha de asignación |

#### `RequestStatusHistory` (Historial de cambios de estado)

| Campo | Tipo | Descripción |
|---|---|---|
| `Id` | `int PK` | |
| `StaffRequestId` | `int FK` | Petición |
| `FromStatus` | `enum RequestStatus?` | Estado origen |
| `ToStatus` | `enum RequestStatus` | Estado destino |
| `ChangedAt` | `DateTime` | Fecha y hora del cambio |
| `ChangedBy` | `string` | Usuario que realizó el cambio |
| `Comment` | `string?` | Comentario opcional del cambio |

---

## 4. Flujo de Negocio Detallado

### 4.1 Paso 1 — Producción crea una petición

El usuario con rol Producción, asignado a una o varias contratas, accede al formulario de nueva petición.

#### Campos del formulario

| Campo | Obligatorio | Reglas |
|---|---|---|
| Contrata | Sí | Desplegable filtrado por las contratas del usuario. Solo contratas activas. |
| Centro de Trabajo | Sí | Desplegable dependiente de la contrata seleccionada. |
| Turno | Sí | Selección entre Mañana, Tarde, Noche. |
| Correturno / Régimen | Sí | Selección entre 5/2, 6/1, 6/0, L-V, Otro. Si "Otro", mostrar campo de texto adicional. |
| Motivo | Sí | Selección: IT, Permiso, Vacaciones, Punta/Nueva plaza. |
| Personas a cubrir (`RequestItems`) | Sí (al menos 1) | Ver reglas por motivo a continuación. |
| Notas | No | Texto libre opcional. |

#### Reglas por motivo en `RequestItems`

- **IT (Incapacidad Temporal):** Exactamente 1 `RequestItem`. `EmployeeId` obligatorio (con autocompletado desde API SAP/PAI), `StartDate` y `EndDate` obligatorias.
- **Permiso:** Exactamente 1 `RequestItem`. `EmployeeId` obligatorio, `StartDate` y `EndDate` obligatorias.
- **Vacaciones:** 1 o N `RequestItems`. Para cada uno: `EmployeeId` obligatorio, `StartDate` y `EndDate` obligatorias. El botón "+ Añadir persona" permite añadir más filas.
- **Punta de trabajo / Nueva plaza:** 1 `RequestItem` con `IsNewPosition = true`. Solo `StartDate` obligatoria, `EndDate` opcional, `EmployeeId` vacío.

#### Generación del número de petición

La fecha de petición y el número de petición se asignan automáticamente en el momento en que todos los campos obligatorios están correctamente informados y el usuario pulsa "Enviar petición". El formato del número es `PET-{AÑO}-{SECUENCIAL4DÍGITOS}`, p.ej. `PET-2026-0001`. El secuencial es por año y por contrata.

Al enviarse la petición, el estado pasa a `Submitted` y se lanza el Domain Event `RequestSubmittedEvent`, que puede disparar notificaciones a usuarios RRHH de esa contrata.

### 4.2 Paso 2 — RRHH gestiona candidatos

Los usuarios con rol RRHH asignados a la contrata de la petición verán la petición en su bandeja de entrada.

#### Secuencia de acciones para RRHH

1. Abrir la petición y revisar los detalles.
2. Añadir un candidato (`RequestCandidate`) rellenando nombre, DNI, y si tiene contrato previo.
3. El sistema evalúa automáticamente si se requiere revisión médica: si `HasPreviousContract = false`, `MedicalReviewRequired = true`. Si `HasPreviousContract = true` pero `LastMedicalReviewDate` es anterior a `(hoy - ValidityMonths)`, también `MedicalReviewRequired = true`.
4. RRHH registra si el candidato ha pasado la revisión médica (`MedicalReviewResult = true/false`) y la fecha.
5. Si **No Apto**: el candidato queda con estado `MedicalReviewFailed`. RRHH puede añadir otro candidato y repetir el proceso.
6. Si **Apto** (o no requería RM): RRHH marca al candidato como seleccionado (`IsSelected = true`) y el estado de la petición avanza a `PayrollPending`. Se lanza el Domain Event `CandidateApprovedEvent`.

### 4.3 Paso 3 — Nóminas gestiona el alta

Los usuarios con rol Nóminas asignados a la contrata verán las peticiones en estado `PayrollPending`.

#### Secuencia de acciones de Nóminas

| # | Acción | Efecto en el sistema |
|---|---|---|
| 1 | Establecer fecha prevista de alta | `ExpectedStartDate` se guarda. Estado → `HighDateSet`. |
| 2 | Registrar Alta SAP | `SAPHighDate = fecha actual`. Estado → `SAPHighDone`. |
| 3 | Registrar Alta SS | `SSHighDate = fecha actual`. Estado → `SSHighDone`. |
| 4a | Confirmar presentación y firma de contrato | `ContractSignedDate = fecha actual`. Estado → `ContractSigned`. Petición **CERRADA** con éxito. |
| 4b | Registrar no presentación | `NoShowDate = fecha actual`. Estado → `NoShow`. |
| 5 *(si 4b)* | Anular Alta SAP | `SAPHighCancelledDate = fecha actual`. Estado → `SAPHighCancelled`. |
| 6 *(si 4b)* | Anular Alta SS | `SSHighCancelledDate = fecha actual`. Estado → `SSHighCancelled`. Estado final → `Cancelled`. |

> **IMPORTANTE:** Las acciones 2 y 3 son independientes y pueden registrarse en cualquier orden. El paso al estado `SSHighDone` requiere que ambas (SAP y SS) estén completadas.

---

## 5. Capa de Aplicación — Commands y Queries

### 5.1 Commands (escritura)

| Command | Rol que lo ejecuta | Descripción |
|---|---|---|
| `CreateStaffRequestCommand` | Production | Crear borrador de petición |
| `UpdateStaffRequestCommand` | Production | Actualizar campos de una petición en `Draft` |
| `SubmitStaffRequestCommand` | Production | Validar y enviar petición (asigna número y fecha) |
| `AddRequestItemCommand` | Production | Añadir `RequestItem` a una petición en `Draft` |
| `RemoveRequestItemCommand` | Production | Eliminar `RequestItem` de una petición en `Draft` |
| `AddCandidateCommand` | HR | Añadir candidato a una petición |
| `UpdateCandidateCommand` | HR | Actualizar datos del candidato |
| `RegisterMedicalReviewCommand` | HR | Registrar resultado de revisión médica |
| `SelectCandidateCommand` | HR | Marcar candidato como seleccionado y avanzar a Nóminas |
| `SetExpectedStartDateCommand` | Payroll | Establecer fecha prevista de alta |
| `RegisterSAPHighCommand` | Payroll | Registrar alta en SAP |
| `RegisterSSHighCommand` | Payroll | Registrar alta en Seguridad Social |
| `ConfirmContractSignedCommand` | Payroll | Confirmar presentación y firma |
| `RegisterNoShowCommand` | Payroll | Registrar no presentación |
| `CancelSAPHighCommand` | Payroll | Anular alta SAP |
| `CancelSSHighCommand` | Payroll | Anular alta SS |
| `CreateContrataCommand` | Admin | Crear nueva contrata |
| `UpdateContrataCommand` | Admin | Actualizar contrata |
| `CreateWorkCenterCommand` | Admin | Crear centro de trabajo |
| `UpdateMedicalReviewValidityCommand` | Admin | Actualizar meses de validez de RM |
| `AssignUserContrataCommand` | Admin | Asignar contrata a usuario |
| `RemoveUserContrataCommand` | Admin | Desasignar contrata de usuario |

### 5.2 Queries (lectura)

| Query | Descripción |
|---|---|
| `GetStaffRequestsQuery` | Lista paginada y filtrable de peticiones según rol y contratas del usuario |
| `GetStaffRequestDetailQuery` | Detalle completo de una petición con items, candidatos e historial |
| `GetMyRequestsQuery` | Peticiones creadas por el usuario actual (rol Producción) |
| `GetPendingHRRequestsQuery` | Peticiones en estado `Submitted` pendientes de RRHH |
| `GetPendingPayrollRequestsQuery` | Peticiones en `PayrollPending` o posteriores para Nóminas |
| `GetContratassQuery` | Lista de contratas con sus centros |
| `GetUsersQuery` | Lista de usuarios con sus roles y contratas asignadas |
| `GetRequestStatusHistoryQuery` | Historial de cambios de estado de una petición |
| `GetDashboardSummaryQuery` | Contadores por estado para el dashboard principal |

### 5.3 Pipeline Behaviors de MediatR

- **`ValidationBehavior<TRequest, TResponse>`:** Ejecuta todos los validadores FluentValidation del command/query antes del handler. Si hay errores, lanza `ValidationException`.
- **`LoggingBehavior<TRequest, TResponse>`:** Registra entrada y salida de cada request con Serilog.
- **`UnhandledExceptionBehavior<TRequest, TResponse>`:** Captura excepciones no controladas y las loggea.
- **`AuthorizationBehavior<TRequest, TResponse>`:** Verifica que el usuario tiene el rol y las contratas necesarias para ejecutar el command.

---

## 6. Capa de Infraestructura

### 6.1 `ApplicationDbContext`

Clase: `ApplicationDbContext : DbContext`

```csharp
DbSet<Contrata> Contratas
DbSet<WorkCenter> WorkCenters
DbSet<StaffRequest> StaffRequests
DbSet<RequestItem> RequestItems
DbSet<RequestCandidate> RequestCandidates
DbSet<RequestStatusHistory> RequestStatusHistories
DbSet<MedicalReviewValidityConfig> MedicalReviewConfigs
DbSet<UserContrata> UserContratas
```

Configuraciones en `OnModelCreating`:

- Global query filters: `.HasQueryFilter(e => !e.IsDeleted)` en todas las entidades `FullAuditableBaseEntity`.
- Conversión de enums a string (`.HasConversion<string>()`) para legibilidad en BD.
- Índices únicos en `Contrata.Code`, `WorkCenter.Code` (por contrata).
- Cascade delete desactivado globalmente; usar soft delete.
- Schema por defecto: `modelBuilder.HasDefaultSchema("dbo")`.

### 6.2 `AuditSaveChangesInterceptor`

Clase: `AuditSaveChangesInterceptor : SaveChangesInterceptor`

Antes de cada `SaveChanges` intercepta las entidades `Added` y `Modified` para rellenar automáticamente:

- `Created` y `CreatedBy` al crear.
- `LastModified` y `LastModifiedBy` al modificar.
- `Deleted` y `DeletedBy` cuando se detecta borrado lógico (`IsDeleted` cambia a `true`).
- El usuario actual se obtiene desde `ICurrentUserService` (interfaz en Application, implementada en Web/Infrastructure).

### 6.3 Domain Events

Tras el `SaveChanges` exitoso, el DbContext publica los `DomainEvents` acumulados en las entidades mediante `MediatR.Publish()`. Los eventos definidos son:

- `RequestSubmittedEvent`: Notifica a usuarios RRHH de la contrata.
- `CandidateApprovedEvent`: Notifica a usuarios Nóminas de la contrata.
- `ContractSignedEvent`: Notifica a Producción que la petición está cerrada.
- `NoShowRegisteredEvent`: Notifica a RRHH y Nóminas del no presentado.

### 6.4 Migraciones EF Core

Se crearán dos grupos de migraciones:

- **Identity migrations:** En proyecto separado o carpeta `Migrations/Identity`.
- **Application migrations:** Carpeta `Migrations/Application`.

Script de seed inicial para:

- Roles: `Administrator`, `Production`, `HR`, `Payroll`, `Supervisor`.
- Usuario `admin@empresa.com` con rol `Administrator`.
- `MedicalReviewValidityConfig` con `ValidityMonths = 12`.

---

## 7. Capa Web — ASP.NET MVC

### 7.1 Estructura de Controllers

| Controller | Acciones principales |
|---|---|
| `AccountController` | `Login`, `Logout`, `AccessDenied` (usa ASP.NET Identity) |
| `DashboardController` | `Index`: pantalla de inicio por rol con contadores y accesos directos |
| `StaffRequestController` | `Index` (lista), `Create` (GET/POST), `Edit` (GET/POST), `Detail`, `Submit`, `Delete` (soft) |
| `CandidateController` | `AddCandidate`, `UpdateCandidate`, `RegisterMedicalReview`, `SelectCandidate` (todos POST) |
| `PayrollController` | `SetExpectedDate`, `RegisterSAPHigh`, `RegisterSSHigh`, `ConfirmContract`, `RegisterNoShow`, `CancelSAPHigh`, `CancelSSHigh` (todos POST) |
| `AdminController` | Contratas (CRUD), WorkCenters (CRUD), Users (lista, crear, editar, asignar contratas, cambiar rol) |
| `ConfigController` | `MedicalReviewValidity` (GET/POST para cambiar el periodo de validez) |
| `ApiController` *(opcional)* | Endpoint AJAX para autocompletar `EmployeeId` desde SAP/PAI |

### 7.2 Autorización

- Política `Production`: solo rol `Production`.
- Política `HR`: solo rol `HR`.
- Política `Payroll`: solo rol `Payroll`.
- Política `Admin`: solo rol `Administrator`.
- Política `SupervisorOrAbove`: roles `Supervisor`, `Administrator`.
- Las contratas son validadas en los Behaviors y/o Services: el usuario solo puede ver/actuar sobre peticiones de sus contratas asignadas.
- Atributo `[Authorize(Policy = "...")]` en cada Controller o Action.

### 7.3 ViewModels principales

| ViewModel | Uso |
|---|---|
| `StaffRequestListViewModel` | Lista de peticiones con filtros (estado, contrata, rango de fechas) |
| `CreateStaffRequestViewModel` | Formulario de nueva petición con listas de selección dinámicas |
| `StaffRequestDetailViewModel` | Vista completa con timeline de estados, candidatos, historial |
| `AddCandidateViewModel` | Modal/formulario para añadir candidato |
| `MedicalReviewViewModel` | Modal para registrar resultado RM |
| `PayrollActionViewModel` | Modal genérico para acciones de Nóminas (fecha + confirmación) |
| `DashboardViewModel` | Contadores por estado y últimas peticiones recientes |
| `UserManagementViewModel` | Lista de usuarios con rol y contratas |
| `ContrataManagementViewModel` | CRUD de contratas con sus centros de trabajo |

### 7.4 Diseño de UI

- Framework CSS: Bootstrap 5.
- Iconos: Bootstrap Icons o FontAwesome.
- El estado de la petición se muestra mediante un **stepper/timeline visual** que refleja los pasos del workflow.
- Las acciones disponibles en cada estado se muestran como botones contextuales según el rol del usuario.
- Los formularios multi-item (`RequestItems` para Vacaciones) usan JavaScript para añadir/eliminar filas dinámicamente.
- Las modales de Bootstrap se usan para acciones rápidas (añadir candidato, registrar RM, confirmar acciones de Nóminas).
- Mensajes de éxito/error mediante `TempData` + Alert de Bootstrap.
- Paginación del lado del servidor en las listas.

---

## 8. Configuración y `appsettings`

### 8.1 `appsettings.json`

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=...;Database=PersonnelMgmt;Trusted_Connection=True"
  },
  "MedicalReview": {
    "DefaultValidityMonths": 12
  },
  "Serilog": {
    "MinimumLevel": "Information",
    "WriteTo": [{ "Name": "File", "Args": { "path": "logs/log-.txt" } }]
  }
}
```

### 8.2 Registro de dependencias (`Program.cs` / `Startup.cs`)

- `AddDbContext<ApplicationDbContext>`: schema `dbo`.
- `AddIdentity<ApplicationUser, IdentityRole>`: schema `identity`. Cookie auth con redirect a `/Account/Login`.
- `AddMediatR`: escanear assemblies de Application.
- `AddAutoMapper`: escanear assemblies de Application.
- `AddFluentValidation`: escanear assemblies de Application.
- `AddScoped<ICurrentUserService, CurrentUserService>`: implementación Web que usa `IHttpContextAccessor`.
- `AddScoped<IUnitOfWork, UnitOfWork>`.
- `AddTransient<AuditSaveChangesInterceptor>`.

---

## 9. Plan de Implementación por Sprints

El desarrollo se divide en 6 sprints de 1-2 semanas cada uno. Cada sprint produce código funcional y compilable.

### Sprint 1 — Estructura base y configuración

1. Crear solución con 4 proyectos: Domain, Application, Infrastructure, Web.
2. Implementar `FullAuditableBaseEntity` con DomainEvents en Domain.
3. Definir todas las enumeraciones del dominio.
4. Definir todas las entidades del dominio con sus relaciones.
5. Implementar `ApplicationDbContext` con configuraciones Fluent API, global query filters y conversiones de enum.
6. Configurar ASP.NET Identity en esquema separado con `ApplicationUser`.
7. Implementar `AuditSaveChangesInterceptor`.
8. Crear migración inicial y seed de roles y admin.
9. Registrar todas las dependencias en `Program.cs`/`Startup.cs`.
10. Implementar Login/Logout con ASP.NET Identity y vista.

### Sprint 2 — Pipeline MediatR y gestión de contratas

1. Implementar los 4 Pipeline Behaviors: Validation, Logging, UnhandledException, Authorization.
2. Implementar `ICurrentUserService` y `CurrentUserService`.
3. Implementar CRUD completo de Contratas (Command, Handler, Validator, AutoMapper Profile, Controller, Views).
4. Implementar CRUD completo de WorkCenters.
5. Implementar gestión de usuarios: lista, crear, asignar roles, asignar contratas.
6. Implementar `ConfigController` para `ValidityMonths`.
7. Layout principal con navbar por rol y Dashboard básico.

### Sprint 3 — Petición de Personal (rol Producción)

1. Implementar `CreateStaffRequestCommand` + Handler + Validator.
2. Implementar `AddRequestItemCommand` / `RemoveRequestItemCommand`.
3. Implementar `SubmitStaffRequestCommand` con generación de número y fecha.
4. Implementar `UpdateStaffRequestCommand` (solo en `Draft`).
5. Implementar `GetMyRequestsQuery` y `GetStaffRequestDetailQuery`.
6. View: formulario de creación con desplegables dependientes (AJAX contrata→centros) y filas dinámicas de `RequestItems`.
7. View: lista de mis peticiones con filtros básicos y paginación.
8. View: detalle de petición con stepper visual de estados.
9. Implementar `RequestSubmittedEvent` y su handler (log inicial, sin email aún).

### Sprint 4 — Gestión de candidatos (rol RRHH)

1. Implementar `AddCandidateCommand` con lógica de evaluación de revisión médica.
2. Implementar `RegisterMedicalReviewCommand` con validación de resultado.
3. Implementar `SelectCandidateCommand` con transición de estado y `CandidateApprovedEvent`.
4. Implementar `GetPendingHRRequestsQuery`.
5. View: bandeja de entrada RRHH (peticiones en `Submitted`).
6. View: detalle de petición para RRHH con sección de candidatos, modales de añadir candidato y registrar RM.
7. Lógica de validez RM en el Handler (comparar `LastMedicalReviewDate` con `ValidityMonths` desde config).

### Sprint 5 — Acciones de Nóminas y cierre del ciclo

1. Implementar `SetExpectedStartDateCommand`.
2. Implementar `RegisterSAPHighCommand` y `RegisterSSHighCommand` (independientes).
3. Implementar `ConfirmContractSignedCommand` con `ContractSignedEvent`.
4. Implementar `RegisterNoShowCommand`.
5. Implementar `CancelSAPHighCommand` y `CancelSSHighCommand`.
6. Implementar `GetPendingPayrollRequestsQuery`.
7. View: bandeja de entrada Nóminas con acciones contextuales por estado.
8. Implementar `RequestStatusHistory`: registrar cambio en cada transición de estado.
9. View: stepper completo con historial de cambios visible en el detalle.

### Sprint 6 — Dashboard, Supervisor y pulido final

1. Implementar `GetDashboardSummaryQuery` con contadores por estado y por contrata.
2. Dashboard completo con cards de métricas y tabla de peticiones recientes.
3. Implementar rol Supervisor: acceso de solo lectura a peticiones de sus contratas.
4. Implementar `GetStaffRequestsQuery` con filtros avanzados (estado, contrata, centro, rango de fechas, motivo).
5. Vista de lista global con todos los filtros (para Admin y Supervisor).
6. Implementar eliminación lógica de peticiones en `Draft`.
7. Revisión completa de autorización y políticas.
8. Manejo global de errores: middleware de excepciones, páginas de error personalizadas.
9. Revisión de validaciones FluentValidation en todos los Commands.
10. Pruebas manuales del flujo completo end-to-end.

---

## 10. Consideraciones Futuras (fuera de alcance v1)

- **Peticiones múltiples simultáneas:** la estructura de datos ya lo soporta parcialmente (N `RequestItems`). En v2 se añadirá la opción de crear N peticiones de golpe desde la misma pantalla, con un wizard multi-paso.
- **Notificaciones por email:** los Domain Event Handlers ya están preparados. Solo requiere añadir una implementación de `IEmailService` (p.ej. con SendGrid o SMTP).
- **Integración con SAP/PAI/Dorlet:** el campo `EmployeeId` ya existe. En v2 se añadirá un endpoint AJAX de autocompletado que consulte la API externa para rellenar nombre y apellidos automáticamente.
- **Exportación a Excel/PDF:** listado de peticiones exportable.
- **Notificaciones en tiempo real:** SignalR para alertar a usuarios cuando una petición cambia de estado.
- **App móvil:** la arquitectura API-first facilita exponer un API REST en el futuro.

---

## 11. Instrucciones Específicas para la IA Programadora

Al implementar este plan, seguir estas directrices:

1. Implementar en el **orden exacto de los sprints**. No avanzar al siguiente sprint hasta completar el anterior.
2. Cada Command debe tener su propio archivo: `{NombreCommand}.cs`, `{NombreCommandHandler}.cs`, `{NombreCommandValidator}.cs`.
3. Cada Query debe tener: `{NombreQuery}.cs`, `{NombreQueryHandler}.cs`, `{NombreQueryValidator}.cs` (si aplica).
4. Los AutoMapper Profiles se organizan por feature: `StaffRequestProfile.cs`, `ContrataProfile.cs`, etc.
5. Todas las transiciones de estado en los Handlers deben también crear un registro en `RequestStatusHistory`.
6. **Nunca** usar `.Remove()` de EF Core directamente. Solo soft delete estableciendo `Deleted = DateTime.UtcNow`.
7. El `ApplicationDbContext` **NO** debe ser inyectado en controllers ni en la capa Web. Solo a través de repositorios o directamente en Infrastructure handlers.
8. Usar `ICurrentUserService` para obtener el usuario actual en los Handlers, nunca `HttpContext` directamente.
9. Todos los ViewModels de formulario deben tener sus propias Data Annotations o FluentValidation para validación client-side.
10. Usar Tag Helpers de ASP.NET MVC (`asp-for`, `asp-validation-for`, etc.) en todas las vistas Razor.
11. El proyecto debe **compilar y ejecutarse correctamente** al final de cada sprint.
12. Incluir comentarios XML en interfaces y clases públicas.

---

## 12. Resumen de Archivos Clave a Crear

### `PersonnelManagement.Domain`

- `Common/FullAuditableBaseEntity.cs`
- `Enums/WorkShift.cs`, `WorkRegime.cs`, `RequestReason.cs`, `RequestStatus.cs`, `CandidateStatus.cs`
- `Entities/Contrata.cs`, `WorkCenter.cs`, `StaffRequest.cs`, `RequestItem.cs`, `RequestCandidate.cs`, `RequestStatusHistory.cs`, `MedicalReviewValidityConfig.cs`
- `Events/RequestSubmittedEvent.cs`, `CandidateApprovedEvent.cs`, `ContractSignedEvent.cs`, `NoShowRegisteredEvent.cs`
- `Interfaces/IApplicationDbContext.cs`, `ICurrentUserService.cs`, `IUnitOfWork.cs`

### `PersonnelManagement.Application`

- `Common/Behaviors/ValidationBehavior.cs`, `LoggingBehavior.cs`, `UnhandledExceptionBehavior.cs`, `AuthorizationBehavior.cs`
- `Features/StaffRequests/Commands/{Create,Update,Submit,AddItem,RemoveItem}StaffRequest/...`
- `Features/StaffRequests/Commands/Candidate/{Add,Update,RegisterMedicalReview,Select}Candidate/...`
- `Features/StaffRequests/Commands/Payroll/{SetExpectedDate,RegisterSAPHigh,RegisterSSHigh,ConfirmContract,RegisterNoShow,CancelSAPHigh,CancelSSHigh}/...`
- `Features/StaffRequests/Queries/{GetMyRequests,GetPendingHR,GetPendingPayroll,GetDetail,GetList,GetStatusHistory}/...`
- `Features/Admin/Contratas/Commands/{Create,Update}Contrata/...`
- `Features/Admin/WorkCenters/Commands/CreateWorkCenter/...`
- `Features/Admin/Users/Commands/{AssignUserContrata,RemoveUserContrata}/...`
- `Features/Admin/Config/Commands/UpdateMedicalReviewValidity/...`
- `Features/Dashboard/Queries/GetDashboardSummary/...`
- `Mappings/StaffRequestProfile.cs`, `ContrataProfile.cs`, `UserProfile.cs`

### `PersonnelManagement.Infrastructure`

- `Persistence/ApplicationDbContext.cs`
- `Persistence/Configurations/{Contrata,WorkCenter,StaffRequest,RequestItem,RequestCandidate,RequestStatusHistory,MedicalReviewValidityConfig,UserContrata}Configuration.cs`
- `Persistence/Migrations/...`
- `Persistence/Interceptors/AuditSaveChangesInterceptor.cs`
- `Persistence/Seed/ApplicationDbContextSeed.cs`
- `DependencyInjection.cs`

### `PersonnelManagement.Web`

- `Controllers/{Account,Dashboard,StaffRequest,Candidate,Payroll,Admin,Config}Controller.cs`
- `Models/ViewModels/{StaffRequestList,CreateStaffRequest,StaffRequestDetail,AddCandidate,MedicalReview,PayrollAction,Dashboard,UserManagement,ContrataManagement}ViewModel.cs`
- `Views/{Account,Dashboard,StaffRequest,Candidate,Payroll,Admin,Config,Shared}/...`
- `Views/Shared/_Layout.cshtml`, `_StepperPartial.cshtml`, `_CandidatesPartial.cshtml`
- `Services/CurrentUserService.cs`
- `Filters/RequireContrataAccessFilter.cs`
- `wwwroot/js/staffrequest.js` (lógica de desplegables dependientes y filas dinámicas)
- `Program.cs` / `Startup.cs`
- `appsettings.json`, `appsettings.Development.json`
