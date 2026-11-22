# Bitácora técnica del proyecto ShipNow

## Actualización Entrega 8: Rendimiento, producción y Docker

En esta pre-entrega revisé el comportamiento de la API y agregué medidas básicas para ejecutarla de forma más controlada. Las decisiones se enfocaron en limitar respuestas, validar la configuración al inicio y documentar una ejecución reproducible con Docker.

### Resumen técnico de lo incorporado

1. **Performance y escalabilidad**
   - Se agregó paginación y límites en listados de usuarios, órdenes, entregas y productos (`page`, `limit` con máximo de 100).
   - Se agregaron filtros en listados (`role`, `email`, `status`, `priority`, `customer`, `driver`, `order`).
   - Los productos muestran disponibles con stock por defecto; `includeInactive=true` amplía el listado.

2. **Preparación para producción**
   - Se validan variables críticas al inicio, incluidos puerto, URI de MongoDB y valores booleanos.
   - En `production`, la autenticación queda activa y requiere `JWT_SECRET`.
   - `REQUEST_BODY_LIMIT` es configurable para evitar cuerpos de request excesivos.
   - El health check en `/api/health` responde solo datos seguros: `status`, `environment`, `uptimeSeconds`, `timestamp`.
   - En producción, mocks, logger test y Swagger quedan deshabilitados por defecto.

3. **Seguridad y consistencia de dominio**
   - Endpoints de escritura usan JWT + roles cuando `AUTH_REQUIRED=true`.
   - Se reforzó la regla de negocio: una orden no puede tener más de una entrega (respuesta 409 en conflicto).
   - Se mantuvo manejo transaccional con fallback controlado según capacidades de MongoDB.

4. **Dockerización**
   - Se incorporó un `Dockerfile` con:
     - `npm ci --omit=dev`
     - ejecución con usuario no-root (`node`)
     - `HEALTHCHECK` sobre `/api/health`
   - `compose.yaml` permite levantar la API y MongoDB localmente; los datos se guardan en volúmenes.
   - Se completó `.dockerignore` para no copiar secretos ni artefactos locales:
     - `.env*`, `node_modules`, logs, uploads generados, coverage y temporales.

5. **Documentación y pruebas**
   - README actualizado con variables por entorno, ejecución local, tests, Swagger y Docker.
   - OpenAPI actualizado con `BearerAuth` y parámetros de paginación/filtros.
   - Suite funcional verificada: **53 pruebas aprobadas**.
   - La imagen se construyó y se comprobó en ejecución: health check, Swagger y listado principal respondieron correctamente.

## 1) Alcance y criterio de trabajo

Este proyecto se desarrolló con foco en un backend profesional y mantenible para una solución logística, priorizando:

- separación de responsabilidades por capas
- integración clara entre negocio y persistencia
- manejo centralizado de errores y logging
- documentación técnica útil para revisión y consumo
- crecimiento controlado para nuevas funcionalidades

## 2) Línea de evolución del proyecto

### Etapa A - Base y estructura

Se definió la base Express + MongoDB con una organización por capas:

- `app.js` para middlewares y rutas
- `server.js` para bootstrap y conexión a MongoDB
- `routes/controllers/services/repositories/models`

Resultado: estructura limpia y escalable para incorporar nuevas entidades y comportamientos sin acoplar HTTP y dominio.

### Etapa B - Dominio y casos de negocio

Se modelaron entidades principales: usuarios, productos, pedidos y entregas.

Se implementaron CRUDs y validaciones iniciales de negocio en services:

- existencia de recursos relacionados
- datos requeridos antes de persistir
- estados y prioridades con constantes
- reglas de negocio para asignación y actualización de entregas

Resultado: API funcional con lógica de negocio centralizada en la capa de servicios.

### Etapa C - Errores centralizados

Se migró a un manejo uniforme de errores:

- diccionario central en `src/utils/errors.js`
- `createError` + `CustomError`
- middleware global de errores
- middleware 404 para rutas inexistentes

Resultado: respuestas consistentes para el cliente y mayor trazabilidad para diagnóstico.

### Etapa D - Logging y observabilidad

Se incorporó logging estructurado:

- logger central con niveles
- middleware HTTP para trazas por request
- salida por consola y archivos rotados
- registros de eventos relevantes para negocio y carga documental

Resultado: mayor visibilidad operativa y mejor capacidad para detectar fallos reales.

### Etapa E - Swagger/OpenAPI

Se documentaron endpoints reales de la API:

- configuración en `src/config/swagger.js`
- UI en `/api/docs` y `/api-docs`
- documentación por módulos (`Users`, `Products`, `Orders`, `Deliveries`, `Mocks`, `Logger`)
- schemas y respuestas reutilizables

Resultado: documentación útil para revisión técnica, integración y validación manual.

