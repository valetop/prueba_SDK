# Variables de Configuración — Proyecto miniPOS (Wompi + TMS)

Este documento describe **todas las variables necesarias** para que el pipeline CI/CD de miniPOS funcione correctamente con **Azure DevOps + TMS (Topwise)**.

La configuración está separada por:
- Variable Groups (Library)
- Variables runtime (generadas en ejecución)
- Environments de Azure DevOps


---

### Artefacto de build
- **`APP_BIN`**  
  Nombre del binario generado por el SDK  
   `app.bin` cambiar por el que necesitemos 

- **`ARTIFACT_NAME`**  
  Nombre del artefacto publicado por el pipeline  
   `app-bin` cambiar por el que necesitemos

---

## Manejo de variables

La configuración se divide en tres categorías:

### Variable Group: `tms-config` (NO secretas, o poner otro nombre)
- Identidad de la app (`TMS_APP_NAME`) : cambiar por lo que se necesite
- Modelo POS (`TMS_MODEL_NAME = M3S`) : valor M3S
- Artefacto (`APP_BIN`, `ARTIFACT_NAME`)
- Endpoints del TMS (`TMS_BASE_URL`, `TMS_LOGIN_PATH`, etc.)

### Variable Groups secretos (por ambiente) Configurar:
- `tms-dev-secrets`
- `tms-uat-secrets`
- `tms-pdn-secrets`

Cada uno contiene:
- `TMS_USERNAME`
- `TMS_PASSWORD_MD5` : encriptarla para que funcione

> Aunque actualmente las credenciales sean las mismas, se deben separar por ambiente por control y escalabilidad.

### Variables runtime  - se guardan de la ejecucion del pipeline
- `TMS_TOKEN`
- `APP_BIN_SHA256`
- `TMS_APP_ID`
- `BUILD_VERSION`

---

## Environments en Azure DevOps

Se utilizan los siguientes **Environments**:
- `dev`
- `uat`
- `prod`

Configuración que tienen:
- DEV → sin aprobación
- UAT → probación manual obligatoria
- PROD → aprobación manual obligatoria

---

## Configuración obligatoria en Azure DevOps

### Variable Groups (Library)

#### Variable Group: `tms-config` o poner otro nombre (NO secretas) 

Este grupo contiene **configuración común**.  
No debe contener contraseñas o credenciales .


| `TMS_APP_NAME` : `wompi-minipos` cambiar por el nombre que se requiera |
| `TMS_MODEL_NAME` : `M3` |
| `APP_BIN` : `app.bin` cambiar por el nombre que se requiera |
| `ARTIFACT_NAME` : `app-bin` cambiar por equivalente al app.bin|
| `TMS_BASE_URL` : `https://tms.topwisesz.com/topwiseapi` |
| `TMS_LOGIN_PATH` : `/tms/openapi/login` |
| `TMS_ADD_APP_PATH` : `/tms/openapi/app/addApp` |
| `TMS_QUERY_PATH` : `/tms/openapi/app/queryPageList` |
| `TMS_PUBLISH_PATH` : `/tms/openapi/app/upAndDownApp` |

---

#### Variable Groups Secretos (uno por ambiente)


##### `tms-dev-secrets`
- `TMS_USERNAME` (secret) - recordar poner el password encriptado
- `TMS_PASSWORD_MD5` (secret) - recordar poner el password encriptado

##### `tms-uat-secrets`
- `TMS_USERNAME` (secret) - recordar poner el password encriptado
- `TMS_PASSWORD_MD5` (secret) - recordar poner el password encriptado

##### `tms-pdn-secrets`
- `TMS_USERNAME` (secret) - recordar poner el password encriptado
- `TMS_PASSWORD_MD5` (secret) - recordar poner el password encriptado


---

## Detalles importantes del pipeline

### Build
- Se ejecuta sobre `windows-latest` porque `build.bat` requiere Windows.
- Usa **Node.js 18.x** por estabilidad.
  - Si el SDK presenta incompatibilidades, se puede cambiar fácilmente a **Node 16.x** (la línea ya está comentada en el template).
- El repositorio se limpia (`checkout clean: true`) para evitar artefactos residuales.

### Validación del binario
Antes de publicar el artefacto:
- se valida que `app.bin` exista,
- que tenga un tamaño mínimo razonable (>= 10 KB),
- se calcula SHA256 para trazabilidad.

El hash se guarda como variable runtime `APP_BIN_SHA256`.

---

### Deploy a TMS
Cada deploy:
1. Hace login al TMS
2. Guarda el token como variable secreta `TMS_TOKEN`
3. Sube el binario usando `addApp`
4. Obtiene el `appId` (directamente o vía `queryPageList`)
5. Publica OTA usando `upAndDownApp`

La diferencia entre DEV / UAT / PDN se controla por parámetros:

| Env | pushRange | otaType    |
|-----|-----------|------------|
| DEV | 3 (group) | 0 (manual) |
| UAT | 3 (group) | 0 (manual) |
| PDN | 1 (all)   | 1 (forced) |

## Resumen parametros importantes

- id 
- type: 
	1 → on shelves
	2 → off shelves
- pushRange: 
	1 → todos
	2 → dispositivos específicos
	3 → grupos
- otaType: 
	0 → manual
	1 → forzada


---

## Variables runtime (NO se configuran manualmente)

Estas variables se generan durante la ejecución del pipeline:

- `TMS_TOKEN`
- `TMS_APP_ID`
- `APP_BIN_SHA256`
- `Build.BuildId` / tag `v*`

No deben crearse en Variable Groups, son automaticas.

---

## Cómo ejecutar el pipeline

### Para DEV / UAT
- Hacer push a `main`
- El pipeline corre automáticamente hasta UAT

### Para PDN
1. Crear un tag siguiendo la convención:

v1.0.0
2. Hacer push del tag
3. Aprobar manualmente el Environment `PROD`



---

## Notas
Este es un resumen de lo que se debe hacer para que **todo el código planteado funcione correctamente**.
- El pipeline **no recompila** en los deploys: siempre utiliza el mismo artefacto.
- El pipeline valida que la app quede **on shelves** en TMS.
- La confirmación de instalación en cada POS se revisa desde el **portal TMS** (App Task / Device Details).
- Esto es una limitación del API de TMS ya que no hay endpoint para esto
