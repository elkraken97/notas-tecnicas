
(leer en modo fuente)
primero dependiendo de la db necesitas como un adaptador en el pom.xml (en el caso de maven)
en el caso que yo uso postgress el adaptador es algo asi:
```
<dependency>  
    <groupId>org.postgresql</groupId>  
    <artifactId>postgresql</artifactId>  
    <version>42.7.3</version>  
</dependency>

```
Conexion a DB
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

