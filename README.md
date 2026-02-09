### Paso 1: El `Dockerfile` (Construcción Multi-etapa)

El primer archivo es un `Dockerfile` que utiliza una técnica llamada **Multi-stage build** (construcción en múltiples etapas). Esto es una mejor práctica para crear imágenes ligeras y seguras.

#### Etapa 1: Construcción (`build`)

El objetivo de esta parte es compilar el código fuente y generar el archivo ejecutable (`.jar`).

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /build
COPY ./pom.xml ./
COPY ./src ./src
RUN mvn clean package -DskipTests

```

1. **`FROM maven... AS build`**:
* Descarga una imagen base que tiene **Maven** y **Java 21** instalados.
* La etiqueta `AS build` nombra esta etapa temporalmente para poder referenciarla después.


2. **`WORKDIR /build`**: Crea y define el directorio de trabajo dentro del contenedor.
3. **`COPY ...`**: Copia el archivo de configuración de Maven (`pom.xml`) y el código fuente (`src`) desde tu máquina al contenedor.
4. **`RUN mvn clean package ...`**: Ejecuta el comando de Maven para compilar el código y empaquetarlo en un archivo `.jar`.
* `-DskipTests`: Omite la ejecución de tests unitarios para acelerar la construcción (útil en entornos rápidos, aunque riesgoso para producción).



#### Etapa 2: Ejecución (Runtime)

El objetivo de esta parte es tomar el `.jar` creado anteriormente y ejecutarlo en una imagen limpia y ligera.

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /build/target/*.jar app.jar
EXPOSE 80
CMD ["java", "-jar", "app.jar"]

```

1. **`FROM eclipse-temurin:21-jre`**:
* Inicia una nueva imagen base **solo con el JRE** (Java Runtime Environment). Esta imagen es mucho más pequeña que la de Maven porque no incluye herramientas de compilación.


2. **`COPY --from=build ...`**:
* Esta es la magia. Copia **solo** el archivo `.jar` generado en la etapa anterior (`/build/target/...`) hacia la nueva imagen (`app.jar`). El resto de archivos de código fuente y caché de Maven se descartan.


3. **`EXPOSE 80`**: Documenta que el contenedor escuchará en el puerto 80 (nota: tu aplicación Java debe estar configurada para usar el puerto 80, ya que Spring Boot usa el 8080 por defecto).
4. **`CMD ...`**: Define el comando que se ejecutará al iniciar el contenedor: `java -jar app.jar`.

---

### Paso 2: El `docker-compose.yml` (Orquestación)

Este archivo simplifica la ejecución. En lugar de escribir comandos largos de Docker, defines la configuración aquí.

```yaml
version: "3.8"
services:
  app: 
    image: spring-boot-app-image
    build: .
    ports:
      - "80:80"

```

1. **`services`**: Define los contenedores que se van a ejecutar. Aquí solo tienes uno llamado `app`.
2. **`build: .`**: Le dice a Docker Compose: "Busca un `Dockerfile` en el directorio actual (`.`) y construye la imagen".
3. **`image: spring-boot-app-image`**: Asigna un nombre a la imagen resultante una vez construida.
4. **`ports: "80:80"`**: Mapeo de puertos.
* El **80 de la izquierda** es el puerto de tu máquina (localhost).
* El **80 de la derecha** es el puerto dentro del contenedor.
* Resultado: Podrás acceder a tu app escribiendo `http://localhost` en tu navegador.



---

### Paso 3: El Comando (`Shell`)

Finalmente, ejecutas la acción.

```bash
docker compose up

```

Este comando realiza las siguientes acciones en secuencia:

1. **Lee** el archivo `docker-compose.yml`.
2. **Construye** la imagen basándose en el `Dockerfile` (compila el Java, crea el JAR, crea la imagen final).
3. **Crea y arranca** el contenedor con la configuración de puertos especificada.
4. Muestra los **logs** de la aplicación en la terminal.

---

### Resumen Visual del Flujo

1. **Código Fuente** ➔ `docker compose up` inicia el proceso.
2. **Maven Image** ➔ Compila el código y crea el `.jar`.
3. **JRE Image** ➔ Recibe el `.jar` y descarta las herramientas de compilación.
4. **Contenedor** ➔ Se inicia y expone el puerto 80.
5. **Usuario** ➔ Accede a la aplicación vía `localhost:80`.

