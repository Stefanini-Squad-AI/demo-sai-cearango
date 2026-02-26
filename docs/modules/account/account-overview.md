# 💳 ACCOUNT - Módulo de Cuentas

**Módulo ID**: `account`  
**Versión**: 1.0  
**Última actualización**: 2026-02-26  
**Propósito**: Proveer al back-office y a administradores una experiencia guiada para consultar y actualizar cuentas de crédito con validaciones automáticas, mascarado de datos sensibles y soporte transaccional completo dentro del menú principal.

---

## 📋 Descripción general

Este módulo combina dos pantallas protegidas (`/accounts/view` y `/accounts/update`) con servicios compartidos para:

- Buscar cuentas por su `accountId` de 11 dígitos y mostrar saldos, ciclos, tarjetas asociadas y datos del cliente.
- Editar información financiera y de contacto del cliente con detección automática de cambios y confirmación transaccional.
- Cumplir reglas PCI/financieras como mascarado de SSN/tarjeta, validaciones de FICO y ZIP, y bloqueo de estados no válidos.
- Enlazar con el menú principal/back-office (`MenuData`) y respetar el rol almacenado en `localStorage` (admin vs back-office).

---

## 🏗️ Arquitectura y componentes clave

1. **`AccountViewPage.tsx` / `AccountUpdatePage.tsx`** – rutas protegidas que validan `userRole` desde `localStorage`, redirigen a `/menu/admin` o `/menu/main` y delegan la UI a los screens especializados.
2. **`AccountViewScreen.tsx`** – formulario con validación del `accountId`, tarjetas de información financiera/personal, toggle para mostrar datos sensibles y alertas para errores/info expuestas por `AccountViewResponse`.
3. **`AccountUpdateScreen.tsx`** – layout multisección (Account + Customer), switch “Edit mode”, validaciones locales (creditLimit, ZIP, status), botones F3/F5/F12 y diálogo modal de confirmación antes del `PUT`.
4. **Hooks `useAccountView` / `useAccountUpdate`** – encapsulan `useMutation` + `apiClient`, exponen estados (`loading`, `error`, `hasChanges`) y lógica de comparación JSON para detectar cambios no guardados.
5. **`apiClient` + `useMutation`** – cliente HTTP central con detección automática de respuestas directas del backend (sin `success`) y de MSW, manejo de errores/timeouts y logging de etapas.
6. **`MenuData`** – expone opciones `account-view` y `account-update` para los menús principales de back-office/admin, indicando rutas y descripciones.
7. **Mocks MSW (`app/mocks/accountHandlers.ts`)** – reproducen `GET /api/account-view`, `GET /api/account-view/initialize`, `/process`, `/test-accounts`, `/test-error/:errorType` con delays controlados y respuestas de error/timeout para pruebas locales.

---

## 🔗 APIs Documentadas

| Endpoint | Propósito | Respuesta clave |
| --- | --- | --- |
| `GET /api/account-view?accountId={padded11}` | Búsqueda principal para `AccountViewScreen`. Valida que el ID tenga 11 dígitos y no sea cero. | `AccountViewResponse` (montos `creditLimit`, `currentBalance`, `ficoScore`, `groupId`, mascarados `customerSsn/cardNumber`, mensajes `infoMessage`/`errorMessage`). |
| `GET /api/account-view/initialize` | Inicializa la vista con metadata (`transactionId`, `infoMessage`, fecha). | `AccountViewResponse` con `inputValid: true` y mensaje “Enter or update id…”. |
| `GET /api/accounts/{accountId}` | Carga los datos editables (`AccountUpdateData`) para el formulario de edición. | Campos financieros + cliente (nombres, addressLine*, `ssn`, `stateCode`, `countryCode`, `ficoScore`). |
| `PUT /api/accounts/{accountId}` | Persiste cambios simultáneos de `Account` y `Customer`. | `AccountUpdateResponse` con `success`, `data` actualizado y posibles `errors`. |
| `POST /api/account-view/process` (MSW) | Simula la búsqueda con delays de 600–1000ms, soporta errores de red/timeouts/500. | Misma estructura de `AccountViewResponse`. |
| `GET /api/account-view/test-accounts` (MSW) | Devuelve 10 cuentas de prueba con balances y ciudades para botones de desarrollo. | Lista `accounts`. |
| `POST /api/account-view/test-error/:errorType` (MSW) | Fuerza errores `network`, `timeout` o `server-error` para validar manejo de errores en UI. |

