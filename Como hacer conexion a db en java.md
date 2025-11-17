
(leer en modo fuente)
primero dependiendo de la db necesitas como un adaptador en el pom.xml (en el caso de maven)
en el caso que yo uso postgress el adaptador JDBC es algo asi:
```xml title:pom.xml

<dependency>  
    <groupId>org.postgresql</groupId>  
    <artifactId>postgresql</artifactId>  
    <version>42.7.3</version>  
</dependency>

```
Esto es un dirver para una base de datos tipo postgresql para cada base datos existe un driver diferente hay que buscarlo y ponerlo en el pom.xml antes de iniciar
Ahora teniendo esto se hace una conexion a db 
puede ser simple en algo como por ejemplo
Codigo:
```java
 
String url = "jdbc:postgresql://localhost:5432/Testeo";  
String usuario = "postgres";  
String contrasena = "root";  
try(Connection conn = DriverManager.getConnection(url,usuario,contrasena)){  
        obtenerDatosDeLaTabla(conn);  
  
}catch (SQLException e){  
    System.out.println(e.getMessage());  
}
```
En este caso obtenerDatosDeLaTabla puede ser cualquier funcion que lance una querry a la conexion 
Y el resto de datos deben ser los de nuestra base de datos en este caso independientemente de la misma debe inicar con jdbc

Query o solicitudes a la db
Veamos un caso de eliminar de obtener a los usuarios

```java

private static void obtenerDatosDeLaTabla(Connection conn) throws SQLException {  
    System.out.println("Conexion exitosa");  
    Statement st = conn.createStatement();  
    ResultSet rs = st.executeQuery("Select * from datos");  
    ResultSetMetaData metaData = rs.getMetaData();  
    int columnas = metaData.getColumnCount();  
    while (rs.next()) {  
  
	for (int i = 1; i <= columnas; i++) {  
            String nombreC = metaData.getColumnName(i);  
            Object c = rs.getObject(i);  
            System.out.println(nombreC +":"+c);  
  }   
         }
         }
```
Este ejemplo sirve para hacer leer todos los datos de una tabla
pero lo importante es Statement y el resto
Statement es una interfaz hecha para hacer conexiones a una db en este caso 
la conexion ya previamente hecah crea un Statement para la Interface Statement

Despues Statement executa la Query que nosotros especifiquemos y regresa los resultados en un ResultSet que es mas o menos el equivalente de una tabla en la memoria actual 
ResultSet puede leer los datos de la tabla con rs.next() (el que se usa en el codigo) pero tambien los previos con  rs.previous(); esto lee cada columna

Siguiendo el codigo ahora se lee la metadata de el ResultSet con .getMetaData() y se guarda en un ResultSetMetdaData
Esto no son los datos de la tabla si no los metadatos como nombres de las columnas si se premiten NULL si son autoincrementables etc esto nos sirve para saber cuantas columnas tenemos por ejemplo

Ahora analisemos el bucle
```java
    int columnas = metaData.getColumnCount();  
  while (rs.next()) {  
  
	for (int i = 1; i <= columnas; i++) {  
            String nombreC = metaData.getColumnName(i);  
            Object c = rs.getObject(i);  
            System.out.println(nombreC +":"+c);  
  }   
         }
         }
```
Ahora el bucle while en cada iteracion hace un rs.next y se dentendra hasta iterar todas las filas de la tabla de ahi el bucle for itera las veces de columnas que anterior mente obtuvimos con la metadata

Con la misma metadata obtenemos el nombre de la columna en la posicion i (la primera columna) que en este caso es 1 (es practicamente un arreglo pero este no inicia en 0 si no siempre inicia en 1)
Ahora inicia un Object para recibir cualquier cosa que regrese la tabla ya sea un Integer un String etc podriamos utilizar .getString(i) pero solo estando seguros de el tipo de dato que devolvera la tabla de lo contrario devolvera un error

Asi que el caso mas seguro Object para contemplar cualquier tipo de objeto


Ahora veamos como pasar una query con variables para un Where o un Insert para la base de datos y limpiarlos de una posible inyeccion sql

Veamos este codigo
```java
    private static void insertarDatos(Usuariodb usr, Connection conn) throws SQLException {
        String sql = "INSERT INTO datos (nombre,edad,tts) VALUES (?,?,?) ";
        PreparedStatement est = conn.prepareCall(sql);
        est.setString(1,usr.getNombre());
        est.setInt(2,usr.getEdad());
        est.setBoolean(3,usr.isTts());
        est.executeUpdate();
    }
```