---
## Configurar el canal de acciones de GitHub para crear la imagen de Docker

Configurar la automatización con GitHub Actions es el paso definitivo para pasar de un desarrollo manual a una mentalidad de **Integración Continua (CI)**. Aquí tienes el desglose técnico de cómo conectar tu código local con la nube de GitHub y qué sucede exactamente cuando "la magia" empieza a correr.

---

## 1. Preparación del Repositorio Local (Git)

Antes de automatizar, necesitamos que GitHub conozca nuestro código. Estos comandos inicializan el control de versiones y vinculan tu carpeta local con el servidor remoto.

| Comando | Acción y Significado Técnico |
| --- | --- |
| `git init` | Crea un nuevo subdirectorio `.git` que contiene el esqueleto de un repositorio de Git. Empieza el rastreo de cambios. |
| `git remote add origin <url>` | Establece una conexión entre tu repo local y el repo en GitHub. `origin` es el nombre estándar para este vínculo. |
| `git add .` | Mueve todos los archivos del directorio actual al **Staging Area** (el área de preparación antes del commit). |
| `git commit -m "msg"` | Captura una instantánea del proyecto. El mensaje debe ser descriptivo para auditorías futuras. |
| `git branch -M main` | Renombra la rama principal a `main`. Es el estándar actual de la industria (reemplazando al antiguo `master`). |
| `git push -u origin main` | Sube tus archivos. El flag `-u` vincula tu rama local con la remota, facilitando futuros comandos `git push`. |

---

## 2. Anatomía del Workflow: `.github/workflows/pipelines.yml`

Este archivo es la "receta" que GitHub Actions seguirá cada vez que detecte actividad.

### Los Disparadores (Triggers)

* **`on: push: branches: - main`**: El pipeline se activa automáticamente cada vez que alguien hace un "push" a la rama principal.
* **`workflow_dispatch`**: Este es un botón "manual". Permite que vayas a la pestaña de Actions en la web de GitHub y ejecutes el proceso sin necesidad de cambiar el código.

### La Infraestructura (Jobs)

* **`runs-on: ubuntu-latest`**: GitHub provisiona una **Máquina Virtual (Runner)** basada en Ubuntu. Es un entorno limpio y aislado donde se ejecutará tu código.

### Los Pasos (Steps)

1. **`uses: actions/checkout@v4`**: Como la máquina virtual empieza vacía, este paso clona tu repositorio dentro del Runner para que el pipeline tenga acceso a tu `Dockerfile` y a tu código Java.
2. **`run: docker build -t spring-boot-app:latest .`**: Aquí es donde ocurre el trabajo pesado. Se ejecuta el motor de Docker dentro del Runner para construir la imagen. El tag `-t` le pone nombre a la imagen para que sea identificable.

---

## 3. ¿Qué ocurre internamente en GitHub Actions?

Cuando ejecutas el último `git push`, se desencadena una serie de eventos en los servidores de GitHub que puedes monitorear en tiempo real:

1. **Activación del Evento**: GitHub recibe los archivos y detecta que en la carpeta `.github/workflows/` hay un archivo YAML válido.
2. **Asignación de Runner**: GitHub busca un servidor disponible (Runner). Si es un repo público, esto es gratuito; si es privado, consume minutos de tu cuota mensual.
3. **Ejecución Secuencial**:
* **Setup**: Se prepara el sistema operativo (Ubuntu).
* **Checkout**: El Runner descarga tu código.
* **Build**: Se inicia el proceso que explicamos anteriormente (Maven compila, se genera el JAR, y Docker empaqueta todo en una imagen).


4. **Reporte de Estado**:
* Si todos los comandos devuelven un código de salida `0`, verás un **check verde (✅)** en tu commit.
* Si algo falla (ej. un error de sintaxis en el Dockerfile o un test fallido), verás una **X roja (❌)**. GitHub te enviará un correo avisándote que el "build" se rompió.

---

Llevar tus imágenes a la nube es el siguiente gran paso. Ya no solo estamos construyendo la aplicación, sino que la estamos dejando lista en un "almacén" privado para que cualquier servidor del mundo (o de AWS) pueda descargarla y ejecutarla.

Aquí tienes la explicación detallada de esta integración.

---

## #1 ¿Qué es AWS ECR y cómo ayuda en CI/CD?

