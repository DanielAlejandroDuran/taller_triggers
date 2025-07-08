# 🚀 **Taller de Triggers en MySQL** -- Daniel Alejandro Duran Franco

## 📌 **Objetivo**

En este taller, aprenderás a utilizar **Triggers** en MySQL a través de casos prácticos. Implementarás triggers para validaciones, auditoría de cambios y registros automáticos.

## **🔹 Caso 1: Control de Stock de Productos**

### **Escenario:**

Una tienda en línea necesita asegurarse de que los clientes no puedan comprar más unidades de un producto del stock disponible. Si intentan hacerlo, la compra debe **bloquearse**.

### **Tarea:**

1. Crear las tablas `productos` y `ventas`.

   **Tablas**

   ```sql
   CREATE TABLE productos (
       id INT PRIMARY KEY AUTO_INCREMENT,
       nombre VARCHAR(50),
       stock INT
   );
   ```

   ```sql
   CREATE TABLE ventas (
       id INT PRIMARY KEY AUTO_INCREMENT,
       id_producto INT,
       cantidad INT,
       FOREIGN KEY (id_producto) REFERENCES productos(id)
   );
   ```

   **Datos**

   ```sql
   INSERT INTO productos (nombre, stock) VALUES 
   ('Laptop', 10),
   ('Mouse', 25),
   ('Teclado', 15),
   ('Monitor', 8);
   ```

   

2. Implementar un trigger `BEFORE INSERT` para evitar ventas con cantidad mayor al stock disponible.

   ```sql
   DELIMITER $$
   
   CREATE TRIGGER validar_stock_antes_venta
   BEFORE INSERT ON ventas
   FOR EACH ROW
   BEGIN
       DECLARE stock_disponible INT;
       
       SELECT stock INTO stock_disponible 
       FROM productos 
       WHERE id = NEW.id_producto;
       
       IF NEW.cantidad > stock_disponible THEN
           SIGNAL SQLSTATE '45000' 
           SET MESSAGE_TEXT = 'Error: No hay suficiente stock disponible para esta venta';
       END IF;
       
       UPDATE productos 
       SET stock = stock - NEW.cantidad 
       WHERE id = NEW.id_producto;
   END$$
   
   DELIMITER ;
   ```

3. Probar el trigger.

   ```sql
   1 -- Stock Inicial
   
   SELECT * FROM productos;
   ```

   ```sql
   +----+---------+-------+
   | id | nombre  | stock |
   +----+---------+-------+
   |  1 | Laptop  |    10 |
   |  2 | Mouse   |    25 |
   |  3 | Teclado |    15 |
   |  4 | Monitor |     8 |
   +----+---------+-------+
   ```

   ```sql
   2 -- Venta
   
   INSERT INTO ventas (id_producto, cantidad) VALUES (1, 3);
   ```

   ```sql
   SELECT 'Venta realizada exitosamente - Stock actualizado' AS resultado;
   ```

   ```sql
   +--------------------------------------------------+
   | resultado                                        |
   +--------------------------------------------------+
   | Venta realizada exitosamente - Stock actualizado |
   +--------------------------------------------------+
   ```

   ```sql
   3 -- Validacion del stock despues de la venta
   
   SELECT * FROM productos WHERE id = 1;
   ```

   ```sql
   +----+--------+-------+
   | id | nombre | stock |
   +----+--------+-------+
   |  1 | Laptop |     7 |
   +----+--------+-------+
   ```

   ```sql
   4 -- Esta venta se pasa del stock
   
   INSERT INTO ventas (id_producto, cantidad) VALUES (1, 15);
   
   //Esta falla//
   ```

   ```sql
   5 -- El stock no cambio despues de la venta fallida
   
   SELECT * FROM productos WHERE id = 1;
   ```

   ```sql
   +----+--------+-------+
   | id | nombre | stock |
   +----+--------+-------+
   |  1 | Laptop |     7 |
   +----+--------+-------+
   ```

   ```sql
   6 -- Consulta de todas las ventas realizadas
   
   SELECT v.id, p.nombre, v.cantidad, p.stock AS stock_restante
       -> FROM ventas v
       -> JOIN productos p ON v.id_producto = p.id;
   ```

   ```sql
   +----+--------+----------+----------------+
   | id | nombre | cantidad | stock_restante |
   +----+--------+----------+----------------+
   |  1 | Laptop |        3 |              7 |
   +----+--------+----------+----------------+
   ```



