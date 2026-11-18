# Festival Check-in

Proyecto de aula listo para clonar. Crearás los recursos en AWS Academy, cargarás un evento y un perfil ficticio y comprobarás el pase por HTTP.

**Para empezar en clase, sigue [0-conf.md](0-conf.md) de principio a fin.** Si trabajas en el navegador, abre [0-codespaces.md](0-codespaces.md).

## Recorrido por casos de uso

Usa el mismo evento y usuario de principio a fin. En cada caso, abre primero `serverless.yml`, sigue las llamadas y ejecuta la comprobación. Usa el cuaderno web para seguir las explicaciones visuales y el Markdown del repositorio para ejecutar los comandos de tu copia de `festival-checkin-01`.

| Caso de uso | Guía del repositorio | Explicación en el cuaderno web |
| --- | --- | --- |
| Preparar AWS y desplegar | [Configurar desde cero](0-conf.md) | [Preparar AWS Academy](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/#iniciar-lab) |
| Guardar un evento por capas con DynamoDB Toolbox | [Crear un evento](1-crear-evento.md) | [Crear con ORM](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/#capas-eventos-orm) · [Infografía de Crear un evento](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/traza-evento/#infografia) |
| Buscar el evento guardado por su ID | [Invocar buscarEvento](1-crear-evento.md#busca-el-evento-desde-lambda) | [Invoke · Buscar evento](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/#invoke-buscar-evento) |
| Preparar el perfil que se inscribirá | [Crear un usuario](2-crear-usuario.md) | [Invocar crearUsuario](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/#sembrar-usuarios) |
| Leer el evento y observar el pase antes de inscribir | [Consultar una inscripción](3-consultar-inscripcion.md) | [Consultar inscripción con ORM](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/#consultar-orm) |
| Crear la relación, comprobar el 409 y leer la fecha | [Inscribir a un usuario](4-inscribir-usuario.md) | [Inscribir con ORM](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/#inscribir-orm) |

[Abre las pestañas de invocación con Serverless](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/#invocaciones) para ejecutar cada función del clon. `--function` usa la clave de `functions` en tu `serverless.yml` y `--data` contiene el JSON que recibe el handler. En [Invoke · Listar eventos](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/#invoke-listar-eventos) puedes comprobar el alta y leer las páginas del listado.

[Abre la chuleta de 12 lecturas](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/consultas-dynamodb/) para comparar GetItem, Query y Scan con las tres tablas del festival.

## 1. Clona y comprueba el proyecto

Usa Git Bash en Windows o Terminal en macOS. Necesitas Git, Node.js 20 o superior y AWS CLI v2. Usa la versión compatible que ya tengas instalada. Lambda ejecuta Node 22 en AWS.

```bash
git clone https://github.com/albertohdezakkari/festival-checkin-01.git
cd festival-checkin-01
node --version
aws --version
npm ci
npm ls serverless --depth=0
```

`npm ci` instala las versiones fijadas en `package-lock.json`. Cuando termine la instalación, el segundo comando lista Serverless como dependencia directa y debe mostrar `serverless@3.40.0`. [Cuándo usar cada comando](0-conf.md#2-clona-e-instala-el-proyecto)

## 2. Configura tu sesión

En AWS Academy, inicia **Learner Lab → Start Lab → AWS Details → AWS CLI**. Guarda las tres credenciales temporales en un perfil de tu archivo personal `~/.aws/credentials`. Usa la cabecera `[academy]` en minúsculas para el perfil del aula. Sigue estos pasos del [cuaderno de preparación](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/#iniciar-lab).

Desde la raíz del proyecto, cambia los valores de ejemplo por los de tu laboratorio. Cada equipo que comparta cuenta y región debe usar un código diferente de hasta 20 caracteres, en minúsculas, con números o guiones y empezando por una letra.

```bash
export AWS_PROFILE=academy
export AWS_REGION=us-east-1
export FCI_EQUIPO=equipo01
aws sts get-caller-identity --profile "$AWS_PROFILE" --region "$AWS_REGION"
```

Si guardaste las credenciales en `[default]`, exporta `AWS_PROFILE=default`. Si STS muestra `ExpiredToken`, renueva en AWS Details los tres valores del mismo perfil.

```bash
LAMBDA_ROLE_ARN=$(aws iam get-role --role-name LabRole \
  --profile "$AWS_PROFILE" --region "$AWS_REGION" --query 'Role.Arn' --output text)
printf '%s\n' "$LAMBDA_ROLE_ARN"
```

Si aparece el ARN de `LabRole`, expórtalo:

```bash
export LAMBDA_ROLE_ARN
```

Si el laboratorio deniega `iam:GetRole`, exporta `LAMBDA_ROLE_ARN` con el ARN completo de LabRole que te facilite el profesor.

## 3. Despliega y prueba

En la misma terminal, desde la raíz del repositorio. Si abriste otra, recupera las cuatro variables del [paso 4 de 0-conf.md](0-conf.md#4-configura-y-comprueba-tu-despliegue).

```bash
npm run deploy -- --aws-profile "$AWS_PROFILE"
```

Cuando Serverless confirme el despliegue, ejecuta:

```bash
npm run aula
```

Para una instalación nueva, verás **GET 200 → POST 201 → POST 409 → GET 200**. El comando carga el evento 1 y a `usuario-02` y muestra la URL de la API al final. Si repites la prueba, ya existe el mismo pase y el POST devuelve 409.

Abre la [pantalla del festival](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/#pantalla), elige **AWS**, pega la URL y consulta el evento **1** para **usuario-02**. El pase del usuario figura como **Inscrito**.

## Prueba la API con curl

Tras desplegar, recupera la URL base desde la terminal del proyecto:

```bash
api_url="$(aws cloudformation describe-stacks \
  --stack-name "festival-check-in-${FCI_EQUIPO}" \
  --profile "$AWS_PROFILE" --region "$AWS_REGION" \
  --query "Stacks[0].Outputs[?OutputKey=='HttpApiUrl'].OutputValue | [0]" --output text)"
printf '%s\n' "$api_url"
```

Prueba las rutas con los ejemplos `curl` que hay en cada handler. El mismo método, URL y JSON servirán al hacer la petición desde React Native.

| Operación | Ruta HTTP | Datos que lee el handler |
| --- | --- | --- |
| [Crear evento](src/eventos/crear-evento/handler.mjs) | `POST /events` | `eventId` y `title` del JSON |
| [Listar eventos](src/eventos/listar-eventos/handler.mjs) | `GET /events` | `cursor` opcional en la consulta |
| [Buscar evento](src/eventos/buscar-evento/handler.mjs) | `GET /events/{eventId}` | `eventId` de la ruta |
| [Crear usuario](src/usuarios/crear-usuario/handler.mjs) | `POST /users` | Perfil del usuario en el JSON |
| [Inscribir usuario](src/inscripciones/inscribir-usuario/handler.mjs) | `POST /events/{eventId}/registrations` | `eventId` de la ruta y `userId` del JSON |
| [Consultar inscripción](src/inscripciones/consultar-inscripcion/handler.mjs) | `GET /events/{eventId}/registrations/{userId}` | Ambos IDs de la ruta |

[Ver las llamadas en el cuaderno](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/#invocaciones)

## 4. Sigue el código desde la petición hasta DynamoDB

En `src/eventos/crear-evento/` están los archivos del alta. En `src/eventos/buscar-evento/` están el handler para invocar la búsqueda directamente y el service y repository que también usan **Consultar inscripción** e **Inscribir usuario**. El modelo que comparten el alta y la lectura vive en `src/eventos/models/`. En `src/inscripciones/` están los casos **Consultar inscripción** e **Inscribir usuario**, que trabajan con `FCI_INSCRIPCIONES`. Cada handler llama directamente al service de su caso de uso. Los handlers HTTP usan `http/peticion.mjs` para leer la clave y convertir el resultado en una respuesta para API Gateway. Su modelo de DynamoDB Toolbox está en `models/`.

| Abre | Qué buscar |
| --- | --- |
| `serverless.yml` | Las seis funciones, las rutas HTTP, las tres tablas y el índice `PorUsuario`. Los nombres de tabla llevan el código del equipo. |
| `src/inscripciones/consultar-inscripcion/handler.mjs` | La entrada de API Gateway llama al caso de uso de consulta. |
| `src/inscripciones/consultar-inscripcion/service.mjs` | Lee el evento y después el pase. Antes de inscribirse, `registration` vale `null`. |
| `src/inscripciones/inscribir-usuario/service.mjs` y `repository.mjs` | Comprueban las referencias y guardan la inscripción con una condición que protege frente a duplicados. |
| `src/eventos/models/evento.model.mjs` y `src/inscripciones/models/inscripcion.model.mjs` | DynamoDB Toolbox traduce `eventId` a `EVENT_ID` y `userId` a `USER_ID` sin cambiar las claves de las tablas. |
| `src/datos/dynamo.mjs` | El cliente del SDK comparte la conexión y lee los nombres de tablas que inyecta Serverless. |

Sigue en el cuaderno los pasos [Crear con ORM](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/#capas-eventos-orm), [Consultar inscripción con ORM](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/#consultar-orm) e [Inscribir con ORM](https://albertohdezakkari.github.io/04-PILARES-SERVERLESS/practica/festival-checkin/#inscribir-orm). El alta del perfil y la lectura por clave usan el modelo de usuario de DynamoDB Toolbox.
# festival-checkin-local
# festival-checkin-local