### Etapa F - Módulo 7: carga de archivos, documentos y comprobantes

**Desafío**: Incorporar gestión de archivos de forma segura, validada y asociada a entidades sin perder trazabilidad.

**Decisiones tomadas**:

1. **Almacenamiento: diskStorage vs base de datos vs cloud**
   - ❌ Guardar en MongoDB: archivos grandes ralentizan consultas
   - ❌ S3/cloud: complejidad innecesaria para proyecto educativo
   - ✅ **Elegí diskStorage**: simple, rápido, separar datos (BD) de archivos (disco)
   - Metadatos en MongoDB para queries rápidas, archivos en carpetas

2. **Organización: carpeta única vs carpetas por tipo**
   - ❌ Una carpeta: difícil de auditar y hacer retención selectiva
   - ✅ **Elegí 3 carpetas** (`documents`, `receipts`, `proofs`): facilita backups, auditoría y políticas de retención diferentes por tipo

3. **Nombres de archivo: original vs timestamp + aleatorio**
   - ❌ Nombre original: caracteres especiales, colisiones, riesgos de seguridad
   - ✅ **Elegí timestamp + aleatorio**: único, seguro, imposible de adivinar. El original se guarda en metadatos

4. **Validaciones: una capa vs dos capas**
   - Multer escribe el archivo en disco antes de que el servicio compruebe que la entidad existe.
   - ✅ **Elegí dos capas**:
     - Multer valida tipo/tamaño/extensión rápidamente
     - Service valida que la entidad exista antes de asociar
     - Si la asociación falla, el archivo no se elimina automáticamente; queda como mejora pendiente definir una limpieza de archivos huérfanos

5. **Qué guardar en BD: solo ruta vs metadatos completos**
   - ❌ Solo ruta: no puedo hacer queries por tipo de documento o tamaño
   - ✅ **Elegí metadatos completos**: `originalName`, `fileName`, `path`, `mimetype`, `size`, `type`, `uploadedAt`
   - Permite auditoría, queries de negocio, soft-deletes

**Resultado**: Los archivos y sus metadatos se almacenan por separado y las cargas tienen límites y tipos permitidos. La aplicación no limpia automáticamente los archivos huérfanos cuando falla una asociación; esa limitación se registra como deuda técnica.

## 3) Ajustes de calidad realizados

Durante la revisión final del proyecto se corrigieron desalineaciones entre configuración, documentación y comportamiento real:

1. Se centralizó la lógica de almacenamiento y validación de archivos en `src/middlewares/uploadFileMiddleware.js`.
2. Se conectó el manejo de errores de Multer con el formato general del backend en `src/middlewares/errorHandler.js`.
3. Se definieron errores específicos para archivos y tipo de documento en `src/utils/errors.js`.
4. Se incorporó la asociación de metadatos a documentos y comprobantes en los modelos correspondientes.
5. Se ignoró la carpeta `uploads/` en Git para evitar subir contenido generado en runtime.
6. Se ajustó la documentación para reflejar el comportamiento real del sistema y ejemplos de uso de endpoints de archivos.
7. Se validó que la configuración de producción desactive por defecto las rutas internas y Swagger, y exija autenticación.
8. Se incorporó un health check y se probaron endpoints principales dentro del contenedor.

## 4) Validación realizada

Durante la revisión del módulo se validó el comportamiento real del sistema con requests concretos a la API, cubriendo casos de:

- carga correcta de documento de usuario
- archivo faltante
- tipo de archivo no permitido
- tipo de documento inválido
- entidad no encontrada
- comprobante asociado a orden/entrega

También se comprobó que la API mantenga el formato estándar de errores, que el health check no exponga secretos y que el contenedor responda en health, Swagger y un endpoint principal.

## 5) Testing y validación

La suite funcional actual cuenta con **53 pruebas aprobadas**. Se usan Mocha, Chai y Supertest para probar las rutas sin iniciar un servidor HTTP externo; las pruebas funcionales se conectan a una base MongoDB de prueba.

| Módulo | Tests | Casos principales |
|---|---:|---|
| Módulo | Pruebas | Casos principales |
|---|---:|---|
| Usuarios | 11 | CRUD, permisos, hash de contraseña, filtros y paginación |
| Pedidos | 8 | CRUD, estados, comprobantes, filtros y paginación |
| Productos | 1 | Paginación y límite máximo |
| Cargas de archivos | 6 | Carga válida, límites, tipos, duplicados y entidad inexistente |
| Entregas | 8 | Estados, sincronización con pedidos, conflictos y paginación |
| Mocks | 6 | Generación, seed y snapshots |
| Logger / Swagger / health / 404 | 6 | Rutas técnicas, documentación, health check y errores |
| Middleware de autenticación | 7 | JWT, roles y respuestas de autorización |
| **Total** | **53** | **Suite funcional** |