## **🔹 Caso 2: Registro Automático de Cambios en Salarios**

### **Escenario:**

La empresa **TechCorp** desea mantener un registro histórico de todos los cambios de salario de sus empleados.

### **Tarea:**

1. Crear las tablas `empleados` y `historial_salarios`.

   ```sql
   CREATE TABLE empleados (
       id INT PRIMARY KEY AUTO_INCREMENT,
       nombre VARCHAR(50),
       salario DECIMAL(10,2)
   );
   
   CREATE TABLE historial_salarios (
       id INT PRIMARY KEY AUTO_INCREMENT,
       id_empleado INT,
       salario_anterior DECIMAL(10,2),
       salario_nuevo DECIMAL(10,2),
       fecha TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
       FOREIGN KEY (id_empleado) REFERENCES empleados(id)
   );
   ```

   **Datos**

   ```sql
   INSERT INTO empleados (nombre, salario) VALUES 
   ('Ana García', 3500.00),
   ('Carlos López', 4200.00),
   ('María Rodríguez', 3800.00),
   ('José Martínez', 5000.00),
   ('Laura Sánchez', 3200.00);
   ```

2. Implementar un trigger `BEFORE UPDATE` que registre cualquier cambio en el salario.

   ```sql
   DELIMITER $$
   
   CREATE TRIGGER registrar_cambio_salario
   BEFORE UPDATE ON empleados
   FOR EACH ROW
   BEGIN
       IF OLD.salario != NEW.salario THEN
           INSERT INTO historial_salarios (id_empleado, salario_anterior, salario_nuevo)
           VALUES (NEW.id, OLD.salario, NEW.salario);
       END IF;
   END$$
   
   DELIMITER ;
   ```

