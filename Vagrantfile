Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  config.vm.network "private_network", ip: "192.168.56.10"
    config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
    vb.cpus = 2
  end

  config.vm.provision "shell", inline: <<-SHELL
    # Actualizar paquetes e instalar dependencias
    apt-get update && apt-get upgrade -y
    apt-get install -y curl git

    # Instalar Node.js y npm
    curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
    apt-get install -y nodejs

    # Instalar PM2 y loadtest
    npm install -g pm2 loadtest

    # Crear el proyecto Node.js
    mkdir -p /home/vagrant/node_project && cd /home/vagrant/node_project
    npm init -y
    npm install express


# Crear app_no_cluster.js
cat <<EOF > /home/vagrant/node_project/app_no_cluster.js
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
EOF

# Crear app_cluster.js
cat <<EOF > /home/vagrant/node_project/app_cluster.js
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
EOF

# Configurar PM2
pm2 start /home/vagrant/node_project/app_no_cluster.js -i 0
pm2 save
pm2 startup systemd

  SHELL
end
