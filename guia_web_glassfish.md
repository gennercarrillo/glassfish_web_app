# 1. Configuración Inicial del Entorno

```bash
# Instalar dependencias
sudo apt-get update
sudo apt-get install -y openjdk-11-jdk mysql-server unzip
sudo service mysql start

# Configurar Java 11 como predeterminado
sudo update-alternatives --config java
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```

# 2. Instalar GlassFish 7.0.12

```bash
wget https://download.eclipse.org/ee4j/glassfish/glassfish-7.0.12.zip
unzip glassfish-7.0.12.zip -d /workspaces/
export GLASSFISH_HOME="/workspaces/glassfish7"
export PATH="$GLASSFISH_HOME/bin:$PATH"
```

# 3. Configurar MySQL

```bash
# Crear base de datos y usuario
sudo mysql -u root -e "CREATE DATABASE ejemplo_db; \
CREATE USER 'ejemplo_user'@'localhost' IDENTIFIED BY 'ejemplo_pass'; \
GRANT ALL PRIVILEGES ON ejemplo_db.* TO 'ejemplo_user'@'localhost'; \
FLUSH PRIVILEGES;"

# Crear tabla usuarios
sudo mysql -u ejemplo_user -pejemplo_pass ejemplo_db -e \
"CREATE TABLE IF NOT EXISTS usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(50),
    email VARCHAR(50)
)"
```

# 4. Configurar Connection Pool en GlassFish

```bash
# Instalar driver MySQL
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-8.0.33.zip
unzip mysql-connector-j-8.0.33.zip
cp mysql-connector-j-8.0.33/mysql-connector-j-8.0.33.jar $GLASSFISH_HOME/glassfish/domains/domain1/lib/

# Eliminar recursos Derby por defecto
asadmin delete-jdbc-connection-pool DerbyPool
asadmin delete-jdbc-resource jdbc/__default

# Crear pool MySQL
asadmin create-jdbc-connection-pool \
  --datasourceclassname com.mysql.cj.jdbc.MysqlDataSource \
  --restype javax.sql.DataSource \
  --property "user=ejemplo_user:password=ejemplo_pass:DatabaseName=ejemplo_db:ServerName=localhost:port=3306:allowPublicKeyRetrieval=true:useSSL=false" \
  mysql-ejemplo-pool

# Crear recurso JDBC
asadmin create-jdbc-resource --connectionpoolid mysql-ejemplo-pool jdbc/miDB

# Verificar conexión
asadmin ping-connection-pool mysql-ejemplo-pool
```

# 5. Estructura del Proyecto y Archivos

Crea exactamente estos archivos con el contenido que compartiste:

```
src/
└── main/
    ├── java/
    │   └── com/ejemplo/servlets/FormServlet.java
    └── webapp/
        ├── index.html
        ├── data.jsp
        └── WEB-INF/
            ├── web.xml
            └── classes/ (se creará al compilar)
```

## FormServlet.java

```java
package com.ejemplo.servlets;

import java.io.IOException;
import java.sql.Connection;
import java.util.*;
import java.sql.PreparedStatement;
import jakarta.annotation.Resource;
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import javax.naming.*;
import javax.sql.DataSource;

@WebServlet("/FormServlet")
public class FormServlet extends HttpServlet {

    private DataSource dataSource;
 
    @Override
    public void init() throws ServletException {
        try {
            Context initContext = new InitialContext();
            dataSource = (DataSource) initContext.lookup("jdbc/miDB");
        } catch (Exception e) {
            throw new ServletException(e);
        }
    }

    protected void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        String nombre = request.getParameter("nombre");
        String email = request.getParameter("email");

        System.out.println("Datos recibidos - Nombre: " + nombre + ", Email: " + email);

        try (Connection con = dataSource.getConnection();
                PreparedStatement ps = con.prepareStatement(
                        "INSERT INTO usuarios (nombre, email) VALUES (?, ?)")) {

            ps.setString(1, nombre);
            ps.setString(2, email);
            int rows = ps.executeUpdate();
            System.out.println("Filas afectadas: " + rows);

        } catch (Exception e) {
            e.printStackTrace();
            response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR,
                    "Error en base de datos: " + e.getMessage());
            return;
        }
        request.getRequestDispatcher("/data.jsp").forward(request,response);
    }
}
```