3. Probar el trigger.

   ```sql
   1 -- Consulta empleados
   
   SELECT * FROM empleados;
   +----+-------------------+---------+
   | id | nombre            | salario |
   +----+-------------------+---------+
   |  1 | Ana García        | 3500.00 |
   |  2 | Carlos López      | 4200.00 |
   |  3 | María Rodríguez   | 3800.00 |
   |  4 | José Martínez     | 5000.00 |
   |  5 | Laura Sánchez     | 3200.00 |
   +----+-------------------+---------+
   ```

   ```sql
   2 -- Cambios de salario
   
   UPDATE empleados SET salario = 3800.00 WHERE id = 1;
   UPDATE empleados SET salario = 4500.00 WHERE id = 2;
   UPDATE empleados SET salario = 4000.00 WHERE id = 3;
   ```

   ```sql
   3 -- Actualizacion nombre cliente
   
   UPDATE empleados SET nombre = 'Ana García Pérez' WHERE id = 1;
   ```

   ```sql
   4 -- Cambio de salario para el mismo empleado
   
   UPDATE empleados SET salario = 4100.00 WHERE id = 1;
   ```

   ```sql
   5 -- Historial de cambios
   
   SELECT 
       ->     h.id,
       ->     e.nombre,
       ->     h.salario_anterior,
       ->     h.salario_nuevo,
       ->     (h.salario_nuevo - h.salario_anterior) AS diferencia,
       ->     h.fecha
       -> FROM historial_salarios h
       -> JOIN empleados e ON h.id_empleado = e.id
       -> ORDER BY h.fecha;
   +----+--------------------+------------------+---------------+------------+---------------------+
   | id | nombre             | salario_anterior | salario_nuevo | diferencia | fecha               |
   +----+--------------------+------------------+---------------+------------+---------------------+
   |  1 | Ana García Pérez   |          3500.00 |       3800.00 |     300.00 | 2025-07-08 12:17:11 |
   |  2 | Carlos López       |          4200.00 |       4500.00 |     300.00 | 2025-07-08 12:17:11 |
   |  3 | María Rodríguez    |          3800.00 |       4000.00 |     200.00 | 2025-07-08 12:17:19 |
   |  4 | Ana García Pérez   |          3800.00 |       4100.00 |     300.00 | 2025-07-08 12:19:15 |
   +----+--------------------+------------------+---------------+------------+---------------------+
   ```

   ```sql
   6 -- Historial de un empleado en especifico
   
   SELECT 
       e.nombre,
       h.salario_anterior,
       h.salario_nuevo,
       h.fecha,
       CASE 
           WHEN h.salario_nuevo > h.salario_anterior THEN 'Aumento'
           WHEN h.salario_nuevo < h.salario_anterior THEN 'Reducción'
           ELSE 'Sin cambio'
       END AS tipo_cambio
   FROM historial_salarios h
   JOIN empleados e ON h.id_empleado = e.id
   WHERE e.id = 1
   ORDER BY h.fecha;
   
   +--------------------+------------------+---------------+---------------------+-------------+
   | nombre             | salario_anterior | salario_nuevo | fecha               | tipo_cambio |
   +--------------------+------------------+---------------+---------------------+-------------+
   | Ana García Pérez   |          3500.00 |       3800.00 | 2025-07-08 12:17:11 | Aumento     |
   | Ana García Pérez   |          3800.00 |       4100.00 | 2025-07-08 12:19:15 | Aumento     |
   +--------------------+------------------+---------------+---------------------+-------------+
   ```

   ```sql
   7 -- Cambios salariales
   
   SELECT 
       ->     COUNT(*) as total_cambios,
       ->     AVG(salario_nuevo - salario_anterior) as promedio_cambio,
       ->     MAX(salario_nuevo - salario_anterior) as mayor_aumento,
       ->     MIN(salario_nuevo - salario_anterior) as mayor_reduccion
       -> FROM historial_salarios;
   +---------------+-----------------+---------------+-----------------+
   | total_cambios | promedio_cambio | mayor_aumento | mayor_reduccion |
   +---------------+-----------------+---------------+-----------------+
   |             4 |      275.000000 |        300.00 |          200.00 |
   +---------------+-----------------+---------------+-----------------+
   ```

   ```sql
   8 -- Empleados con mas cambios salariales
   
   SELECT 
       ->     e.nombre,
       ->     COUNT(h.id) as num_cambios,
       ->     MIN(h.salario_anterior) as salario_inicial,
       ->     MAX(h.salario_nuevo) as salario_actual
       -> FROM empleados e
       -> LEFT JOIN historial_salarios h ON e.id = h.id_empleado
       -> GROUP BY e.id, e.nombre
       -> ORDER BY num_cambios DESC;
   +--------------------+-------------+-----------------+----------------+
   | nombre             | num_cambios | salario_inicial | salario_actual |
   +--------------------+-------------+-----------------+----------------+
   | Ana García Pérez   |           2 |         3500.00 |        4100.00 |
   | Carlos López       |           1 |         4200.00 |        4500.00 |
   | María Rodríguez    |           1 |         3800.00 |        4000.00 |
   | José Martínez      |           0 |            NULL |           NULL |
   | Laura Sánchez      |           0 |            NULL |           NULL |
   +--------------------+-------------+-----------------+----------------+
   ```



## **🔹 Caso 3: Registro de Eliminaciones en Auditoría**

### **Escenario:**

La empresa **DataSecure** quiere registrar toda eliminación de clientes en una tabla de auditoría para evitar pérdidas accidentales de datos.

### **Tarea:**

1. Crear las tablas `clientes` y `clientes_auditoria`.

   ```sql
   CREATE TABLE clientes (
       id INT PRIMARY KEY AUTO_INCREMENT,
       nombre VARCHAR(50),
       email VARCHAR(50)
   );
   
   CREATE TABLE clientes_auditoria (
       id INT PRIMARY KEY AUTO_INCREMENT,
       id_cliente INT,
       nombre VARCHAR(50),
       email VARCHAR(50),
       fecha_eliminacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );
   ```

   **Datos**

   ```sql
   INSERT INTO clientes (nombre, email) VALUES 
   ('Juan Pérez', 'juan.perez@email.com'),
   ('María González', 'maria.gonzalez@email.com'),
   ('Carlos Rodríguez', 'carlos.rodriguez@email.com'),
   ('Ana Martínez', 'ana.martinez@email.com'),
   ('Luis Sánchez', 'luis.sanchez@email.com'),
   ('Carmen López', 'carmen.lopez@email.com');
   ```