En este caso de codigo primero preparamos la query que vamos a ejecutar en este caso un insert a nombre edad y un parametro boolean tts (es por probar booleanos en db) ahora como de esta ejecucion no obtendremos ninguna respuesta solo tenemos que ejecutarlo
En lugar de usar Statement y concatenar las variables en la String que ejecutaremos usaremos PreparedStatement dentro de la query o el sql a ejecutar ponder (?) donde iran los datos que insertara PraparedStatement esto se hace para evitar inyecciones sql dentro de la base de datos y se consdiera una buena practica para esto

Ahora para insertar los datos necesarios dentro de la llamada primero usamos conn.prepareCall()
    (recordemos que conn es la conexion previamente hecha en el inicio)
 y la guardamos en PreparedStatement en la variante que llamaremos est

 ahora llamaremos a est.set();
 y el tipo de dato que queramos insertar para la columna de la tabla
 en este caso est.setString(); inserta un string

Cualquier est.set() independientemente del dato tiene dos parametros el primero es un int que especifica en que lugar de los (?) que marcamos en la query  va a ocupar y el segundo parametro es el dato que guardaremos

Por ejemplo tenemos en el codigo 

```java
        est.setString(1,usr.getNombre());
        est.setInt(2,usr.getEdad());
        est.setBoolean(3,usr.isTts());
```
 
y por ultimo le decimos a prepared Statement que ejecute la query
con 
```java
         est.executeUpdate();
```
Si la query que se ejecuto tiene un error o durante la actualizacion en db ocurre algo esto lansara una excepcion lo mismo con el caso anterior que devuelve todo de la tabla

En caso de que no ocurra un excepcion en esta parte del codigo es que todo a ido bien 

Para ver especificamente los errores es necesario manejar los errors por try catch separados para cada caso 

# Conexion DB Hikari

Hikari es un pool de bases de datos osea que gestiona las conexiones a las bases de datos de forma eficiente en lugar de cerrar y abrir conexiones constantemente

Como lo estabamos haciendo resulta costoso y lento abrir y cerrar con cada consulta asi que solo es ideal para pruebas para un entorno de mas concurrencia y recomendado se usa HikariCP 

la dependencia para esto es

```java
     <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
            <version>5.1.0</version>
        </dependency>
```


y la implementacion es de esta forma:

```java
package org.example.ProvedoresDeDatos;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

import javax.sql.DataSource;

public class ProvedorDeDatosHIkari {

    private static volatile HikariDataSource ds;

    ProvedorDeDatosHIkari() {}

    public static DataSource get() {
        if (ds == null) {
            synchronized (ProvedorDeDatosHIkari.class) {
                if (ds == null) {
                    HikariConfig cfg = new HikariConfig();


                    cfg.setJdbcUrl(env("PG_URL", "jdbc:postgresql://localhost:5432/Aplicacion46"));
                    cfg.setUsername(env("PG_USER", "postgres"));
                    cfg.setPassword(env("PG_PASSWORD", "pwd"));

                    cfg.setMaximumPoolSize(intEnv("PG_POOL_SIZE", 10));
                    cfg.setMinimumIdle(intEnv("PG_MIN_IDLE", 2));
                    cfg.setConnectionTimeout(longEnv("PG_CONN_TIMEOUT_MS", 10_000));
                    cfg.setIdleTimeout(longEnv("PG_IDLE_TIMEOUT_MS", 60_000));
                    cfg.setMaxLifetime(longEnv("PG_MAX_LIFETIME_MS", 30 * 60_000));
                    cfg.setPoolName(env("PG_POOL_NAME", "pg-pool"));

                    ds = new HikariDataSource(cfg);

                    Runtime.getRuntime().addShutdownHook(new Thread(() -> {
                        HikariDataSource x = ds;
                        if (x != null && !x.isClosed()) x.close();
                    }));
                }
            }
        }
        return ds;
    }

    private static String env(String k, String def) {
        String v = System.getenv(k);
        return (v == null || v.isEmpty()) ? def : v;
    }

    private static int intEnv(String k, int def) {
        try { return Integer.parseInt(env(k, String.valueOf(def))); }
        catch (Exception e) { return def; }
    }

    private static long longEnv(String k, long def) {
        try { return Long.parseLong(env(k, String.valueOf(def))); }
        catch (Exception e) { return def; }
    }


}

```


Analizemos el codigo 

Primero 
```java
import javax.sql.DataSource;
```
esto es una interface de java que nos proporciona una conexion en lugar de un DriveManager para obtener una conexion tambien se puede utilizar DataSource
de esta forma
```java
DataSource ds = new MysqlDataSource();
ds.setServerName("localhost");
ds.setDatabaseName("mi_base");
ds.setUser("usuario");
ds.setPassword("clave");

Connection conn = ds.getConnection();
```
La diferencia con DriveManager es que es mas flexible y permite un manejo mas eficiente de las conexiones con una implementacion adecuada aunque si es un proyecto pequeño sigo recomendando mas DriveManager y no es algo que se limite a bases de datos 

