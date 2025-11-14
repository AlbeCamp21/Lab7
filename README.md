# **Lab7**

## **Patrón Adapter**

### Ejecución

![adapter_1](/img/adapter_1.png)

![adapter_2](/img/adapter_2.png)

### Ejercicios teóricos y prácticos

1. Describe cómo garantizarías la validez del contrato `outputs()` en un proyecto con tres adaptadores distintos (por ejemplo, para GCP, AWS y fichero CSV). Indica qué pruebas unitarias escribirías para verificar que cada adaptador cumple su interfaz.

    Para garantizar que los adaptadores sigan el contrato de `outputs()`, se deben indicar reglas para que `outputs()` siempre devuelva una lista con exactamente tres strings (user, identity y role). Se podría poner comprobaciones al inicio del código para validar la forma y tipos, fallando si es que algo no sigue las reglas.

    Para las pruebas unitarias, un test fijo podría ser el test donde se valide el contenido esperado (verificar que cada tupla sea: user, identity y role). Otro test puede ser de datos erroneos como valores nulos o tipos incorrectos, verificando también que se lancen las excepciones correctas (como ValueError o TypeError).

2. Analiza la complejidad temporal y espacial de `LocalIdentityAdapter` y de `LocalProjectUsers` en función del número de roles $R$ y usuarios $U$. ¿Cómo escalaría el sistema si duplicas el módulo de metadatos original?

    Viendo el código de la clase `LocalIdentityAdapter`, este recorre los roles y los usuarios dentro de dichos roles para la creación de listas. Esto quiere decir que el tiempo que tarda y el uso de memoria crece directamente según la cantidad de usuarios (entre más usuarios más trabajo).

    Luego, la clase `LocalProjectUsers` recibe dicha lista y crea bloques JSON para caad tupla. Entonces, si se duplica el módulo de metadatos, también se estaría duplicando el trabajo y el JSON. Para evitar esto, una buena opción podría ser eliminar elementos duplicados o realizar la generación del JSON por secciones.

3. Introduce deliberadamente un error en el mapeo de roles (por ejemplo, `read` -> `read_only` en `LocalIdentityAdapter`). Ejecuta Terraform `validate` y `plan` para observar y documentar el fallo. Luego corrige el error y compara la salida de los comandos.

    La modificación que se hizo fue en el archivo `main.py`, cambiando la linea:

    ```python
    for user in users:
        self.local_users.append((user, user, permission))
    ```

    por:

    ```python
    for user in users:
        role = 'read_only' if permission == 'read' else permission
        self.local_users.append((user, user, role))
    ```

    Al volver a ejecutar `terraform plan -out=tfplan_error`, obtenemos esto:

    ![adapter_3](/img/adapter_3.png)

4. Configura un pipeline sencillo (por ejemplo, con GitHub Actions) que automatice:

   * Ejecución de `python main.py` para regenerar `main.tf.json`.
   * Comandos `terraform validate` y `terraform plan`.
   * Reporte de errores en caso de validación o planificación fallida.

    ```yml
    name: CI
    on: [push]

    jobs:
      terraform:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          - uses: hashicorp/setup-terraform@v2
          - uses: actions/setup-python@v4
          
          - name: Run workflow
            run: |
              python3 main.py
              terraform init
              terraform validate
              terraform plan
          
          - name: Reporte de Error
            if: failure()
            run: echo "El pipeline falló"
    ```


## **Patrón Facade**

### Ejecución

![facade_1](/img/facade_1.png)

![facade_2](/img/facade_2.png)

### Ejercicios teóricos

1. **Comparativa de patrones**

    1. Describe en un diagrama UML las responsabilidades de cada patrón.
    ```mermaid
    classDiagram
        class Adapter {
            "Traduce" una interfaz para que sea compatible con otra.
        }

        class Facade {
            Brinda una interfaz simple para un sistema complejo.
        }

        class Mediator {
            Centra la comunicación entre objetos, para que no hablen entre si.
        }

        class Dependency Injection {
            Un objeto recibe dependencias desde fuera, en vez de crearlas.
        }

        class Inversion of Control {
            El framework maneja el flujo del programa, no el código.
        }
    ```

    2. Explica por qué elegimos Facade para desacoplar módulos de Terraform y Python en lugar de, por ejemplo, Adapter.

        Se usa Facade porque el objetivo de este patrón es facilitar un proceso que tiene varios pasos. El proceso de generación de los archivos JSON (con dependencias y detalles), para python y terraform, puede llegar a ser complejo. Por eso, el facade funciona como una "fachada" que oculta toda esa complejidad en un método fácil de usar. En cambio, adapter se usaría si se tuviera un formato de datos a otro para que dos sistemas se entiendan, lo cual no pasa con python y terraform. 

    3. Discute los pros y contras de usar Facade en IaC a gran escala (mantenimiento, legibilidad, potencial de acoplamiento).

        Para gran escala, usar el patrón facade tiene como ventajas el mantenimiento y legibilidad, ya que solamente se necesitaría aprender una interfaz simple en lugar de los otros complejos. Ahora, hablando de desventajas, el facade puede convertirse en una herramienta que abarca todo (que hace todo), causando un cuello de botella debido a la gran cantidad de operaciones que tiene que realizar.

