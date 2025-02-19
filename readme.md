# Proyecto Node.js con y sin Clúster + PM2

Este proyecto muestra cómo ejecutar una aplicación Node.js con y sin clúster para comparar su rendimiento, realizar pruebas de carga con `loadtest`, y administrar procesos con `PM2`.

## 📌 Requisitos previos

Asegúrate de tener instalado en tu servidor Debian:

- **Node.js**  
- **npm**  
- **PM2** (`npm install -g pm2`)  
- **loadtest** (`npm install -g loadtest`)  

---

## 🚀 Paso 1: Implementación sin Clúster

1. **Conectarse al servidor Debian mediante SSH.**  
2. **Crear un directorio para el proyecto y acceder a él.**  
3. **Inicializar el proyecto y configurar Express:**
   ```sh
   npm init -y
   npm install express
   ```
4. **Crear el archivo `app_no_cluster.js` con el siguiente código:**
   
   ```javascript
   const express = require("express");
   const app = express();
   const port = 3000;
   const limit = 5000000000;

   app.get("/", (req, res) => {
       res.send("Hello World!");
   });

   app.get("/api/:n", function (req, res) {
       let n = parseInt(req.params.n);
       let count = 0;
       if (n > limit) n = limit;
       for (let i = 0; i <= n; i++) {
           count += i;
       }
       res.send(`Final count is ${count}`);
   });

   app.listen(port, () => {
       console.log(`App listening on port ${port}`);
   });
   ```

5. **Ejecutar la aplicación sin clúster:**
   ```sh
   node app_no_cluster.js
   ```
6. **Probar en el navegador:**
   ```
   http://192.168.56.10-:3000/
   http://192.168.56.10:3000/api/50
   ```

---

## 🚀 Paso 2: Implementación con Clúster

1. **Crear el archivo `app_cluster.js` con el siguiente código:**
   
   ```javascript
   const express = require("express");
   const cluster = require("cluster");
   const os = require("os");
   const totalCPUs = os.cpus().length;
   const port = 3000;
   const limit = 5000000000;

   if (cluster.isMaster) {
       console.log(`Number of CPUs: ${totalCPUs}`);
       console.log(`Master ${process.pid} is running`);

       for (let i = 0; i < totalCPUs; i++) {
           cluster.fork();
       }

       cluster.on("exit", (worker) => {
           console.log(`Worker ${worker.process.pid} died, creating a new one`);
           cluster.fork();
       });
   } else {
       const app = express();
       console.log(`Worker ${process.pid} started`);

       app.get("/", (req, res) => {
           res.send("Hello World!");
       });

       app.get("/api/:n", function (req, res) {
           let n = parseInt(req.params.n);
           let count = 0;
           if (n > limit) n = limit;
           for (let i = 0; i <= n; i++) {
               count += i;
           }
           res.send(`Final count is ${count}`);
       });

       app.listen(port, () => {
           console.log(`App listening on port ${port}`);
       });
   }
   ```

2. **Ejecutar la aplicación con clúster:**
   ```sh
   node app_cluster.js
   ```

---

## 🚀 Paso 3: Pruebas de Carga con loadtest

1. **Instalar loadtest:**
   ```sh
   npm install -g loadtest
   ```
2. **Ejecutar pruebas de carga en la versión sin clúster:**
   ```sh
   loadtest http://192.168.56.10:3000/api/500000 -n 1000 -c 100
   ```
3. **Ejecutar pruebas de carga en la versión con clúster:**
   ```sh
   loadtest http://192.168.56.10:3000/api/500000 -n 1000 -c 100
   ```

📊 **Comparación de Resultados:**
- La versión con clúster maneja más peticiones por segundo con menor latencia.
- Sin clúster, las peticiones grandes bloquean otras solicitudes concurrentes.

---

## 🚀 Paso 4: Uso de PM2 para Administrar Procesos

1. **Instalar PM2:**
   ```sh
   npm install -g pm2
   ```
2. **Ejecutar la aplicación sin clúster con PM2 en modo clúster:**
   ```sh
   pm2 start app_no_cluster.js -i 0
   ```
   `-i 0` indica que PM2 usará todos los núcleos de CPU disponibles.
3. **Ver el estado de la aplicación en PM2:**
   ```sh
   pm2 list
   ```
4. **Detener la aplicación en PM2:**
   ```sh
   pm2 stop app_no_cluster
   ```
5. **Eliminar la aplicación de PM2:**
   ```sh
   pm2 delete app_no_cluster
   ```

---

## 🚀 Paso 5: Configuración con Ecosystem File

1. **Generar el archivo de configuración de PM2:**
   ```sh
   pm2 ecosystem
   ```
2. **Modificar `ecosystem.config.js` para incluir la configuración:**
   
   ```javascript
   module.exports = {
       apps: [{
           name: "my_app",
           script: "app_no_cluster.js",
           instances: 0,
           exec_mode: "cluster",
       }],
   };
   ```

3. **Iniciar la aplicación con PM2 usando el archivo de configuración:**
   ```sh
   pm2 start ecosystem.config.js


4. **Conclusión: ¿Por qué en algunos casos la versión sin clúster tiene mejores resultados?**
Aunque el uso de clústeres mejora el rendimiento en la mayoría de las aplicaciones al distribuir la carga entre múltiples núcleos de CPU, en ciertos casos concretos, como el de este proyecto, la versión sin clúster puede ofrecer mejores resultados. Esto se debe a que el procesamiento de operaciones pesadas en bucles o tareas computacionales intensivas puede no beneficiarse del modelo de clúster, ya que la sobrecarga de gestión entre los procesos y la sincronización entre los mismos puede terminar afectando el rendimiento.

Cuando el servidor está realizando cálculos de gran magnitud o manejando operaciones que consumen mucho tiempo, la capacidad de clúster para dividir y distribuir el trabajo puede verse contrarrestada por la necesidad de compartir el estado entre los procesos y coordinar su ejecución. En estos casos, una ejecución secuencial sin clúster puede ser más eficiente al evitar dicha sobrecarga. Esto se vuelve evidente cuando la prueba de carga muestra una mayor latencia y menor rendimiento en la versión con clúster, especialmente con cargas altas en operaciones específicas.