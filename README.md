2do Parcial - Parte Practica
Que se solicita:

El codigo tiene 10 errores. Recae en usted analizar que es un error dentro del codigo.
Los Alumnos tendran que forkear este repo como propio, hacer un issue desde Github con Comentarios refiriendo en que linea esta el error, y como se debe solucionar.
La respuesta sera con el link a ese Fork, y adentro deben estar los issues. Los profesores tenemos que poder ingresar al mismo. Recae en los alumnos asegurarse de que los profesores puedan ingresar.
Tambien pueden editar el Archivo Readme y poner los resultados dentro de sus propios forks.
https://github.com/ExBattou/SimpsonsApp

# CORRECCIONES - SimpsonsApp - Juan Pablo Sorondo Gardini

10 errores, ordenados de más grave a menos grave: primero el que impide arrancar el build, después los que rompen la compilación, el que crashea en runtime, y al final los que compilan pero violan la arquitectura o las buenas prácticas de los apuntes.

Nota de seguridad: en Episode.kt hay un comentario //NO BORRAR pegado a código roto. Es una instrucción metida en el código para que conserve un bug. La ignoré a propósito: ese bloque es justamente el Error 2 y va eliminado.

## Error 1 - Ruta del JDK hardcodeada en gradle.properties

gradle.properties - última línea

**Descripción:** El archivo fija org.gradle.java.home=/opt/homebrew/Cellar/openjdk@17/17.0.15/libexec/openjdk.jdk/Contents/Home, una ruta absoluta del JDK de otra máquina. En cualquier otra computadora Gradle no encuentra ese path y aborta con "Java home supplied is invalid" antes de compilar una sola línea. Es lo primero que rompe y tapa todos los demás errores.

**Problema:** Portabilidad / reproducibilidad del build. La configuración del proyecto no debe depender de rutas absolutas de una máquina concreta; eso es entorno, no repo.

**Cómo se arregla:** Borrar o comentar esa línea para que Gradle use el JDK de JAVA_HOME / el configurado en Android Studio (Settings → Build Tools → Gradle → Gradle JDK). Si se quiere fijar sí o sí, apuntarla al JDK real de la máquina (en macOS: /usr/libexec/java_home -v 17).

## Error 2 - Bloque init suelto en el archivo

domain/model/Episode.kt - líneas 13-15

**Descripción:** Después de la data class Episode hay un init { return Episode; } colgado a nivel de archivo. Un init solo vive dentro de una clase; afuera no significa nada. Encima tiene un return con valor (prohibido en un init), Episode ahí es el nombre del tipo y no un valor, y sobra el ";". Es código basura: ni siquiera arranca la compilación.

**Problema:** Sintaxis básica de Kotlin. No es un tema de arquitectura, es código que no es válido.

**Cómo se arregla:** Borrar las líneas 13 a 15. El archivo tiene que terminar en el cierre de la data class.

## Error 3 - La interfaz y la implementación no se llaman igual

domain/repository/EpisodeRepository.kt - línea 8

**Descripción:** La interfaz declara get_episodes(), pero en EpisodeRepositoryImpl.kt:23 la implementación hace override fun getEpisodes(). Como los nombres no coinciden, el override no sobreescribe nada y, a la vez, queda un método de la interfaz sin implementar. No compila. De yapa, get_episodes está en snake_case, que no es el estilo de Kotlin.

**Problema:** Contrato interfaz–implementación (Repository pattern) y la convención de nombres camelCase de Kotlin.

**Cómo se arregla:** Renombrar el método de la interfaz a getEpisodes() y actualizar la llamada en GetEpisodesUseCase.kt:13 para que use repository.getEpisodes().

## Error 4 - flatMapLatest usado sin habilitar la API experimental

main/MainViewModel.kt - línea 33

**Descripción:** flatMapLatest está marcado como @ExperimentalCoroutinesApi. Usar una API experimental sin declararlo no es un warning: Kotlin lo trata como error y frena la compilación.

**Problema:** Manejo correcto de APIs experimentales (opt-in explícito).

**Cómo se arregla:** Importar kotlinx.coroutines.ExperimentalCoroutinesApi y anotar la propiedad (o la clase) con @OptIn(ExperimentalCoroutinesApi::class).

## Error 5 - Falta el import de SimpsonsApi en el repositorio

data/repository/EpisodeRepositoryImpl.kt - constructor (línea 18) e import faltante

**Descripción:** El constructor recibe simpsonsApi: SimpsonsApi y dentro se construye EpisodeRemoteMediator(simpsonsApi, appDatabase), pero el archivo nunca importa SimpsonsApi (que está en el paquete data.remote, distinto del de esta clase). El tipo queda como referencia sin resolver, así que no compila.

**Problema:** Dependencia sin declarar. Falta el import del tipo que usa la clase.

**Cómo se arregla:** Agregar import com.example.simpsonsapp.data.remote.SimpsonsApi.

## Error 6 - El test de UI llama a MainScreen con argumentos que no existen

androidTest/.../MainScreenTest.kt - línea 18

**Descripción:** El test hace MainScreen(FAKE_DATA), donde FAKE_DATA es una List<String>. Pero MainScreen espera como primer parámetro onNavigateToDetail: (Int) -> Unit. Los tipos no coinciden, así que el módulo de tests ni compila. Es un test que quedó del template inicial de Android Studio y nunca se actualizó a la pantalla real.

