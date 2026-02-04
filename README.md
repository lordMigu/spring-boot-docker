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