Ahora 

```java
  private static volatile HikariDataSource ds;
```

Primero HikariDataSource es una implementacion de la DataSource (el anteriormente importado) que ya trae la dependencia (o libreria) de hikari esta se implementa a modo de singleton marcandola como privada y statica es decir privada es solo de la clase y static que pertence a la instancia y no a una instancia 
Y por ultimo volatile esto hace que todos los hilos vean a ds o el objeto de la implementacion HikariDataSource y ademas tengan el valor de la variable actualizado y no un valor del cache de la JVM
(dejemoslo en que es necesario el volatile)
```java
ProvedorDeDatosHIkari() {}
```
Despues tenemos un constructor vacio para llamar a la conexion
Ahora vamos por partes
```java
public static DataSource get() {
```
Inciamos un metodo estatico llamado get que retorna un DataSource esto siguiendo el patron de diseño sigleton para devolver la misma instancia
Despues
```java
if (ds == null) {
```
Verificamos si ds es nulo esto quiere decir que no se a creado ninguna instancia de este objeto (o no se a creado la instancia) por lo que si es nula la creara pero si no lo es quiere decir que ya existe la instancia por lo que la retornara<
Ahora 
```java
synchronized (ProvedorDeDatosHIkari.class) {
```
Con esto se sincroniza el acceso a la clase para que dos o mas hilos puedan acceder a la instancia o intentar crearla ya que suponiendo que no hay instancia y no tenemos este sincronizar podriamos generar sin querer dos instancias cuando solo requerimos una o podemos tener bloqueda la instancia por un hilo y terminar creando otra por eso con esta evitamos que varios hilos accedan a la clase o instancia al mismo tiempo
```java
if (ds == null) {
```
Despues volvemos a verificar si es nullo por si en esa concurrencia se creo la instancia o no teniamos una clase actualizada

```java
 HikariConfig cfg = new HikariConfig();
```
Aqui creamos un nuevo objeto de configuracion que nos ayudara a configurar los datos de la db a la que necesitamos conectarnos 
```java
    cfg.setJdbcUrl(env("PG_URL","jdbc:postgresql://localhost:5432/Aplicacion46"));
            cfg.setUsername(env("PG_USER", "postgres"));
            cfg.setPassword(env("PG_PASSWORD", "pwd"));

            cfg.setMaximumPoolSize(intEnv("PG_POOL_SIZE", 10));
            cfg.setMinimumIdle(intEnv("PG_MIN_IDLE", 2));
            cfg.setConnectionTimeout(longEnv("PG_CONN_TIMEOUT_MS", 10_000));
            cfg.setIdleTimeout(longEnv("PG_IDLE_TIMEOUT_MS", 60_000));
            cfg.setMaxLifetime(longEnv("PG_MAX_LIFETIME_MS", 30 * 60_000));
            cfg.setPoolName(env("PG_POOL_NAME", "pg-pool"));
```
Las primeras 3 lineas son para configurar donde esta nuestra base de datos primero la base de datos 
```java
"jdbc:postgresql://localhost:puerto/nombre_de_DB"
```

las demas lineas son configuraciones mas explicitas de la configuracion que van a tener las conexiones o solicitudes que hagamos
```java
        cfg.setMaximumPoolSize(intEnv("PG_POOL_SIZE", 10));
        cfg.setMinimumIdle(intEnv("PG_MIN_IDLE", 2));
```
empezemos con MaximumPoolSize esta linea lo que hace es poner un limite a la cantidad de conexiones simultaneas que puede tener la DB si llega a tener mas el servicio esperara hasta que una de las conexiones termine para conectarse

El MinimunIdle es la cantidad minima de hilos que el servicio ya tiene preparado tomando el resto con menos recursos para mejor eficiencia cuando se esten usand pocas conexiones 


```java
            cfg.setConnectionTimeout(longEnv("PG_CONN_TIMEOUT_MS", 10_000));
            cfg.setIdleTimeout(longEnv("PG_IDLE_TIMEOUT_MS", 60_000));
            cfg.setMaxLifetime(longEnv("PG_MAX_LIFETIME_MS", 30 * 60_000));
            cfg.setPoolName(env("PG_POOL_NAME", "pg-pool"));
```
ConnectionTimeout Determina cuantos milisegundos puede tomar una solicitud antes de fallar es decir si toma mas de este tiempo se tomara como un error

IdleTimeout determina cuanto tiempo permanece activa una conexion antes de cerrarse 

MaxLifetime determina el tiempo que permanece abierta la conexion antes de cerrarse definitivamente (toda la conexion a la db)

Por ultimo pool name es para que en depuracion salga este nombre y se sepa que hilo salio mal y sepamos que fue parte de la DB

