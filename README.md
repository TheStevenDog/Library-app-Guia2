# Biblioteca · App Móvil (`library_app`)

Aplicación móvil construida con **Expo SDK 57**, **React Native 0.86**, **TypeScript**, **Expo Router**, **SQLite (`expo-sqlite`)**, **Almacenamiento Seguro (`expo-secure-store`)**, **NetInfo** y **Cámara/Galería (`expo-image-picker` + `expo-image-manipulator`)** para la Unidad 03 (Semanas 9 a 12 · Hito 3).

---

## Credenciales de Prueba y URLs de Producción

- **API y Sitio Web en Producción (Vercel)**: [https://library-api-guia1.vercel.app](https://library-api-guia1.vercel.app)
- **Documentación Interactiva (Swagger UI)**: [https://library-api-guia1.vercel.app/api/docs](https://library-api-guia1.vercel.app/api/docs)

| Rol | Correo | Contraseña |
| :--- | :--- | :--- |
| **Estudiante (`student`)** | `estudiante@biblioteca.edu.co` | `Estudiante123` |
| **Bibliotecario (`librarian`)** | `bibliotecario@biblioteca.edu.co` | `Bibliotecario123` |

---

## 1. Cómo ejecutar y verificar el proyecto

### 1.1 Levantar la API (`library_api`)
En una terminal dentro de `../library_api`:
```bash
pnpm seed
pnpm dev:lan
```

### 1.2 Configurar variables de entorno en `library_app`
El archivo `.env` puede apuntar al servidor desplegado en Vercel o a la IP Wi-Fi local de tu equipo (`192.168.1.5`):
```env
EXPO_PUBLIC_API_URL=https://library-api-guia1.vercel.app
```
- **Producción (Vercel)**: `https://library-api-guia1.vercel.app`
- **Teléfono físico en red local (Expo Go)**: `http://192.168.1.5:3000` (o `http://192.168.1.4:3000` por Ethernet).
- **Emulador de Android**: `http://10.0.2.2:3000`
- **Simulador de iOS**: `http://localhost:3000`

### 1.3 Arrancar la app móvil
```bash
npx expo start
```

### 1.4 Comandos de verificación y calidad (Puntos de control)
```bash
npm run typecheck
npx expo lint
npm test
npx expo export
```

### 1.5 Ejecutar la prueba E2E real contra `library_api`
Con `library_api` encendida en `http://localhost:3000` (o apuntando a Vercel):
```bash
E2E_API_URL=https://library-api-guia1.vercel.app \
E2E_STUDENT_EMAIL=estudiante@biblioteca.edu.co E2E_STUDENT_PASSWORD=Estudiante123 \
E2E_LIBRARIAN_EMAIL=bibliotecario@biblioteca.edu.co E2E_LIBRARIAN_PASSWORD=Bibliotecario123 \
npx jest e2e-api
```

---

## 2. Evidencia Semana 9 · Consumo de API REST

### Tabla de pruebas de consumo y estados de pantalla
| Escenario | Cómo provocarlo | Resultado obtenido | Estado |
| :--- | :--- | :--- | :---: |
| **Respuesta válida** | Abrir la app con sesión iniciada y API activa | Se descargan y listan los 10 libros del catálogo | ✅ |
| **Estado vacío** | Buscar `zzzz` en el buscador | Se muestra `EmptyView` con título *"Sin resultados"* | ✅ |
| **Servidor apagado** | Detener `library_api` sin caché previa o forzar sincronización | Se muestra estado de error legible y botón **Reintentar** / aviso en `ConnectionBanner` | ✅ |
| **Reintento exitoso** | Encender la API y tocar **Reintentar** / banner | Se ejecuta `syncNow()` y aparece la lista de libros | ✅ |
| **URL incorrecta / Timeout** | Cambiar `EXPO_PUBLIC_API_URL` a una IP inalcanzable | `AbortController` cancela la petición a los 10 s (`REQUEST_TIMEOUT_MS = 10_000`) y lanza `NetworkError` sin congelar la UI | ✅ |
| **Detalle** | Tocar cualquier tarjeta de libro (`BookCard`) | Navega a `/book/[id]` mostrando portada, autor, categoría, año, disponibilidad y descripción | ✅ |

---

## 3. Evidencia Semana 10 · Modelado de Amenazas y Seguridad (OWASP MASVS)

| Activo | Amenaza identificada | Control implementado en `library_app` y `library_api` | Verificación |
| :--- | :--- | :--- | :--- |
| **Tokens de sesión (`idToken`, `refreshToken`)** | Lectura desde copia de seguridad no cifrada, SQLite o teléfono con root | Guardados únicamente en `expo-secure-store` (Keychain en iOS / Keystore en Android), **cada valor en su propia llave** (`auth.idToken`, `auth.refreshToken`, `auth.expiresAt`, `auth.user`) para respetar el límite de ~2 KB en iOS | `__tests__/session-manager.test.ts` |
| **Contraseña del usuario** | Persistencia accidental en almacenamiento local o exposición en logs | Nunca se guarda ni se registra en `activity_log` ni consola; solo viaja en el cuerpo de `/api/auth/login` y `/api/auth/register` | Revisión de `auth-service.ts` y `repositories.ts` |
| **Rol del usuario (`student` / `librarian`)** | Modificación del cliente móvil para mostrar botones de bibliotecario a un estudiante | Las rutas y vistas ocultan opciones por UX, pero **`library_api` verifica el rol desde el JWT firmado por Firebase en cada petición**, respondiendo `403 FORBIDDEN` | `__tests__/e2e-api.test.ts` |
| **Préstamos de otro usuario** | Un estudiante consulta o modifica un préstamo ajeno por ID | La API filtra por `uid` del token y responde `404` si no pertenece al estudiante | Reglas de `library_api` |
| **Datos de otra cuenta en el mismo teléfono** | Un segundo usuario inicia sesión en el mismo teléfono y hereda préstamos o la cola `outbox` del usuario anterior | `DataProvider` guarda `owner.uid` en `sync_meta`. Si al iniciar sesión `owner !== uid`, ejecuta `loans.clear()` y `outbox.clear()`. Al cerrar sesión borra préstamos, cola y metadatos | `src/providers/data-provider.tsx` |
| **Token vencido o revocado** | Peticiones fallidas tras 1 hora de vigencia o sesión eterna tras robo de token | Renovación proactiva 60 s antes de vencer (`EXPIRY_MARGIN_MS`), deduplicación de promesas de refresh en vuelo, reintento único ante `401`, y cierre automático de sesión (`clear()`) si `/api/auth/refresh` responde `401/403` | `__tests__/api-client.test.ts` y `session-manager.test.ts` |
| **Pérdida temporal de red** | Cerrar la sesión del usuario por un fallo de internet al intentar renovar el token | Si el refresh lanza `NetworkError`, **no se cierra la sesión**: se conserva el usuario en local para permitir trabajo offline | `__tests__/session-manager.test.ts` |

---

## 4. Evidencia Semana 11 · Justificación de la Estrategia Offline-First y Resolución de Conflictos

### 4.1 Por qué la interfaz siempre lee de SQLite
La aplicación implementa el patrón **Offline-First**: las pantallas (`BooksScreen`, `LoansScreen`, `BookDetailScreen`) nunca llaman a `fetch` directamente para renderizar listas; leen de SQLite mediante los hooks de `use-library-data.ts`. La red solo alimenta y vacía la base de datos local a través de `SyncService`. Esto garantiza respuesta inmediata al abrir la app y funcionamiento continuo sin conexión.

### 4.2 Orden de escritura local (Cola antes que préstamo)
En `loan-service.ts`, al solicitar un libro sin conexión:
1. Primero se inserta la operación en la tabla `outbox` (`enqueue`).
2. Después se inserta el préstamo local con `sync_state = 'pending'` (`insertPending`).
Si la aplicación se cierra de forma inesperada entre ambos pasos, en el siguiente arranque `pushOutbox()` enviará la solicitud al servidor y luego `pullLoans()` descargará el préstamo creado, evitando que quede un préstamo "fantasma" en pantalla que jamás se enviaría.

### 4.3 Idempotencia con `clientRequestId`
Cada solicitud genera un UUID en el dispositivo (`clientRequestId`). Si el teléfono envía `POST /api/loans` y pierde la conexión antes de recibir la respuesta `201`, el elemento permanece en `outbox` y se reenviará al reconectar. Como el servidor verifica `clientRequestId`, devuelve el préstamo ya creado en lugar de duplicarlo, y `upsertFromServer` reemplaza la fila temporal `local-<clientRequestId>` por el ID definitivo del servidor dentro de una transacción.

### 4.4 Justificación de la estrategia de conflictos: *"Gana el servidor"*
En una biblioteca física, **el inventario real de ejemplares disponibles es un recurso compartido centralizado**. El teléfono solo conoce una instantánea pasada del catálogo:
- Si un estudiante solicita el último ejemplar de *Rayuela* estando en modo avión, y antes de que recupere conexión otro estudiante lo solicita y el bibliotecario lo aprueba, el servidor ya no tiene ejemplares (`availableCopies = 0`).
- El teléfono **no puede imponer su estado local** sobre el inventario físico. Por ello, **el servidor es la fuente de verdad ("gana el servidor")**: responde `409 NO_COPIES` (o `ALREADY_ACTIVE`).
- Al recibir `409`, `SyncService` no bloquea el resto de la cola `outbox`: marca ese préstamo local como `sync_state = 'conflict'`, guarda la explicación legible en `sync_error` (*"Cuando se envió tu solicitud ya no quedaban ejemplares disponibles."*), retira el ítem de `outbox` y actualiza el catálogo local con `pullBooks()`. El estudiante ve el motivo exacto en rojo y puede descartar el aviso con el botón **Descartar**.
- Del mismo modo, las acciones del bibliotecario (**Prestar**, **Rechazar**, **Registrar devolución**) exigen conexión activa: decidir sobre el inventario compartido sin conexión produciría conflictos cruzados entre múltiples estudiantes difíciles de revertir.

---

## 5. Evidencia Semana 12 · Servicio del Dispositivo (Cámara/Galería) y Trazabilidad

### 5.1 Buenas prácticas implementadas (`src/services/device/cover-service.ts`)
- **Mínimo privilegio**: Solo se solicita permiso de cámara (`requestCameraPermissionsAsync`), con `microphonePermission: false` en `app.json`. Para la galería se utiliza el selector del sistema (`launchImageLibraryAsync`), que no requiere permisos de lectura de todo el almacenamiento.
- **Permiso en contexto y recuperación**: El permiso se solicita únicamente cuando el bibliotecario toca **Tomar foto**. Si es denegado y `canAskAgain === false`, se ofrece un diálogo con enlace directo a los ajustes del sistema (`Linking.openSettings()`).
- **Reducción y compresión adaptativa antes de subir**: La imagen se redimensiona a `600 px` de ancho y se comprime en JPEG probando calidades decrecientes (`0.7`, `0.5`, `0.35`, `0.2`) hasta que el payload base64 sea $\le 700\,000$ caracteres (~520 KB).
- **Trazabilidad sin datos sensibles**: Cada paso registra un evento técnico en `activity_log` (`cover.permission`, `cover.pick`, `cover.upload`) conservando máximo los últimos 200 registros y sin almacenar la foto ni tokens en la bitácora.

---

## 6. Guía rápida para la Sustentación en Vivo (Sección 6.6 · Hito 3)

1. **¿Cómo viaja una solicitud de préstamo desde el toque del botón hasta Firestore, y qué pasa si falla cada tramo?**
   - **Toque en UI (`BookDetailScreen`)**: Llama a `requestLoan(book)` del `DataProvider`.
   - **Validación y persistencia local (`loan-service.ts`)**: Verifica en SQLite que no haya solicitud activa (`hasActiveForBook`) y que `availableCopies > 0`. Genera `clientRequestId` (UUID), encola en `outbox` y guarda en `loans` con `sync_state = 'pending'`. Si aquí no hay red, la UI ya muestra *"Solicitud guardada · pendiente de envío"*.
   - **Sincronización (`sync-service.ts`)**: `pushOutbox()` lee `outbox` en orden FIFO y envía `POST /api/loans` con `{ bookId, clientRequestId }` y el header `Authorization: Bearer <idToken>`.
     - Si falla por red (`NetworkError`), conserva toda la cola para cuando `NetInfo` o `AppState` detecten reconexión.
     - Si falla por `5xx` o `429`, incrementa `attempts` en `outbox` (`recordFailure`) y reintenta hasta `MAX_ATTEMPTS = 5`.
     - Si falla por `409`, marca `sync_state = 'conflict'` con mensaje claro y lo saca de `outbox`.
   - **Servidor (`library_api`)**: Valida el JWT con Firebase Admin, comprueba idempotencia por `clientRequestId` y disponibilidad en Firestore, crea el documento en `loans` y responde `201`.
   - **Convergencia en SQLite**: `upsertFromServer` borra la fila `local-...` e inserta la versión oficial con `sync_state = 'synced'`, y `outbox.remove` limpia la cola.

2. **¿Por qué los tokens están en `SecureStore` y qué ocurre con un `401`?**
   - Porque `AsyncStorage` y SQLite guardan texto plano extraíble desde un backup o un dispositivo con root. `expo-secure-store` cifra los tokens usando el Keychain de iOS y el Keystore de Android, guardando cada token en su propia llave para no superar el límite de 2 KB de iOS.
   - Cuando una petición autenticada recibe `401`, el cliente HTTP (`client.ts`) invoca `auth.refreshSession()`, que en `SessionManager` envía el `refreshToken` a `/api/auth/refresh` (compartiendo una única promesa si hay varias peticiones simultáneas). Si se renueva, guarda los nuevos tokens y **reintenta la petición original una sola vez**. Si el servidor responde `401/403` al refresh, `SessionManager` ejecuta `clear()` y `Stack.Protected` redirige automáticamente al login.

3. **¿Qué es `clientRequestId` y qué problema resuelve?**
   - Es una llave de idempotencia (UUID v4) generada en el teléfono al momento de crear la solicitud. Resuelve el problema de **duplicación por incertidumbre de red**: si el teléfono envía la petición, el servidor la procesa, pero la conexión cae antes de que llegue la respuesta al teléfono, el teléfono volverá a enviar el mismo `clientRequestId` al reconectar y el servidor devolverá el préstamo original en lugar de crear un segundo préstamo.

4. **¿Qué pasa con una solicitud hecha sin conexión cuando el ejemplar se agota?**
   - Al volver la conexión, `pushOutbox()` envía `POST /api/loans` y el servidor responde `409` con código `NO_COPIES`. `SyncService` captura ese `ApiRequestError`, traduce el código con `conflictMessage` a *"Cuando se envió tu solicitud ya no quedaban ejemplares disponibles."*, actualiza el préstamo local a `sync_state = 'conflict'` (`markConflict`), elimina la tarea de `outbox` para no bloquear las siguientes solicitudes, y descarga el catálogo actualizado (`availableCopies = 0`).

5. **¿Cómo agregarías una columna a `books` con una migración nueva?**
   - En `src/db/migrations.ts`:
     1. Incrementar `export const DATABASE_VERSION = 3;` (nunca editar los bloques `version < 1` ni `version < 2`).
     2. Añadir al final de `migrateDbIfNeeded`, antes de actualizar `PRAGMA user_version`:
        ```ts
        if (version < 3) {
          await db.withTransactionAsync(async () => {
            await db.execAsync(`ALTER TABLE books ADD COLUMN isbn TEXT;`);
          });
          version = 3;
        }
        ```

---

## 7. Bitácora de Uso de IA Generativa (Sección 9)

| Fecha / Semana | Herramienta y prompt (resumido) | Qué respondió | Cómo lo validé | Decisión |
| :--- | :--- | :--- | :--- | :--- |
| **Semana 9** | *"Cómo implementar timeout y cancelación en fetch para React Native sin librerías externas"* | Propujo usar `AbortController` con `setTimeout` y limpiar el timer en `finally`, propagando también `options.signal`. | Ejecuté las pruebas unitarias en `__tests__/api-client.test.ts` verificando que al superar `timeoutMs` se rechaza con `NetworkError`. | **Aceptada**: integrada en `src/services/api/client.ts`. |
| **Semana 10** | *"Dónde guardar el idToken y refreshToken de Firebase en una app Expo y cómo serializarlos"* | Inicialmente sugirió guardar un único JSON con `JSON.stringify({ tokens, user })` en `AsyncStorage` o en una sola clave de `SecureStore`. | Contrasté con **OWASP MASVS** y la documentación de `expo-secure-store`: `AsyncStorage` no es seguro (texto plano) y en iOS `SecureStore` rechaza valores mayores a ~2 KB (un JWT de Firebase mide ~1 KB). | **Rechazada parcialmente**: usé `expo-secure-store` separando cuatro llaves independientes (`auth.idToken`, `auth.refreshToken`, `auth.expiresAt`, `auth.user`) y lo validé en `__tests__/session-manager.test.ts`. |
| **Semana 11** | *"Cómo probar consultas y migraciones de expo-sqlite en Jest sin emulador"* | Sugirió crear un adaptador sobre `DatabaseSync` de `node:sqlite` (incluido en Node 24) que implemente `execAsync`, `runAsync`, `getAllAsync`, `getFirstAsync` y `withTransactionAsync`. | Ejecuté `__tests__/database.test.ts` y `__tests__/offline-flow.test.ts` comprobando que el SQL real (incluyendo `ON CONFLICT`, `CHECK` y `PRAGMA user_version`) corre en memoria en Jest. | **Aceptada**: implementada en `test-utils/node-sqlite.ts`. |
| **Semana 12** | *"Cómo comprimir una foto de cámara en Expo para no superar 700 KB en base64"* | Propujo redimensionar a un ancho fijo (`600 px`) con `expo-image-manipulator` e iterar sobre un arreglo de calidades JPEG decrecientes (`[0.7, 0.5, 0.35, 0.2]`) hasta que `base64.length <= MAX_BASE64_LENGTH`. | Verifiqué el tipado con `npm run typecheck`, el linter con `npx expo lint` y el manejo de errores (`ImageTooLargeError` y `PermissionDeniedError`). | **Aceptada**: integrada en `src/services/device/cover-service.ts`. |