### Ejemplo de request/response real:

Request:
```http
GET /api/account-view?accountId=00011111111
```

Response (simplificado):
```json
{
  "accountId": 11111111111,
  "creditLimit": 5000,
  "currentBalance": 1250.75,
  "customerSsn": "***-**-6789",
  "inputValid": true,
  "infoMessage": "Displaying details of given Account"
}
```

---

## 📊 Modelos de datos

### `AccountViewResponse` (TypeScript - `app/types/account.ts`)

```typescript
interface AccountViewResponse {
  currentDate: string;
  transactionId: string;
  accountStatus?: 'Y' | 'N';
  creditLimit?: number;
  cashCreditLimit?: number;
  groupId?: string;
  customerSsn?: string;
  ficoScore?: number;
  cardNumber?: string;
  infoMessage?: string;
  errorMessage?: string;
  inputValid: boolean;
}
```

### `AccountUpdateData` / `AccountUpdateResponse` (`app/types/accountUpdate.ts`)

```typescript
interface AccountUpdateData {
  accountId: number;
  activeStatus: 'Y' | 'N';
  creditLimit: number;
  cashCreditLimit: number;
  stateCode: string;
  zipCode: string;
  ssn: string;
  ficoScore: number;
  primaryCardIndicator: string;
}

interface AccountUpdateResponse {
  success: boolean;
  data?: AccountUpdateData;
  errors?: string[];
}
```

---

## 📋 Reglas de negocio (RN)

- **RN-001**: `accountId` debe tener exactamente 11 dígitos numéricos y no ser `00000000000` antes de ejecutar la búsqueda o la actualización.  
- **RN-002**: `activeStatus` solo admite `'Y'` (activo) o `'N'` (inactivo); el select/control lo valida localmente.  
- **RN-003**: `creditLimit`, `cashCreditLimit` y `currentBalance` deben ser numéricos; `currentBalance` puede ser negativo (sobregiro).  
- **RN-004**: `zipCode` cumple `^\d{5}(-\d{4})?$` y `ficoScore` permanece entre 300 y 850 para habilitar el botón Save.  
- **RN-005**: SSN y número de tarjeta están enmascarados por defecto (`***-**-1234`, `****-****-****-1234`) hasta que el usuario activa “Show Sensitive Data”.  
- **RN-006**: El módulo requiere modo edición, detecta cambios (`hasChanges`) y abre un diálogo de confirmación antes de enviar el `PUT` transaccional.

---

## 🎯 User Story Patterns

1. *Como representante de servicio*, quiero buscar una cuenta por ID para presentar rápidamente balances y límites y resolver consultas en menos de 500 ms.  
2. *Como administrador*, quiero editar el límite de crédito y los datos de contacto con validaciones automáticas para mantener la consistencia del CRM.  
3. *Como oficial de cumplimiento*, quiero que el SSN y el número de tarjeta estén enmascarados hasta que el operador confirme su visualización.  
4. *Como QA*, quiero forzar errores de red (MSW `/test-error/network`) para verificar el flujo de manejo de fallos sin depender del backend.

---

## ⚡ Patrones de aceleración