``` java 
            ds = new HikariDataSource(cfg);

           Runtime.getRuntime().addShutdownHook(new Thread(() -> {
                    HikariDataSource x = ds;
                    if (x != null && !x.isClosed()) x.close();
                }));
```
La primera linea hace que ds (la conexion y la instancia singleton) sea igual a ya la implementacios de Hikari tomando en el constructor la configuracion previamente establecida


Por ultimo esto llamado "ShutdownHook" sirve para cerrar correctamente la conexion hecha es decir esto se ejecuta cuando lo ejecutemos y detengamos con ctrl + c la pausa del compilador etc y lo que hace es copiar la instancia que teniamos verficar que exista y si no esta cerrada
De no ser asi procederemos a cerrarla 
Esto para cerrar correctamente y evitar concurrencia no deseada

Por ultimo lo usamos de esta forma
``` java
 DataSource dataSource = ProvedorDeDatosHIkari.get();
//(esto tomando como que ProvedorDeDatosHIkari es el nombre de la clase donde esta toda la funcionalidad explicada)
```

Ahora supongamos una clase que enviara o recibira datos de la DB
Necesitamos una instancia del proveedor de datos
entonces:
```java
 private final DataSource ds;

    public RepositorioUsuariosSql(DataSource ds) {

        this.ds = ds;
    }
    
```
Recordemos que Hikari instancia de DataSource asi que usamos a esta para usarla

Ahora supongamos que queremos guardar algo en la db

```java

 @Override
    public User guardarUsuario(User usr) {
        String sql = "insert into users(nombre) values (?)";
        try(Connection cn = ds.getConnection()){
            PreparedStatement preparedStatement =  cn.prepareStatement(sql);
        preparedStatement.setString(1, usr.getNombre());
        preparedStatement.execute();
        return usr;


        }
        catch (SQLException e){
            System.out.println("Error al insertar usuario en la conexion a db"+e.getMessage());
        }
        return usr;
    }

```

Recordemos que ds es nuestra instancia al proveedor de datos al usar .getConnection() DataSource ya tiene configurado todo para darnos una conexion y la guardamos en un objeto Connection llamado "cn"
Usamos PreparedStatement para preparar la llamada con cn.prepareStatement(sql); donde dentro de esta funcion ira la query

luego si nuestra llamada tiene parametros los especificamos con .setString(); o .setInteger(); etc (depende del tipo de dato que vayamos encadenar) esto previniendo inyecciones a sql

Ahora contrario a lo habitual de que la posicion 0 sea la primera aqui la 1 si es la primera asi que ignoremos el uso habitual de indices en arreglos
Por ultimo a la varibale del PreparedStatement usamos .execute() para lanzar la llamada 
Si es un update o delete (algo que no haga un retorno directo) y todo hasta ahora no a lanzado algun error por el try significa que la llamada a sido exitosa

Ahora supongamos que buscamos un usuario
```java


 @Override
    public Optional<User> buscarUsuarioPorNombre(String nombre) {
        String sql = "select * from users where nombre = (?)  LIMIT 1";

        try(Connection cn = ds.getConnection()){
            PreparedStatement preparedStatement = cn.prepareStatement(sql);
            preparedStatement.setString(1, nombre);
            try (ResultSet resultSet = preparedStatement.executeQuery()) {


                if (resultSet.next()) {
                    User user = new User(resultSet.getString("nombre"));
                    user.setId(resultSet.getLong("id"));
                    return Optional.of(user);
                }


            }
        }catch (Exception e){
            System.out.println("Error al buscar usuario en la conexion a db"+e.getMessage());
        }


        return Optional.empty();
    }
```
Nuevamente los pasos 
Obtenemos la conexion de ds y la tomamos en un objeto Connection
Preparamos y declaramos los argumentos que llevara la solicitud 
Ahora atencion a esta linea

```java
try (ResultSet resultSet = preparedStatement.executeQuery()) {
```
Aqui estamos declarando un ResultSet esto obtendra algo al executar la sentencia 
Este resultSet funciona como un buffer por lo que en este ejemplo en las lineas 
```java
                User user = new User(resultSet.getString("nombre"));
                user.setId(resultSet.getLong("id"));
```
Aqui del resultSet estamos obteniendo los valores que necesitamos de acuerdo a la tabla o fila obtenida en este caso el nombre y la id del usuario pero si fueran varios resultados que debemos de leer como select * from 
Seria asi:
```java
    while(resultSet.next()){
                User user = new User(resultSet.getString("nombre"));
                user.setId(resultSet.getLong("id"));
                users.add(user);

            }
```
Aqui en cada vuelta del while ejecutara resultSet.next() hasta que se lean todos los resultados obtenidos (tal como un buffer)

            