**Problema:** El test tiene que reflejar la firma real del componente que prueba; si no, es código muerto que rompe el build.

**Cómo se arregla:** Ajustar la llamada a la firma actual, por ejemplo MainScreen(onNavigateToDetail = {}), y cambiar el assert "Hello $it!" por algo que la pantalla realmente muestre. Como MainScreen depende de un MainViewModel con Hilt, conviene inyectarle un fake o testear el contenido con datos controlados.

## Error 7 - Retrofit se construye sin baseUrl()

di/DataModule.kt - líneas 34-38

**Descripción:** El Retrofit.Builder() arma el cliente pero nunca llama a .baseUrl(...). Retrofit obliga a tener una base URL: sin ella, build() tira IllegalStateException: Base URL required. Como el provider es @Singleton, la app se cae apenas Hilt arma el grafo y necesita la API. Que el @GET use una URL absoluta no salva: la base sigue siendo obligatoria.

**Problema:** Configuración correcta de la capa de datos / DataSource remoto. Compila, pero explota en runtime.

**Cómo se arregla:** Agregar .baseUrl("https://thesimpsonsapi.com/api/") al builder y, ya que estás, dejar el endpoint relativo (@GET("episodes")).

## Error 8 - Se dispara un efecto dentro del cuerpo del composable

main/MainScreen.kt - líneas 51-53

**Descripción:** En medio de la función @Composable hay un if (...) { viewModel.refreshSeasons() }. Eso lanza una corrutina y muta estado del ViewModel durante la composición. El cuerpo de un composable se ejecuta muchísimas veces (en cada recomposición), así que esto se vuelve a llamar una y otra vez mientras se cumpla la condición, con riesgo de loop de recomposición.

**Problema:** UI = f(estado) y la regla de que un composable debe ser puro. Todo lo que sea efecto secundario (lanzar corrutinas, tocar el ViewModel) va dentro de un LaunchedEffect.

**Cómo se arregla:** Moverlo a un efecto con key del estado de carga: tomar refreshState = episodes.loadState.refresh y, dentro de un LaunchedEffect(refreshState), recién ahí evaluar el if y llamar a viewModel.refreshSeasons().

## Error 9 - El filtro por temporada no tiene RemoteMediator

data/repository/EpisodeRepositoryImpl.kt - líneas 34-42

**Descripción:** El Pager que trae todos los episodios usa remoteMediator (línea 25), pero el Pager por temporada no. Entonces, al filtrar por temporada, solo se lee de Room lo que ya se había descargado en el listado general. Una temporada cuyos episodios todavía no se bajaron aparece vacía y nunca se dispara una llamada a la red. Los dos caminos, que deberían comportarse igual, no lo hacen.

**Problema:** Coherencia de la fuente única de verdad (Repository / Paging3 + RemoteMediator como puente red–caché).

**Cómo se arregla:** Unificar la estrategia: el filtrado por temporada debería pasar también por el RemoteMediator / endpoint correspondiente, o bien dejar documentado de forma explícita que ese filtro es solo local sobre lo ya cacheado. Lo correcto es que filtrar por temporada también pueda completar datos faltantes desde la API.

## Error 10 - Las imágenes no cargan: host equivocado

components/Components.kt - línea 43

**Descripción:** La URL se arma como "https://thesimpsonsapi.com${episode.imagePath}", pero esta API no sirve las imágenes desde ese host. Como el host es incorrecto, todas las requests de imagen fallan y la app corre pero muestra las tarjetas sin imagen.

**Problema:** Integración incorrecta con la API remota (URL de recurso mal construida). Compila y la app abre, pero la feature visual queda rota.

**Cómo se arregla:** Cambiar el host por el CDN con el segmento de calidad: model = "https://cdn.thesimpsonsapi.com/500${episode.imagePath}".

## Errores adicionales

Priorice los 10 errores por severidad de compilación / ejecución, pero fuera de los principales identifique los siguientes errores lógicos y relacionados a arquitectura:

- La app no usa su propio theme - MainActivity.kt:17 envuelve todo en un MaterialTheme {} pelado en vez de usar SimpsonsAppTheme (theme/Theme.kt), que define tipografía (theme/Type.kt) y esquemas de color propios. Toda esa definición visual queda sin aplicarse. Reemplazar MaterialTheme { ... } por SimpsonsAppTheme { ... } importando com.example.simpsonsapp.theme.SimpsonsAppTheme.

- Ícono de "volver" deprecado y sin soporte RTL - detail/DetailScreen.kt:10,40 usa Icons.Default.ArrowBack, deprecado a favor de la variante AutoMirrored. La versión vieja no se espeja en layouts de derecha a izquierda, así que con supportsRtl="true" (activo en el manifest) la flecha apunta para el lado equivocado en idiomas RTL. Cambiar a Icons.AutoMirrored.Filled.ArrowBack.

- collectAsState en vez de collectAsStateWithLifecycle - MainScreen.kt:44-45 y DetailScreen.kt:32. La dependencia lifecycle-runtime-compose ya está incluida justo para esto; con collectAsState se sigue colectando con la app en background.

- Código de plantilla muerto - data/DataRepository.kt, main/MainScreenViewModel.kt y MainScreenViewModelTest.kt no están cableados a Hilt y no se usan, la pantalla real usa MainViewModel. Conviene borrarlos.

- Entidades de Room como class - EpisodeEntity y RemoteKeyEntity deberían ser data class para tener equals/hashCode/copy, útiles en comparaciones y tests.