2. **Principio de inversión de dependencias**

    1. Señala en el código dado cuáles son las "abstracciones".

        Las abstracciones son las clases `StorageBucketModule` y `StorageBucketAccessModule`. Estas clases no "piensan" en los detalles de los recursos `null_resource`, sino, estos piensan en conceptos de alto nivel como modelos de bucket. Estas clases tienen funciones como `resource()` o `outputs()` que son la muestra de estas abstrancciones, ya que ocultan la complejidad de la construcción del JSON.

    2. Propón una refactorización que invierta aún más las dependencias (por ejemplo, inyectando el intérprete "python" como parámetro).

        Una refactorización podría ser dejar de escribir los archivos JSON directamente desde el `main.py`. ENtonces, se podría crear una clase con un método llamado `write` con parametros como el nombre del archivo y la data. Entonces, en vez de que `main.py` cree los archivos, reciba instancia de esta nueva clase indicando que escriba estos datos en este archivo, sin saber qué es lo que hace por dentro.

    3. Justifica cómo tu cambio mejora (o no) la adherencia al DIP.

        El cambio mejora y cumple con el principio de inversión de dependencias porque la lógica principal de `main.py` ya no dependería de las funciones de bajo nivel (como escribir archivos), sino que ahora dependería de la clase nueva proponida, haciendo el código flexible y fácil de probar en tests.

3. **Escalabilidad y mantenimiento**

    1. Imagina 10 módulos de alto nivel que consumen el mismo facade del bucket (`name`, `path`, `region`, `labels`, etc.).
    2. Describe el problema de "explosión de referencias" al renombrar un campo.

        Suponiendo que esos 10 módulos dependen de un determinado campo. En caso que se quiera renombrar ese campo a otro nombre dentro de facade, ocurriría un problema, que es que se tendría que ir a cada uno de esos 10 módulos y cambiar manualmente dicha refencia a ese campo. Esto es llamado "explosión de referencias", debido a que un solo cambio en el facade puede provocar más trabajo.

    3. Propón dos alternativas arquitectónicas (por ejemplo, subdividir el facade o introducir otro nivel de API interna) y di en qué casos usarías cada una.

        Alternativa 1: Dividir el facade. En vez de tener un único facade que maneje toda la configuración, se dividiría en varias partes con funciones específicas, como un facade para las etiquetas y nombre y otro facade para la ruta.
        Alternativa 2: Agregar una capa de API interna entre los módulos y el facade. Para este caso, cada módulo no se comunicaría directamente con el facade, sino con un adaptador que recibe las peticiones.

## **Patrón Mediator**

### Ejecución

![mediator_1](/img/mediator_1.png)

![mediator_2](/img/mediator_2.png)

### Ejercicion Teóricos

1. Idempotencia y Mediador
    Define "idempotencia" en el contexto de IaC y explica por qué es un requisito fundamental para el patrón mediador.
    Discute qué podría fallar si el mediador no garantiza idempotencia y da un ejemplo concreto basado en tu proyecto (por ejemplo, creación duplicada de recursos).

    La idempotencia nos asegura que aplicar una configuración varias veces genera el mismo resultado que aplicarlo una vez. Para el patrón mediator es importante ya que evita la duplicación de recursos.
    Si el mediator no garantizaría la idempotencia, cada ejecución que se haga podría generar recursos adicionales que no se desean.

2. Comparativa de patrones
    Compara el patrón mediador con el manejo nativo de dependencias en Terraform (graph-driven).
    Señala al menos dos ventajas y dos desventajas de implementar tu propio mediador en Python frente a dejar que Terraform ordene los recursos automáticamente.

    El mediator, para python, se dedica a gestionar las dependencias de forma clara en el código, dando así un control detallado del flujo. Por otro lado, terraform utiliza un grafo de dependencias para que se determine el orden de creación de los recursos.  
    Ventajas: Control total ("orquestación" personalizada) y flexibilidad (facilita la integración con otros sistemas).  
    Desventajas: Complejidad y perdida de optimización.

