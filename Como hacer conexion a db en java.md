
(leer en modo fuente)
primero dependiendo de la db necesitas como un adaptador en el pom.xml (en el caso de maven)
en el caso que yo uso postgress el adaptador JDBC es algo asi:
```
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
```
 
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

```

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
```
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
```
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

```
        est.setString(1,usr.getNombre());
        est.setInt(2,usr.getEdad());
        est.setBoolean(3,usr.isTts());
```
 
y por ultimo le decimos a prepared Statement que ejecute la query
con 
```
         est.executeUpdate();
```
Si la query que se ejecuto tiene un error o durante la actualizacion en db ocurre algo esto lansara una excepcion lo mismo con el caso anterior que devuelve todo de la tabla

En caso de que no ocurra un excepcion en esta parte del codigo es que todo a ido bien 

Para ver especificamente los errores es necesario manejar los errors por try catch separados para cada caso 

# Conexion DB Hikari

Hikari es un pool de bases de datos osea que gestiona las conexiones a las bases de datos de forma eficiente en lugar de cerrar y abrir conexiones constantemente

Como lo estabamos haciendo resulta costoso y lento abrir y cerrar con cada consulta asi que solo es ideal para pruebas para un entorno de mas concurrencia y recomendado se usa HikariCP 

la dependencia para esto es

```
     <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
            <version>5.1.0</version>
        </dependency>
```


y la implementacion es de esta forma:

```
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

