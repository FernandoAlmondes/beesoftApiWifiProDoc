### --> beesoftApiWifiPro v1.0 (By: Bee Solutions) <-- ###

--- ---
# Projeto destinado ao monitoramento Wireless via Web com Grafana Infinity + API Beesoft
- Shellscript, Python e Django Rest
- Zabbix e Grafana
--- ---
- Sobre o projeto:
> - Desenvolvido por: Bee Solutions
> - Autor: Fernando Almondes
> - Principais ferramentas: Shellscript, Python e Django Rest
--- ---

- Distribuições homologados (Sistemas Operacionais Linux Server)
> - Ubuntu Server 22.04 LTS (Ou superior)
> - Debian 12 (Ou superior)

--- ---

- Compatibilidade:
> - Ubiquiti/Ubnt (Disponível ✅)
> - Mikrotik (Disponível ✅)
> - Mimosa (Disponível ✅)
> - Intelbras (Disponível ✅)
--- ---

# 1# Dashboard de exemplo | Análise Wireless (Ubiquiti)

![Painel](https://beesolutions.notion.site/image/https%3A%2F%2Fprod-files-secure.s3.us-west-2.amazonaws.com%2F523e516e-2d85-4d13-a6cc-d443190816f1%2F2881e57f-47b1-4e34-a1d8-383cdeef70b2%2Fdashboard-analise-aps-ubiquiti.png?table=block&id=7f47c9e4-103d-4f04-8fdf-0428d4a9670c&spaceId=523e516e-2d85-4d13-a6cc-d443190816f1&width=2000&userId=&cache=v2)
--- ---

### Parte 1 - Fazendo download e instalando aplicação ###

- Crie o diretorio base para o projeto (BeesoftPro).
```shell
mkdir /opt/bee/
```

- Navegue até o diretorio base do projeto.
```shell
cd /opt/bee/
```

- Faça download do código fonte (Copie para o diretório /opt/bee/ o diretório do projeto baixado pelo link do Google Drive (beesoftApiWifiPro), use o WinSCP ou Filezila).
```shell
Veja no Canal Pro (Telegram)
```

- Entre no diretorio do projeto e crie o diretorio de jsons (tmp).
```shell
cd /opt/bee/beesoftApiWifiPro/
mkdir tmp/
```

- Instate o python3-venv para gerenciar ambientes virtuais com Python e também as dependencias do Linux uuid, git.
```shell
apt install python3-venv uuid git
```

- Crie um novo ambiente virtual Python.
```shell
python3 -m venv venv
```

- Ative o seu ambiente virtual.
```shell
source venv/bin/activate
```

- Instale as dependencias do python para o projeto.
```shell
pip install -r dependencias.txt
```

- Gere as chaves de segurança para usar no seu settings.py (Guarde essas chaves, voce as usara em seguida).
```shell
python /opt/bee/beesoftApiWifiPro/chave.py
```

- Renomei o arquivo .env e preencha as informações a seguir.
```shell
mv /opt/bee/beesoftApiWifiPro/beesoft/.env.exemplo /opt/bee/beesoftApiWifiPro/beesoft/.env
```
- Ajuste o arquivo .env e preencha as informações corretamente.
```shell
nano /opt/bee/beesoftApiWifiPro/beesoft/.env
```
```shell
SECRET_KEY=TOKEN-SECRET_KEY-GERADO-PELO-SCRIPT-CHAVE.PY-AQUI
DEBUG=True
ALLOWED_HOSTS=127.0.0.1, localhost, beesoftapiwifi.meudominio.com.br
BEE_API_TOKEN=TOKEN-BEE_API_TOKEN-GERADO-PELO-SCRIPT-CHAVE.PY-AQUI

```

- Realizando teste no servidor Django.
```shell
python manage.py runserver
```
```shell
# Exemplo de resultado esperado:
(venv) root@bee:/opt/bee/beesoftApiWifiPro# python manage.py runserver
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).

You have 21 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, painel, sessions.
Run 'python manage.py migrate' to apply them.
May 20, 2023 - 00:54:57
Django version 4.2.1, using settings 'beesoft.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

# Parte 2 - Fazendo o deploy da aplicação usando Nginx e Gunicorn  #

- Criando serviço para execução em segundo plano.
```shell
nano /etc/systemd/system/beesoftapiwifi.service
```
```shell
[Unit]
Description=Beesoft API daemon
Requires=beesoftapiwifi.socket
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/opt/bee/beesoftApiWifiPro
ExecStart=/opt/bee/beesoftApiWifiPro/venv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/beesoftapiwifi.sock \
          beesoft.wsgi:application

[Install]
WantedBy=multi-user.target

```

- Criando socket (Será usado no proxy reverso com Nginx).
```shell
nano /etc/systemd/system/beesoftapiwifi.socket
```
```shell
[Unit]
Description=Beesoft API socket

[Socket]
ListenStream=/run/beesoftapiwifi.sock

[Install]
WantedBy=sockets.target
```

- Instalando Nginx (Se você já tiver o apache2 rodando ajuste as portas).
```shell
apt install nginx
```

- Desative ou remova o link simbolico default do Nginx (Ou você pode ter conflito na porta 80, caso tenha o apache2 instalado).
```shell
unlink /etc/nginx/sites-enabled/default
```

- Criando servidor web com o Nginx.
```shell
nano /etc/nginx/sites-enabled/beesoftapiwifi.seudominio.com.br
```
```shell
server {
    listen 5006;
    server_name beesoftapiwifi.seudominio.com.br;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /opt/bee/beesoftApiWifiPro;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/beesoftapiwifi.sock;
    }
}
```

- Iniciando e verificando serviços necessários.
```shell
systemctl daemon-reload

systemctl start beesoftapiwifi.socket beesoftapiwifi.service

nginx -t
service nginx restart

systemctl status beesoftapiwifi.service
```

- Habilitando inicialização junto so boot dos serviços.
```shell
systemctl enable beesoftapiwifi.service beesoftapiwifi.socket
```

- Após validação, lembre-se de desativar o modo debug no arquivo .env e reiniciar a aplicação.
```shell
nano /opt/bee/beesoftApiWifiPro/beesoft/.env
```
```shell
DEBUG=False
```
- Reiniciando serviços após desativar o modo Debug.
```shell
systemctl restart beesoftapiwifi.socket beesoftapiwifi.service

service nginx restart
```

### Parte 3 - Importe o dashboard no Grafana e configurações adicionais ###

- Instale no Grafana o plugin Infinity
- Crie o datasource e salve (Não precisa configurar nada no datasource inicialmente)

- Importe o arquivo json no Grafana e ajuste as variaveis "servidor" e "token" de acordo com o seu ambiente:
- Dashboard -> Settings -> Variables -> token (Adicione o seu token)
- Dashboard -> Settings -> Variables -> servidor (Adicione o IP do seu servidor)

### Parte 4 - Comunidade no Telegram e Canal do YouTube ###

- [Comunidade no Telegram](https://t.me/beesolutions)
- [Canal no Youtuve](https://www.youtube.com/beesolutions)

> Participe e colabore com nossos projetos (Bee Solutions 2024).

--- ---