4. Patrón Mediador y buenas prácticas
    Explica cómo mantener el código del mediador limpio y mantenible a medida que crecen los módulos (pista: considera estrategias como "registrar" los módulos en un array con su prioridad).
    ¿Qué técnicas de testing recomendarías para validar la lógica de inyección de dependencias?

    Para mantener el mediator "limpio" a medida que crece el código, se puede añadir un sistema de registro de módulos. Entonces, cada módulo se registraría con sus dependencias y el mediator leería dicho registro para determinar el orden de la ejecución, en vez de usar por ejemplo lógica conddicional.
    Ahora, para validar la lógica de inyección de dependencias, se podría usar pruebas unitarias que verifiquen cada módulo y que estas reciben sus dependencias correctas.

## **Inversión de Control**

![inv_control_1](/img/inv_control_1.png)

![inv_control_2](/img/inv_control_2.png)

![inv_control_3](/img/inv_control_3.png)

### Ejercicios teóricos

1. Relaciona la Inversión de Control con el principio de D (Dependecy Inversion) en SOLID. Describe un ejemplo en IaC.

    El IoC es la aplicación práctica de la inversión de dependencias. Entonces, en lugar de que un módulo de alto nivel controle a uno de bajo nivel, ambos dependerían de una abstracción, invirtiendo así el flujo de control.
    Por ejemplo, en el proyecto, el `ServerModule` no crea su propia configuración de la red, sino que un script externo brinda está configuración. Así el módulo del servidor no depende directamente de la implementación de la red, sino de la estructura de datos que este recibe.

2. Explica las diferencias entre IoC y un enfoque tradicional (referencia directa), indicando ventajas y limitaciones.

    Según eun enfoque tradicional, el módulo del servidor importaría y crearía directamente la configuración de la red. Con IoC, la configuración vendría desde afuera.
    La ventaja del IoC es que puede desacoplarse, permitiendo cambiar o probar componentes de forma aislada. Ahora, una limitación es que puede volverse complicado ya que el flujo de ejecución es menos directo.

3. Discute cómo IoC facilita la introducción de nuevos atributos (gateway, tags, etc.) en el módulo de red sin tocar el módulo de servidor.

    El IoC nos permite añadir atributos como `gateway` o `tags` al módulo de la red sin modificar el servidor. El `ServerModule`, para este caso, solo espera un diccionario con los datos que necesita (name, cidr y zone) y simplemente ignora cualquier otro atributo que se le pase.

## **Inyección de Dependencias**

![iny_depen_1](/img/iny_depen_1.png)

![iny_depen_2](/img/iny_depen_2.png)

![iny_depen_3](/img/iny_depen_3.png)

![iny_depen_4](/img/iny_depen_4.png)

### Ejercicios


1. Definición y principios

   * Explica en tus propias palabras qué es la **inyección de dependencias** y cómo combina los principios de **inversión de control** e **inversión de dependencias**.
   * Compara la inyección de dependencias con la interpolación de atributos y las salidas de módulo en Terraform: ¿qué ventajas y desventajas presenta cada enfoque?

    La inyeccion de dependencias es una técnica donde los componenetes reciben sus respectivas dependencias desde una fuente externa, en vez de crearlas.
    Se implementa la inversión de control, debido a que un framework externo gestiona las dependencias y, la inversión de dependencias, esto debido a que los módulos dependen de los contratos y no de implementaciones especificas.

2. Blast radius y acoplamiento

   * Describe qué se entiende por "blast radius" en IaC y cómo la inyección de dependencias ayuda a minimizarlo.
   * Propón al menos dos escenarios reales en los que un cambio en un módulo de bajo nivel podría "romper" un módulo de alto nivel si no existiera la capa de inyección de dependencias.

   El blast radius se refiere a como un cambio de un componente afecta a otros. Ahora, la inyección de dependencias lo minimiza al separar los módulos. Si el JSON se respeta, los cambios internos en un módulo no afectan a los que dependen de él.  
   Escenarios: Primero, si se renombra un recurso de red en `network.tf`, el módulo del servidor que lo referencia fallaría. Segundo, si el módulo de red cambia el nombre de una de sus variables de salida, el módulo de servidor dejaría de funcionar.  

4. Contratos y versiones

   * Propón un esquema para versionar tu contrato de metadatos (por ejemplo, añadiendo un campo `version` al JSON). ¿Cómo gestionarías la compatibilidad hacia atrás en `main.py`?

   Para poder versionar el contrato, se podría añadir un campo que indique la versión del contrato al archivo `network_metadata.json`.
   En el archivo `main.py` se administraría la compatibilidad hacia atrás leyendo dicho campo. Entonces, se usaría una estructura de condicionales para seleccionar la lógica del procesamiento dependiendo de la versión del contrato.