## index.html

```html
<!DOCTYPE html>
<html>
<head>
    <title>Formulario</title>
</head>
<body>
    <h1>Registrar Usuario</h1>
    <form action="FormServlet" method="post">
        Nombre: <input type="text" name="nombre"><br>
        Email: <input type="email" name="email"><br>
        <input type="submit" value="Guardar">
    </form>
    <a href="data.jsp">Ver Usuarios</a>
</body>
</html>
```

## data.jsp

```jsp
<%@ page import="java.sql.*, javax.sql.DataSource, javax.naming.*" %>
<!DOCTYPE html>
<html>
<head>
    <title>Usuarios Registrados</title>
</head>
<body>
    <h1>Usuarios</h1>
    <table border="1">
        <tr>
            <th>ID</th>
            <th>Nombre</th>
            <th>Email</th>
        </tr>
        <%
            Connection con = null;
            try {
                Context ctx = new InitialContext();
                DataSource ds = (DataSource) ctx.lookup("jdbc/miDB");
                
                con = ds.getConnection();
                Statement stmt = con.createStatement();
                ResultSet rs = stmt.executeQuery("SELECT * FROM usuarios");
                
                while (rs.next()) {
        %>
                    <tr>
                        <td><%= rs.getInt("id") %></td>
                        <td><%= rs.getString("nombre") %></td>
                        <td><%= rs.getString("email") %></td>
                    </tr>
        <%
                }
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                if (con != null) con.close();
            }
        %>
    </table>
    <a href="index.html">Volver al Formulario</a>
</body>
</html>
```

## web.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee https://jakarta.ee/xml/ns/jakartaee/web-app_5_0.xsd"
         version="5.0">

    <servlet>
        <servlet-name>FormServlet</servlet-name>
        <servlet-class>com.ejemplo.servlets.FormServlet</servlet-class>
    </servlet>

    <resource-ref>
        <description>MySQL DataSource</description>
        <res-ref-name>jdbc/miDB</res-ref-name>
        <res-type>javax.sql.DataSource</res-type>
        <res-auth>Container</res-auth>
    </resource-ref>
</web-app>
```

# 6. Compilación y Despliegue

```bash
# Compilar el Servlet
mkdir -p src/main/webapp/WEB-INF/classes
javac -cp "$GLASSFISH_HOME/glassfish/modules/*" src/main/java/com/ejemplo/servlets/FormServlet.java -d src/main/webapp/WEB-INF/classes/

# Crear archivo WAR
cd src/main/webapp
jar cvf ../../../app.war *

# Desplegar en GlassFish
asadmin deploy --force=true ../../../app.war
```

# 7. Verificación Final

Accede a:

- Formulario: https://TU_CODESPACE-8080.app.github.dev/app/index.html
- Lista de usuarios: https://TU_CODESPACE-8080.app.github.dev/app/data.jsp

# 8. Actualización Manual del WAR

En caso de problemas, corregir el código y ejecutar el siguiente bloque desde la raíz del proyecto:

Este bloque compila el codigo de java y lo carga al servidor de glassfish

```bash
asadmin stop-domain domain1
javac -cp "$GLASSFISH_HOME/glassfish/modules/*" src/main/java/com/ejemplo/servlets/FormServlet.java -d src/main/webapp/WEB-INF/classes
cd src/main/webapp
jar cvf ../../../app.war *
cp ../../../app.war $GLASSFISH_HOME/glassfish/domains/domain1/autodeploy/
asadmin start-domain domain1
```