**ECR (Elastic Container Registry)** es el servicio de Amazon diseñado para almacenar, gestionar y desplegar imágenes de contenedores Docker de forma segura y escalable.

### ¿Por qué lo necesitamos en CI/CD?

En un flujo profesional, no guardas las imágenes en tu computadora. El flujo funciona así:

1. **Integración Continua (CI):** GitHub Actions construye la imagen y la prueba.
2. **Almacenamiento:** GitHub Actions envía esa imagen a **ECR**.
3. **Despliegue Continuo (CD):** Servicios como AWS ECS o EKS detectan que hay una nueva imagen en ECR y actualizan tu aplicación automáticamente.

**Ventajas clave:**

* **Seguridad:** Solo personas o servicios autorizados pueden "puchar" o "pullear" imágenes.
* **Velocidad:** Al estar dentro de la red de AWS, la descarga de imágenes hacia tus servidores es casi instantánea.
* **Disponibilidad:** Tus imágenes están replicadas; si un servidor falla, la imagen siempre está disponible para levantar uno nuevo.

---

## #2 Desglose del Pipeline (Explicación paso a paso)

Este fragmento de YAML utiliza **Actions oficiales de AWS** para manejar la seguridad y la transferencia de datos.

| Paso | Acción Técnica | ¿Por qué es importante? |
| --- | --- | --- |
| **Checkout code** | Descarga tu código en el Runner. | Sin esto, el Runner no tiene acceso al `Dockerfile`. |
| **Configure AWS Credentials** | Usa las claves (`Access Key` e `ID`) para identificarte ante AWS. | Permite que GitHub "hable" con tu cuenta de AWS de forma segura usando **Secrets**. |
| **Log in to Amazon ECR** | Ejecuta un `docker login` especial para AWS. | Docker necesita una contraseña temporal para poder subir archivos a los servidores de Amazon. |
| **Build Docker image** | Construye la imagen con el tag del repositorio. | Genera el artefacto final usando tu `Dockerfile`. |
| **Push Docker image** | Sube la imagen a la nube de AWS. | Envía los gigas/megas de tu imagen al registro ECR. |

---

### Un punto crítico: Los "Secrets"

Habrás notado el uso de `${{ secrets.VARIABLE }}`. Esto es **obligatorio** por seguridad:

* **Nunca** pongas tus credenciales de AWS directamente en el código YAML.
* Debes ir a tu repositorio en GitHub: **Settings > Secrets and variables > Actions** y añadir allí:
* `AWS_ACCESS_KEY_ID`
* `AWS_SECRET_ACCESS_KEY`
* `AWS_REGION` (ej. `us-east-1`)
* `ECR_REPO` (la URL de tu repositorio en AWS, algo como `123456789.dkr.ecr.us-east-1.amazonaws.com/mi-app`)



---

## ¿Qué ocurre en el panel de AWS?

Una vez que el pipeline de GitHub Actions termina con éxito (check verde):

1. Vas a la consola de **AWS ECR**.
2. Entras en tu repositorio.
3. Verás una nueva línea con el tag `latest`, el tamaño de la imagen y la fecha exacta de subida.

# Como crear usuario IAM

Aquí tienes los pasos para crear un usuario IAM en AWS:

1. **Acceder a la Consola de AWS**:
   - Inicia sesión en tu cuenta de AWS y ve a la consola de administración.

2. **Navegar a IAM**:
   - En la barra de búsqueda, escribe "IAM" y selecciona "IAM" para abrir el servicio de gestión de identidades y accesos.

3. **Seleccionar 'Users'**:
   - En el menú de la izquierda, haz clic en "Users".

4. **Crear un nuevo usuario**:
   - Haz clic en el botón "Add user" (Agregar usuario).

5. **Configurar el nombre del usuario**:
   - Ingresa un nombre para el usuario (por ejemplo, `gitHub-actions-user`).

6. **Seleccionar en el usuario creado el tipo de acceso**:
   - Marca la opción "CLI acceso" para permitir el acceso a través de la API.   
   - Copia Access Key y Secret Access Key

7. **Configurar permisos**:
   - En la sección de permisos, puedes elegir "Attach existing policies directly" (Adjuntar políticas existentes directamente) y seleccionar la política "AmazonEC2ContainerRegistryFullAccess" para otorgar acceso completo al ECR.

8. **Revisar y crear**:
   - Revisa la configuración y haz clic en "Create user" (Crear usuario).
