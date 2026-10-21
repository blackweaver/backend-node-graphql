# backend-node-graphql

Backend de práctica del curso de Node.js, construido con Express y GraphQL. Permite registrar usuarios, iniciar sesión con JWT y administrar publicaciones con título, contenido e imagen. Los usuarios y las publicaciones se guardan en MongoDB mediante Mongoose.

## Funcionalidades

- Registro de usuarios con contraseñas protegidas mediante hashes de bcrypt.
- Inicio de sesión y autenticación mediante tokens JWT con una duración de una hora.
- Creación, consulta, edición y eliminación de publicaciones. Solo el autor puede editar o eliminar sus publicaciones.
- Listado de publicaciones con paginación de dos resultados por página.
- Consulta del perfil y actualización del estado del usuario.
- Carga de imágenes PNG y JPEG.
- Interfaz GraphiQL para explorar y probar la API desde el navegador.

## Requisitos

- Node.js y npm instalados.
- Docker instalado y en ejecución (por ejemplo, Docker Desktop).
- Puertos `8080` (API) y `27017` (MongoDB) disponibles.

Los siguientes pasos ejecutan la API en tu máquina y MongoDB dentro de un contenedor Docker. Los comandos están escritos para una terminal Bash o Zsh.

## Paso a paso para correr el proyecto

### 1. Instalar las dependencias

Desde la carpeta del proyecto:

```bash
npm ci --legacy-peer-deps
```

Se utiliza `--legacy-peer-deps` porque la versión de `express-graphql` del proyecto declara compatibilidad con versiones de GraphQL anteriores a la instalada. Si no tenés el archivo `package-lock.json`, usá `npm install --legacy-peer-deps`.

### 2. Iniciar MongoDB con Docker

Creá un volumen para persistir los datos y levantá el contenedor:

```bash
docker volume create backend-node-graphql-mongo-data

docker run -d \
  --name backend-node-graphql-mongo \
  -p 127.0.0.1:27017:27017 \
  -v backend-node-graphql-mongo-data:/data/db \
  mongo:7.0
```

El contenedor se llama `backend-node-graphql-mongo`, publica MongoDB en `127.0.0.1:27017` y guarda los datos en el volumen `backend-node-graphql-mongo-data`. Esta configuración local no requiere usuario ni contraseña y limita el puerto publicado a tu máquina. El uso de la imagen y del volumen sigue la [documentación oficial de la imagen de MongoDB](https://hub.docker.com/_/mongo).

Verificá que el contenedor esté en ejecución:

```bash
docker ps --filter name=backend-node-graphql-mongo
docker exec backend-node-graphql-mongo mongosh --quiet --eval 'db.adminCommand({ ping: 1 })'
```

Esperá a que el segundo comando devuelva `ok: 1` antes de iniciar la API. Si MongoDB todavía está arrancando, volvé a ejecutarlo unos segundos después.

### 3. Configurar las variables de entorno

Creá un archivo `.env` en la raíz del proyecto, junto a `app.js`, o agregá esta variable al que ya tengas:

```dotenv
MONGO_LOCAL_URI=mongodb://127.0.0.1:27017/backend-node-graphql
```

`MONGO_LOCAL_URI` es la URI completa de conexión y tiene prioridad sobre la conexión a MongoDB Atlas configurada en `app.js`. Para usar MongoDB local no necesitás configurar `MONGO_URI`: actualmente esa variable se utiliza como contraseña de la conexión alternativa a Atlas.

La base `backend-node-graphql` se crea al guardar los primeros datos. El archivo `.env` está excluido de Git mediante `.gitignore`.

### 4. Crear la carpeta de imágenes

Multer guarda los archivos subidos en la carpeta `images/`, que debe existir antes de cargar una imagen:

```bash
mkdir -p images
```

### 5. Iniciar la API

```bash
npm start
```

Este comando ejecuta `nodemon app.js` y reinicia el servidor al modificar el código. La API escucha en el puerto **8080**, definido en `app.js`, una vez que logra conectarse a MongoDB. Para detenerla, presioná `Ctrl+C`.

### 6. Comprobar que funciona

Abrí [GraphiQL](http://localhost:8080/graphql) y ejecutá:

```graphql
query {
  __typename
}
```

La respuesta esperada es:

```json
{
  "data": {
    "__typename": "RootQuery"
  }
}
```

También podés comprobarlo desde otra terminal:

```bash
curl -X POST http://localhost:8080/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ __typename }"}'
```

## Uso de la API

El endpoint de GraphQL es `http://localhost:8080/graphql`. En GraphiQL podés consultar el esquema y los argumentos de cada operación.

- **Sin autenticación:** la mutación `createUser` y la consulta `login`.
- **Con autenticación:** las consultas `posts`, `post` y `user`, y las mutaciones `createPost`, `updatePost`, `deletePost` y `updateStatus`.

Después de iniciar sesión, enviá el token devuelto por `login` en cada petición protegida:

```http
Authorization: Bearer <token>
```

Para subir una imagen, enviá una petición autenticada `PUT http://localhost:8080/post-image` con un cuerpo `multipart/form-data` y el archivo en el campo `image`. La respuesta incluye `filePath`, que podés usar como `imageUrl` al crear o actualizar una publicación. Las imágenes quedan disponibles en `http://localhost:8080/images/<nombre-del-archivo>`.

## Comandos útiles de Docker

Ver los logs de MongoDB:

```bash
docker logs -f backend-node-graphql-mongo
```

Detener el contenedor:

```bash
docker stop backend-node-graphql-mongo
```

Volver a iniciarlo en otra sesión, conservando los datos:

```bash
docker start backend-node-graphql-mongo
```

El comando `docker run` del paso inicial se usa una sola vez. En las siguientes sesiones, ejecutá `docker start backend-node-graphql-mongo` y luego `npm start` desde la carpeta del proyecto. Los datos se conservan en el volumen aunque se elimine el contenedor.

## Problemas frecuentes

- **Docker no responde:** comprobá que Docker Desktop o el servicio Docker esté iniciado.
- **El nombre del contenedor ya existe:** usá `docker start backend-node-graphql-mongo` para iniciar el contenedor creado previamente.
- **El puerto 27017 está ocupado:** cambiá el mapeo a `-p 127.0.0.1:27018:27017` al crear el contenedor y usá el puerto `27018` en `MONGO_LOCAL_URI`.
- **La API no responde en el puerto 8080:** revisá los errores de conexión en la terminal y verificá que MongoDB esté disponible. El servidor HTTP solo arranca después de conectarse a la base.
- **Error al guardar una imagen:** verificá que exista `images/` y ejecutá la API desde la raíz del proyecto.

## Estructura principal

```text
app.js              Configuración de Express, rutas y conexión a MongoDB
graphql/schema.js   Tipos, consultas y mutaciones de GraphQL
graphql/resolvers.js Lógica de las operaciones de GraphQL
middleware/auth.js  Validación del token JWT
models/             Modelos de usuarios y publicaciones
util/file.js        Eliminación de imágenes
images/             Archivos subidos; se crea durante la configuración
```

El proyecto todavía no tiene pruebas automatizadas configuradas: `npm test` devuelve un error indicando que no hay tests definidos.