Al comenzar el Módulo 7, los tests de uploads cubrían los casos principales de carga correcta, archivo faltante, tipo inválido y entidad inexistente. En la Entrega 8 la cobertura se amplió para incluir restricciones adicionales y duplicados.

### Ejecución

```bash
npm test
```

Requisitos:
- Node.js 20+
- MongoDB activo y accesible mediante `MONGO_URI_TEST` (o `MONGODB_URI_TEST`)
- Variables definidas en `.env.test` o en el entorno

En la última validación de la entrega se obtuvieron **53 pruebas aprobadas**. También se construyó la imagen Docker y se comprobaron el health check, Swagger y el listado de usuarios.

## 6) Riesgos, deuda técnica y próximos pasos

Luego de la Entrega 8, identifico las siguientes limitaciones y mejoras:

**Decisiones que hice con conocimiento de riesgos**:

1. **Archivos locales y archivos huérfanos**
   - Riesgo: el almacenamiento local puede perderse si el volumen no tiene respaldo. Si falla la asociación después de la carga, puede quedar un archivo sin referencia.
   - Mitigación actual: `UPLOAD_BASE_DIR` queda fuera del repositorio y Compose usa un volumen.
   - Mejora pendiente: definir retención, limpieza automática y respaldo; para producción real, evaluar almacenamiento de objetos.

2. **Lecturas públicas**
   - Riesgo: los endpoints GET no requieren autenticación en esta entrega.
   - Decisión: se conserva el comportamiento existente para el alcance educativo; en producción real debe revisarse la autorización por recurso.

3. **Validación del contenido de archivos**
   - Riesgo: MIME type y extensión no garantizan por sí solos que el contenido sea seguro.
   - Mejora futura: inspección del contenido y análisis antivirus antes de asociar el archivo.

4. **Retención y observabilidad**
   - Los logs rotativos locales no reemplazan un sistema centralizado.
   - Mejora futura: centralizar logs y configurar alertas en el entorno de despliegue.

5. **MongoDB con transacciones**
   - En producción se requiere replica set o mongos por defecto.
   - Una instancia standalone no iniciará con la configuración productiva.
   - Compose usa MongoDB standalone solo para desarrollo local y desactiva el requisito de transacciones.

## 7) Cierre del Módulo 7 (registro histórico)

El proyecto quedó en un estado consistente entre implementación, documentación y criterio profesional. Todas las decisiones fueron documentadas para que se entienda por qué se eligió cada camino.

### Reflexiones y decisiones clave tomadas

**Separación de responsabilidades**:
- `uploadFileMiddleware.js`: valida archivo (Multer level)
- `userService.js`: valida entidad + asocia (Business logic)
- `errorHandler.js`: normaliza error (HTTP response)
- Esta separación hace que sea fácil reutilizar o cambiar

### Prioridades mantenidas

- ✅ corrección técnica
- ✅ claridad de diseño  
- ✅ trazabilidad operativa
- ✅ almacenamiento mínimo y seguro
- ✅ documentación estudiante-first (explicar decisiones, no solo qué)
- documentación útil para integración y revisión
- escalabilidad para futuras entidades y validaciones

## 8) Cierre de la Entrega 8

La API conserva sus endpoints principales y suma límites explícitos para listados y cargas, configuración validada al inicio, un health check y una imagen Docker ejecutada como usuario no-root. La configuración de producción exige `JWT_SECRET` y desactiva por defecto mocks, logger test y Swagger. El puerto interno del contenedor es `3003`.

### Checklist de entrega

- [x] Listados de usuarios, pedidos, entregas y productos con máximo de 100 resultados por página.
- [x] Uploads limitados a 5 MB y a PDF, PNG o JPEG; los errores de carga tienen respuestas controladas.
- [x] Variables críticas validadas al iniciar y separadas mediante `.env.example`, `.env.test.example` y variables del entorno productivo.
- [x] Health check en `/api/health` sin URI de base de datos ni secretos.
- [x] `Dockerfile`, `.dockerignore` y `compose.yaml` disponibles.
- [x] README con ejecución local, pruebas, Swagger, variables y ejecución del contenedor.
- [x] `npm test`: 53 pruebas funcionales aprobadas.
- [x] Prueba de contenedor: health, Swagger y listado principal disponibles.

### Aprendizaje

En esta etapa comprobé que preparar una API para despliegue no consiste solo en crear una imagen. También hay que limitar las consultas y los archivos, separar los valores por entorno, evitar copiar secretos al contexto Docker y documentar qué datos persisten. Como siguiente mejora técnica queda automatizar la limpieza de uploads huérfanos y definir formalmente su retención.