2. Implementar un trigger `AFTER DELETE` para registrar los clientes eliminados.

   ```sql
   DELIMITER $$
   
   CREATE TRIGGER registrar_cliente_eliminado
   AFTER DELETE ON clientes
   FOR EACH ROW
   BEGIN
       INSERT INTO clientes_auditoria (id_cliente, nombre, email)
       VALUES (OLD.id, OLD.nombre, OLD.email);
   END$$
   
   DELIMITER ;
   ```

3. Probar el trigger.

   ```sql
   1 -- Consulta clientes
   
   SELECT * FROM clientes;
   +----+-------------------+----------------------------+
   | id | nombre            | email                      |
   +----+-------------------+----------------------------+
   |  1 | Juan Pérez        | juan.perez@email.com       |
   |  2 | María González    | maria.gonzalez@email.com   |
   |  3 | Carlos Rodríguez  | carlos.rodriguez@email.com |
   |  4 | Ana Martínez      | ana.martinez@email.com     |
   |  5 | Luis Sánchez      | luis.sanchez@email.com     |
   |  6 | Carmen López      | carmen.lopez@email.com     |
   +----+-------------------+----------------------------+
   ```

   ```sql
   2 -- Eliminacion de un cliente
   
   DELETE FROM clientes WHERE id = 1;
   ```

   ```sql
   3 -- Verificacion para ver si se registro en auditoria
   
   SELECT * FROM clientes_auditoria;
   +----+------------+-------------+----------------------+---------------------+
   | id | id_cliente | nombre      | email                | fecha_eliminacion   |
   +----+------------+-------------+----------------------+---------------------+
   |  1 |          1 | Juan Pérez  | juan.perez@email.com | 2025-07-08 12:27:52 |
   +----+------------+-------------+----------------------+---------------------+
   ```

   ```sql
   4 -- Eliminacion de varios clientes
   
   DELETE FROM clientes WHERE id IN (2, 3);
   ```

   ```sql
   5 -- Historial de eliminaciones
   
   SELECT 
       ->     ca.id,
       ->     ca.id_cliente,
       ->     ca.nombre,
       ->     ca.email,
       ->     ca.fecha_eliminacion,
       ->     'Cliente eliminado' as accion
       -> FROM clientes_auditoria ca
       -> ORDER BY ca.fecha_eliminacion;
   +----+------------+-------------------+----------------------------+---------------------+-------------------+
   | id | id_cliente | nombre            | email                      | fecha_eliminacion   | accion            |
   +----+------------+-------------------+----------------------------+---------------------+-------------------+
   |  1 |          1 | Juan Pérez        | juan.perez@email.com       | 2025-07-08 12:27:52 | Cliente eliminado |
   |  2 |          2 | María González    | maria.gonzalez@email.com   | 2025-07-08 12:30:00 | Cliente eliminado |
   |  3 |          3 | Carlos Rodríguez  | carlos.rodriguez@email.com | 2025-07-08 12:30:00 | Cliente eliminado |
   +----+------------+-------------------+----------------------------+---------------------+-------------------+
   ```

   ```sql
   6 -- Comparacion entre clientes activos y eliminados
   
   SELECT 
       ->     'Activos' as estado,
       ->     COUNT(*) as cantidad
       -> FROM clientes
       -> UNION ALL
       -> SELECT 
       ->     'Eliminados' as estado,
       ->     COUNT(*) as cantidad
       -> FROM clientes_auditoria;
   +------------+----------+
   | estado     | cantidad |
   +------------+----------+
   | Activos    |        3 |
   | Eliminados |        3 |
   +------------+----------+
   ```

   ```sql
   7 -- ELiminacion de cliente por email
   
   DELETE FROM clientes WHERE email = 'ana.martinez@email.com';
   ```

   ```sql
   8 -- Cliente eliminado en auditoria
   
   SELECT 
       ->     ca.nombre,
       ->     ca.email,
       ->     ca.fecha_eliminacion,
       ->     TIMESTAMPDIFF(MINUTE, ca.fecha_eliminacion, NOW()) as minutos_desde_eliminacion
       -> FROM clientes_auditoria ca
       -> WHERE ca.email = 'ana.martinez@email.com';
   +---------------+------------------------+---------------------+---------------------------+
   | nombre        | email                  | fecha_eliminacion   | minutos_desde_eliminacion |
   +---------------+------------------------+---------------------+---------------------------+
   | Ana Martínez  | ana.martinez@email.com | 2025-07-08 12:32:56 |                         0 |
   +---------------+------------------------+---------------------+---------------------------+
   ```