- **Hooks centralizados `useAccountView` / `useAccountUpdate`**: reusables para cualquier pantalla que necesite buscar o actualizar cuentas; encapsulan `useMutation`, `apiClient` y `reset`.  
- **`SystemHeader` + `LoadingSpinner`**: layout consistente con título/programa e íconos Material-UI ya disponibles.  
- **Material-UI**: `TextField`, `Chip`, `Grid`, `Dialog` y `Button` con estilos compartidos aceleran la creación de nuevas secciones.  
- **Validaciones locales**: switch “Edit mode”, `validationErrors` y detección de cambios permiten extender reglas sin tocar backend.  
- **Mocks MSW**: `/account-view/test-accounts` y `/account-view/test-error` habilitan pruebas offline y casos de error controlados.

---

## 🧭 Dependencias internas y externas

- **Dependencias internas**: `apiClient`, `useMutation`, `MenuData`, `SystemHeader`, `LoadingSpinner`.  
- **Dependencias externas**: Material-UI (`@mui/material`, `@mui/icons-material`), React Router (paras rutas protegidas), Redux Toolkit (para el menú y autenticación), MSW (mocks).  
- **Módulos consumidores**: `Menu` (presenta opciones `account-view`/`account-update`), `Auth` (requiere sesión con `userRole`), `Layout` (shared header y footer).

---

## 🧪 Testing, mocks y QA

1. `app/mocks/accountHandlers.ts` cubre `GET /api/account-view`, `/initialize`, `/process`, `/test-accounts`, `/test-error/:errorType` con delays y respuestas de error explícitas.  
2. `useMutation` imprime logs y lanza errores `ApiError` para que los screens los muestren vía `Alert`.  
3. Validaciones de campo se simulan en pantalla (regex para ZIP, numérico, FICO).  
4. Los botones F3/F5/F12 y conjunto de shortcuts se probaban manualmente en el UI actual.

---

## 🚨 Riesgos y mitigaciones

- **Performance en búsquedas**: búsqueda encadenada `CardXref → Account → Customer` podría degradarse. *Mitigación*: índices en `accountId`/`customerId`, considerar caché Redis para cuentas frecuentes.  
- **Falta de i18n**: textos hardcodeados (alertas, placeholders). *Mitigación*: introducir `react-i18next` antes de nuevas historias (evaluado en ~5 pts).  
- **Sin auditoría**: no hay trazabilidad de quién modificó valores. *Mitigación*: integrar Spring Data Envers o similar con `Audit` (prioridad 5 pts).  
- **Validaciones antiguas**: verificaciones migradas de COBOL están comentadas. *Mitigación*: validar/reemplazar por reglas nuevas y unificar en `AccountValidationService`.  
- **Dependencia de mocks**: MSW se usa ampliamente, por lo que el backend debe garantizar la misma estructura JSON para evitar falls.

---

## 📈 Métricas de éxito

- Búsqueda completada en < 500 ms (P95).  
- Capacidad para 100 búsquedas concurrentes/segundo sin degradar la UI.  
- Amazon de adopción: 98% de los operadores usan la pantalla para atención.  
- Cobertura de pruebas de validaciones críticas (Account ID, ZIP, status).

---

## 🔁 Flujo principal

1. Usuario navega desde menú (`MenuData`) → `AccountViewPage` / `AccountUpdatePage`.  
2. Hook (`useAccountView` / `useAccountUpdate`) ejecuta `useMutation` que llama a `apiClient`.  
3. `apiClient` detecta si la respuesta es MSW o backend real y la normaliza.  
4. El screen muestra datos, valida campos y publica alertas/mascarados.  
5. Si hay cambios, `hasChanges` activa el botón Save y un diálogo confirma antes de `PUT`.  
6. Respuesta exitosa resetea `originalData` y actualiza `hasChanges`.

---

## 📚 Referencias

- `app/pages/AccountViewPage.tsx`, `AccountUpdatePage.tsx`  
- `app/components/account/AccountViewScreen.tsx`, `AccountUpdateScreen.tsx`  
- `app/hooks/useAccountView.ts`, `app/hooks/useAccountUpdate.ts`  
- `app/types/account.ts`, `app/types/accountUpdate.ts`  
- `app/mocks/accountHandlers.ts`  
- `app/data/menuData.ts`  
- `docs/site/modules/accounts/index.html` (guía complementaria)