## **🔹 Caso 4: Restricción de Eliminación de Pedidos Pendientes**

### **Escenario:**

En un sistema de ventas, no se debe permitir eliminar pedidos que aún están **pendientes**.

### **Tarea:**

1. Crear las tablas `pedidos`.

   ```sql
   CREATE TABLE pedidos (
       id INT PRIMARY KEY AUTO_INCREMENT,
       cliente VARCHAR(100),
       estado ENUM('pendiente', 'completado')
   );
   ```

   **Datos**

   ```sql
   INSERT INTO pedidos (cliente, estado) VALUES 
   ('Juan Pérez', 'pendiente'),
   ('María González', 'completado'),
   ('Carlos Rodríguez', 'pendiente'),
   ('Ana Martínez', 'completado'),
   ('Luis Sánchez', 'pendiente'),
   ('Carmen López', 'completado');
   ```

2. Implementar un trigger `BEFORE DELETE` para evitar la eliminación de pedidos pendientes.

   ```sql
   DELIMITER $$
   
   CREATE TRIGGER evitar_eliminacion_pendientes
   BEFORE DELETE ON pedidos
   FOR EACH ROW
   BEGIN
       IF OLD.estado = 'pendiente' THEN
           SIGNAL SQLSTATE '45000' 
           SET MESSAGE_TEXT = 'Error: No se puede eliminar un pedido pendiente. Debe completarse primero.';
       END IF;
   END$$
   
   DELIMITER ;
   ```

3. Probar el trigger.

   ```sql
   1 -- Consulta pedidos
   
   SELECT * FROM pedidos;
   +----+-------------------+------------+
   | id | cliente           | estado     |
   +----+-------------------+------------+
   |  1 | Juan Pérez        | pendiente  |
   |  2 | María González    | completado |
   |  3 | Carlos Rodríguez  | pendiente  |
   |  4 | Ana Martínez      | completado |
   |  5 | Luis Sánchez      | pendiente  |
   |  6 | Carmen López      | completado |
   +----+-------------------+------------+
   ```

   ```sql
   2 -- Intento de eliminacion de un pedido pendiente
   
   SELECT 'Intentando eliminar pedido pendiente (ID: 1)...' as prueba;
   +-------------------------------------------------+
   | prueba                                          |
   +-------------------------------------------------+
   | Intentando eliminar pedido pendiente (ID: 1)... |
   +-------------------------------------------------+
   
   DELETE FROM pedidos WHERE id = 1;
   // Esta consulta falla//
   ```

   ```sql
   3 -- Eliminacion de un pedido completado
   
   SELECT 'Eliminando pedido completado (ID: 2)...' as prueba;
   +-----------------------------------------+
   | prueba                                  |
   +-----------------------------------------+
   | Eliminando pedido completado (ID: 2)... |
   +-----------------------------------------+
   
   DELETE FROM pedidos WHERE id = 2;
   ```

   ```sql
   4 -- Verificacion para ver si se elimino el pedido completado
   
   SELECT * FROM pedidos;
   +----+-------------------+------------+
   | id | cliente           | estado     |
   +----+-------------------+------------+
   |  1 | Juan Pérez        | pendiente  |
   |  3 | Carlos Rodríguez  | pendiente  |
   |  4 | Ana Martínez      | completado |
   |  5 | Luis Sánchez      | pendiente  |
   |  6 | Carmen López      | completado |
   +----+-------------------+------------+
   ```

   ```sql
   5 -- Completar un pedido pendiente y despues eliminarlo
   
   UPDATE pedidos SET estado = 'completado' WHERE id = 3;
   
   DELETE FROM pedidos WHERE id = 3;
   ```

   ```sql
   6 -- Estado actual
   
   SELECT * FROM pedidos;
   +----+---------------+------------+
   | id | cliente       | estado     |
   +----+---------------+------------+
   |  1 | Juan Pérez    | pendiente  |
   |  4 | Ana Martínez  | completado |
   |  5 | Luis Sánchez  | pendiente  |
   |  6 | Carmen López  | completado |
   +----+---------------+------------+
   ```

   



